# ADR-0008: Azure 운영 배포 — pgvector RAG (Postgres · Blob · OpenAI API-key · Container Apps)

> **실구현 보충(2026-07)**: 시크릿은 Key Vault 대신 **Container Apps secrets**(`secretref:`)로 운영 중이며, Blob 저장(`services/storage.py`)·임베딩 분기는 결정대로 구현 완료. 본문은 결정 시점(2026-06-10) 기록으로 보존한다.


- 상태: **승인 — 프로비저닝 진행(검토 패키지 `infra/` 동봉)**
- 날짜: 2026-06-10 (개정 3판 — AI Search 제외, pgvector 채택)
- 관련: `infra/main.bicep`·`infra/README.md`(IaC·비용·절차·태그), ADR-0001(샌드박스), ADR-0004/0005(self-modifying tools·Claude Code), ADR-0006(Progressive Workspace)

## 배경

회사 방침으로 **챗봇 운영 티어만** Azure에 배포한다. 개발은 로컬을 유지한다(운영/개발 분리는 `APP_ENV=prod` 런타임 플래그로, 코드포크 없음 — `CONTRIBUTING.md`). 구독 `SKSM`, 리전 `koreacentral`, 리소스 그룹 `ai-part-rg`(공유 — 태그로 비용 분리), 도메인 `chatbot.sinokor.co.kr`.

개정 이력: 1판 pgvector 유지·AI Search 보류 → 2판 AI Search 채택 → **3판(최종) AI Search 제외, PostgreSQL + pgvector 채택**. 이유: 현 RAG 하이브리드(HNSW + tsvector + RRF + LLM 리랭커)가 PG 안에서 단일 권한 모델로 이미 동작 → AI Search 도입은 코드 재작성·재임베딩·고정비(~$75/월)만 추가. pgvector 유지가 **재임베딩 불필요 + 검색 코드 무변경 + 절반 이하 비용**.

## 결정

### 1. 검색 — PostgreSQL + pgvector (AI Search 미채택)
- Postgres Flexible Server 에 `azure.extensions=VECTOR,PG_TRGM` 활성 → 현 `rag.py` 하이브리드(HNSW cosine + tsvector + RRF + 선택적 LLM 리랭커) **코드 그대로**.
- 벡터·청크 본문·메타·팀/사용자 가시성 필터가 **한 DB·한 권한 모델**에 유지 → 교차 테넌트 누출 방지 로직(기존 visibility 필터) 그대로.
- **재임베딩 불필요** — 임베딩 소스만 일관 유지하면 됨.
- 재검토 트리거(AI Search 재오픈): 코퍼스 수십만 건+로 단일 DB HNSW 한계 / 시맨틱 랭커·다국어 분석기 강한 요구 / 검색 트래픽이 DB 자원 경합.

### 2. LLM/임베딩 — API key 방식, sinokor-ai-foundry 미사용
- 공유 `sinokor-ai-foundry`에 Managed Identity를 **묶지 않는다**(결합·권한 회피). **API key**로 LLM/임베딩 접근(Key Vault 시크릿).
- LiteLLM 게이트웨이·폴백 체인 유지. 현재 OpenAI 키 그대로 사용 가능 → 별도 Azure OpenAI 리소스 신규 생성 불필요(필요 시 후속).

### 3. 저장소 — Blob + 드라이버 추상화
- 신규 `backend/app/services/storage.py`(`LocalStorageBackend`/`AzureStorageBackend`). `Document.storage_path`는 로컬 경로 ↔ Blob URI 이중 의미. 인증은 UAMI(Managed Identity).

### 4. 호스팅 — Container Apps (FE/BE), Front Door 미도입
- 백엔드(FastAPI/SSE)·프론트(Next.js 15) 모두 Container Apps(SWA 하이브리드 SSR이 Preview라 단일 플랫폼 통일).
- 프론트 **min-replicas=0**(scale-to-zero), 백엔드 **min-replicas=1**(SSE 콜드스타트 회피·상시 과금).
- 단일 리전 내부앱 → Front Door/WAF 미도입. TLS·커스텀 도메인은 Container Apps 관리형 인증서.
- **SSE 240초 ingress 캡**: keep-alive 주기 전송 + 클라이언트 `Last-Event-ID` 재연결 설계.

### 5. 인증 — 단일 UAMI 키리스
- ACR Pull · Key Vault Secrets User · Storage Blob Data Contributor 를 단일 User-Assigned Managed Identity로. (LLM만 API key 예외, DB는 비밀번호+KV)

### 6. 데이터 — PostgreSQL Flexible Server (관계형 + 벡터)
- 관계형 데이터 + pgvector 벡터/HNSW. Burstable B1ms 시작, 코퍼스↑ 시 B2s/GP 상향. 운영 강화 시 VNet/Private + Entra 인증.

### 7. 비용 추적 — 공유 RG + 태그
- `ai-part-rg` 공유 → 전 chatbot 리소스에 `tags`(project/env/costCenter/managedBy) 부착. Cost Management 에서 `project=chatbot` 필터로 분리 집계. 태그는 Bicep 단일 소스, 재배포로 변경(소급 아님). 상세 `infra/README.md`.

## RAG 흐름

```
업로드  → BE → Blob(documents/{team}/{doc}_{file})
인덱싱  → BE → 청킹 → 임베딩(API key) → pgvector(embedding) + tsvector INSERT
검색    → BE(세션→team/scope 주입) → pgvector HNSW + tsvector → RRF → (옵션)LLM 리랭크 → 가시성 필터
응답    → 번호 컨텍스트 + injection 가드(ADR #4) → LLM(API key) → SSE 스트리밍
```

## 코드 마이그레이션 체크리스트 (기본값=현행, 플래그 전환)

| 파일 | 변경 | 신규 env |
|---|---|---|
| `core/config.py` | 스토리지/임베딩 프로바이더 플래그 + Azure 설정 | `STORAGE_DRIVER`, `AZURE_STORAGE_ACCOUNT`, `AZURE_STORAGE_CONTAINER`, (선택)`AZURE_OPENAI_*` |
| `services/embeddings.py` | provider 분기(OpenAI ↔ Azure OpenAI, API key) | `EMBEDDING_PROVIDER` |
| `services/rag.py` | **변경 없음** (pgvector 하이브리드 유지) | — |
| `services/ingest.py` | 파일 read 를 storage 드라이버로(Blob/local) | — |
| `features/documents/router.py` | 업로드/다운로드/삭제 → storage 드라이버 | — |
| **신규** `services/storage.py` | Local/Blob 추상화 | — |
| `db/models.py` | **변경 없음** (`DocumentChunk.embedding` pgvector 유지) | — |
| `pyproject.toml` | `azure-storage-blob`, `azure-identity` 추가 (azure-search-documents **불필요**) | — |

> AI Search 안을 폐기해 `services/search_azure.py` 신규·재임베딩·`db/models.py` deprecate 가 **모두 불필요**해졌다. 실질 코드 변경은 **Blob 스토리지 드라이버 + 임베딩 API-key 분기**뿐.

## 리스크
- 비용은 사용량(vCPU-초/토큰) 변동 — `infra/README.md` 추정표 + Pricing Calculator 확정.
- pgvector 규모: B1ms 는 소규모 내부 앱 기준. 코퍼스↑ 시 HNSW 인덱스 메모리·성능 위해 SKU 상향.
- 콘텐츠 필터(Azure OpenAI 경로 사용 시) 기본 medium — RAG 주입 문서/스트리밍 영향 가능(public OpenAI 키 유지 시 해당 없음).
- 커스텀 도메인은 중간 프록시 없이 직접 CNAME/TXT 필요.

# 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────────┐
│ 브라우저 (Next.js) │
│ /login · /register · /chat · /documents · /chatbots · /admin · /team │
│ /notices · /faq · /qa · /mypage · /guide (운영 포함) │
│ /tools · /skills · /workflows · /schedules · /workspaces · /studio │
│ (개발 전용·운영 미노출 — isRagOnly 가 메뉴/화면 숨김) │
└──────────┬──────────────────────────────────────────────────────┘
 │ HTTPS · Bearer JWT · SSE stream
 ▼
┌─────────────────────────────────────────────────────────────────┐
│ FastAPI (backend/app) — feature-sliced: features/<도메인>/router.py │
│ 운영(prod): auth · documents · chatbots · chat · team · admin · │
│ conversations · notices · faqs · usage · stats … RAG 핵심 │
│ 개발 전용(FEATURE_* off → main.py 가 등록 생략 → 404): │
│ tools · skills · workflows · schedules · claude_code · workspace │
│ services/ ingest · rag · agent · chatbot_rag · tool_registry · llm_runtime │
│ document_parser · chunking · embeddings · code_sandbox │
│ core/middleware/request_logging │
└──────┬──────────────┬──────────────┬─────────────┬──────────────┘
 │ │ │ │
 ▼ ▼ ▼ ▼
┌──────────────┐ ┌────────────┐ ┌──────────┐ ┌─────────────────┐
│ PostgreSQL │ │ Blob/로컬 │ │ LiteLLM │ │ 외부 MCP/HTTP │
│ pgvector+gin │ │ data/ │ │ ↔ LLM │ │ Gmail / Kakao / │
│ HNSW cosine │ │ uploads │ │ Providers│ │ Calendar / ... │
└──────────────┘ └────────────┘ └──────────┘ └─────────────────┘
```

## 핵심 설계 원칙

1. **팀 격리(Tenant Isolation)** — 모든 쿼리는 `team_id`를 기준으로 필터. 문서·챗봇·대화·도구 자격증명 모두 팀 경계를 넘지 않습니다.
2. **권한 계층** — `super_admin > team_admin > team_auditor > team_member`. 상위 권한은 하위 권한 작업을 수행할 수 있습니다(관리자 대행). 자세한 내용은 [rbac.md](./rbac.md).
3. **문서 = 원본 파일 + 청크 세트** — 원본은 로컬 캐시(`data/uploads`) + 운영은 Azure Blob write-through(`STORAGE_DRIVER=azure_blob`, ADR-0008), 메타데이터·청크·임베딩은 DB. Cascade Delete로 문서 삭제 시 관련 청크가 자동 삭제됩니다.
4. **챗봇 = 모델 + 프롬프트 + 도구 + 문서 스코프** — 챗봇마다 대상 문서, 허용 도구, 가시성, 시스템 프롬프트를 고정해 재사용 가능한 에이전트 프로필로 사용합니다.
5. **도구 플러그인화** — 도구는 `tools` 테이블에 등록, `chatbot_tools`로 챗봇에 바인딩. 빌트인(Python 구현) · HTTP 어댑터 · OAuth 기반(Gmail 등) 3가지 종류.
6. **성능(검색) 포커스** — 문서 수·청크 수가 많아져도 쿼리가 선형적으로 느려지지 않도록, 챗봇 단위로 가시 문서 집합을 제한하고 `document_chunks`는 `(document_id, embedding)` 기준 HNSW · `content_tsv` GIN을 유지합니다. 데이터가 크게 증가하면 [erd.md](./erd.md)에 기술된 파티셔닝 전략으로 확장합니다.

## 런타임 흐름

### 1) 로그인

```
[프론트] POST /auth/login ──▶ [auth/router.py]
 식별자: 이메일 또는 그룹웨어 아이디(@ 유무로 분기)
 검증: bcrypt(자체) OR SHA-256 위임(그룹웨어 해시 — HR 동기화가 매일 갱신)
 → access(60분) + 회전 refresh 쿠키(14일)
 (포털 경유는 POST /auth/sso/exchange — HS256 토큰 교환·JIT 가입)
 ▲
 │ 가입이 아직 승인되지 않았으면 403 · 운영 회원가입은 FEATURE_SIGNUP=false
```

### 2) 문서 업로드 → 벡터스토어 자동 생성

```
[프론트] POST /documents/upload (multipart)
 │
 ▼
[documents.py] 파일 저장(로컬 캐시 + 운영은 Blob write-through)
 Document 레코드 status=processing
 asyncio.create_task(process_document_job(doc_id))
 │
 ▼
 [ingest.py] extract_text ─▶ split_text ─▶ embed_texts
 │
 ▼
 DocumentChunk N건 INSERT → status=ready
```

### 3) 문서 삭제 → 벡터스토어 청크 삭제

```
[프론트] DELETE /documents/{id}
 │
 ▼
[documents.py] 권한 검증 → db.delete(Document)
 (cascade) DocumentChunk 일괄 삭제
 (cascade) DocumentShare 삭제
 (cascade) ChatbotDocument 링크 삭제
 파일 시스템의 원본도 unlink
```

### 4) 챗봇 대화 (SSE 스트림)

```
[프론트] POST /chat/stream { chatbot_id, message, ... }
 │
 ▼
[chat.py] 챗봇 로드 → 시스템 프롬프트·허용 도구·문서 스코프 결정
 ├─ Chatbot.rag_scope에 따라 chatbot_rag.search()
 └─ agent.run_agent(model, messages, tools=챗봇에 바인딩된 도구)
 │
 ▼ SSE events: text / step_start / step_end / sandbox / image / done
 Message 저장 · 백그라운드로 제목/유저 메모리 갱신
```

### 5) 도구 실행 (예: Gmail 발송)

```
[에이전트 루프] tool_call → tool_registry.dispatch("gmail.send", args, user_ctx)
 │
 ▼
 ToolCredential 조회(팀/사용자) → OAuth 토큰
 │
 ▼
 외부 API 호출 (Gmail REST / MCP)
 │
 ▼
 결과를 LLM에 다시 주입 → 최종 응답
```

## 경계(Boundaries)

- **인증 경계**: JWT는 `Authorization: Bearer <token>` 헤더 하나로만 수용. 쿠키 인증은 사용하지 않음.
- **파일 경계**: 업로드 파일은 절대 LLM에 바이너리로 전달되지 않음. 항상 파서가 텍스트로 변환한 뒤 청크화.
- **샌드박스 경계**: 코드 실행(개발 전용·운영 미노출 — `code_interpreter` 는 ADR-0005 로 폐기, `claude_code` 가 대체)은 Docker 컨테이너에서만(네트워크 없음, 메모리 512MB, 30s).
- **외부 API 경계**: 외부 도구(카카오톡, Gmail)는 `tool_registry` 어댑터를 통해서만 호출. 사용자 자격증명은 `tool_credentials`에 저장(암호화 대상 필드).

## 성능 포커스

문제: 문서가 많아질수록 단일 `document_chunks` 테이블의 HNSW 인덱스가 커져 검색 지연이 증가.

해결:
1. **챗봇-문서 스코프 축소**: 챗봇이 `rag_scope=linked_only`라면 `chatbot_documents`에 연결된 문서의 청크만 본다. 챗봇 단위로 검색 공간을 제한하여 평균 검색량을 줄임.
2. **조건부 HNSW 스캔**: 사전 필터(`document_id IN (...)`)를 거친 뒤 HNSW를 탐색하므로, pgvector의 `hnsw.ef_search` 파라미터가 작아도 정확도 유지.
3. **수평 확장 준비 — 파티셔닝**: 팀이 많아지면 `document_chunks`를 `team_id` 해시 파티션으로 분할. `document_id`/`chatbot_id` 스코프로 검색 공간을 좁힌다(별도 `vectorstore_partition` 컬럼은 없음).
4. **LRU 캐시**: 자주 재사용되는 챗봇 시스템 프롬프트/문서 스코프 id 목록은 `chatbot_service.py`에서 메모리 캐시.
5. **임베딩 배치**: `embed_texts`가 32건 단위로 OpenAI 호출 → 업로드 지연 감소.

구체적 인덱스 설계와 파티셔닝 SQL은 [erd.md](./erd.md#성능-전략)에 있습니다.

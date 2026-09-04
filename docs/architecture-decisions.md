# 아키텍처 결정 — 내재화(In-Process) vs 외재화(Service)

> 챗봇이 커지면서 단일 프로세스가 모든 책임을 지면 위험합니다.
> 이 문서는 **무엇을 메인 FastAPI 에 두고**, **무엇을 별도 서비스로 분리**하는지의 기준과 결정 결과를 정리합니다.
> 기본 아키텍처 그림은 [architecture.md](architecture.md) 참고.

## 결정 기준

각 모듈을 5가지 축으로 평가:

| 축 | "내재화" 신호 | "외재화" 신호 |
|---|---|---|
| **응답 지연** | <100ms 동기 처리 가능 | 수백 ms+ 또는 비동기 워커 적합 |
| **리소스 프로파일** | 경량 CPU, JSON I/O 위주 | GPU 필요 / 대용량 메모리 / CPU-바운드 장시간 |
| **장애 격리** | 실패해도 채팅 전체에 큰 영향 없음 | 실패 시 챗봇 전체 다운 위험 |
| **변경 빈도** | 자주 안 바뀜, 도메인 핵심 | 독립적으로 자주 배포해야 함 |
| **외부 인터페이스** | 내부 사용 전용 | 다른 시스템/팀이 호출 가능해야 함 |

## 결정 매트릭스

### A. 메인 FastAPI 프로세스 (내재화)

지연이 짧고 채팅 흐름과 강하게 결합된 책임만 남깁니다.

| 모듈 | 위치 | 이유 |
|---|---|---|
| Auth / 세션 | `features/auth` | 짧고 동기적 |
| Chat router (SSE 스트림) | `features/chat` | 사용자와 직접 연결, 다운되면 전체 다운 |
| **RAG 검색** (HNSW + 리랭킹) | `services/rag.py` | <100ms, 채팅 흐름과 강결합 |
| **Skills/Recovery 조회** | `services/learning.py` | DB 조회 + 키워드 매칭, 매우 경량 |
| Tool 디스패치 (web/stock 등) | `services/tool_registry.py` | 짧은 외부 API 호출 |
| **Theme 조회** | `features/themes` (신규) | 정적 데이터, 캐시 가능 |
| Agent 루프 (LLM 응답 스트림) | `services/agent.py` | SSE 와 강결합, 별도 프로세스 거치면 latency 증가 |

### B. 외재화 (별도 서비스/워커)

장시간 / GPU / 외부 의존성 큰 책임은 분리합니다.

| 모듈 | 분리 형태 | 인터페이스 | 이유 |
|---|---|---|---|
| **Embedding 서비스** | 별도 HTTP API | `POST /embed` (batch) | GPU 사용 가능, 독립 scaling |
| **문서 파싱 / OCR** | 워커 (큐 컨슈머) | DB job row 또는 큐 | CPU-바운드 분 단위, 메인 응답 안 막음 |
| **RAG 인덱싱** (embed → upsert) | 위 워커와 동일 | 큐 잡 | 업로드 후 비동기 |
| **Skill 자동 수집기** (skillsmp.com, getdesign.md) | 크론/워커 | 정기 동기화 | 외부 API rate-limit, 실패 격리 |
| **Self-improvement** (FailureLog → RecoveryTip 자동 생성) | 워커 | 배치 잡 (시간당) | LLM 호출 비용, 비동기 |
| **레포트 생성 / 이메일 발송** | 워커 | 큐 잡 | 사용자 응답 안 막음 |
| **MCP 서버 통합** (외부) | 별도 프로세스 또는 외부 서비스 | MCP 표준 프로토콜 | 표준 인터페이스, 위임 |

### C. 외부 SaaS / MCP 위임

사내에서 만들지 않고 외부 의존만:

| 책임 | 위임 대상 |
|---|---|
| LLM 호출 | OpenAI / Anthropic / Google (litellm) |
| 외부 도구 OAuth | Gmail, Google Calendar, Kakao |
| 외부 검색 | 자체호스팅 SearXNG 1순위 + DuckDuckGo 폴백 (web_search) |
| MCP 서버 (외부 도구 통합) | 표준 MCP 프로토콜로 외부 서버 연결 |
| 디자인 자료 | getdesign.md (시드) |
| 커뮤니티 스킬 | skillsmp.com (시드 + 주기 동기화) |

## 배포 토폴로지

### 작은 운영 (지금)
```
[user] → [FastAPI: chat + tools + RAG search + Skills + Themes]
 ↓
 [PostgreSQL + pgvector]
```
모든 게 한 프로세스. 가장 단순. ~100 동시 사용자.

### 중간 운영 (~수백 사용자)
```
[user] → [FastAPI (main)] ←─→ [Embedding Worker (GPU)]
 │ │
 ├──→ [Ingest Worker (CPU)] ←── ingest_jobs 큐
 │ │
 ├──→ [Curator Worker] ←── 주기 (skillsmp 동기화)
 │ │
 ↓ ↓
 [PostgreSQL + pgvector] ←── 공유
```
워커는 메인과 같은 DB 사용. 큐는 Postgres `LISTEN/NOTIFY` 또는 Redis.

### 대규모 (수천+ 사용자)
- FastAPI 를 k8s 다중 레플리카로 수평 확장
- 워커 풀 분리 (embedding, ingest, self-improvement, curator)
- 리드 레플리카로 DB 부하 분산
- pgvector → 별도 벡터 DB(Weaviate/Qdrant) 이전 검토

## 모듈별 결정

### 1. Skills 마켓플레이스 → **DB 우선 + YAML 시드 + 외부 수집기**
- 운영 중 사용자 추가 가능해야 함 → DB 가 진실 소스 (`skills` 테이블)
- 새 환경 초기화는 `backend/config/skills/*.yaml` 시드
- 자동 수집은 **별도 curator 워커** — skillsmp.com REST API, getdesign.md 사이트 정기 동기화
- `Skill.source` 필드로 출처 표시 (`manual`, `seed`, `skillsmp`, `getdesign`, `auto_generated`)

### 2. Theme (브랜드별 design.md) → **DB + 정적 시드**
- 운영 중 새 브랜드 테마 추가 가능
- 시드: `backend/config/themes/*.yaml` (디자인 토큰 + 마크다운 본문)
- 사용자는 본인/팀의 `active_theme_id` 설정
- 프론트는 GET `/themes/active` 한 번 호출 → CSS 변수 주입
- 무거운 렌더링/이미지 가공 등은 외재화 검토 (지금은 단순)

### 3. RAG 고도화 → **단계적**
- **이번 PR (메인 내재화)**: 적응형 인용 임계값, 키워드 추출 개선
- **다음 PR (메인 내재화)**: 반복적 질의 분해 (sufficiency-aware)
- **장기 (외재화)**: 레이아웃 인식 파싱 (deepdoc) — vision 모델 큼, GPU 워커

### 4. Self-Improvement (자가 학습) → **워커**
- `failure_logs` 적재는 메인에서 (각 실패마다 한 줄, 빠름)
- 누적 분석 + RecoveryTip 자동 생성은 **별도 self-improvement 워커**
- 워커는 시간당/일별 배치로 실행

### 5. LLM 호출은 **모두 외부**
- 모든 LLM 호출은 litellm 경유
- 사내 LLM 도입 시 `llm_runtime.py` 한 곳만 수정

## 큐/메시지 디자인

지금: **Postgres LISTEN/NOTIFY + 작업 테이블**
- 외부 의존성 0 추가
- 메인이 row INSERT → 워커가 NOTIFY 수신 → 처리

```sql
CREATE TABLE ingest_jobs (
 id UUID PRIMARY KEY,
 kind VARCHAR(40) NOT NULL, -- 'parse', 'embed', 'curate_skills', 'gen_recovery_tip'
 status VARCHAR(20) DEFAULT 'queued',
 payload JSONB,
 created_at TIMESTAMPTZ DEFAULT now(),
 started_at TIMESTAMPTZ,
 finished_at TIMESTAMPTZ,
 error TEXT
);
```

**옮길 때**: 큐 사용량이 많아지면 Redis Streams 또는 SQS 로 전환.

## 향후 검토

- **메시지 큐 선택**: Postgres NOTIFY 로 충분한가, Redis/Celery 로 갈 것인가?
- **워커 인증**: 메인-워커 간 인증은 짧은 JWT 공유 또는 mTLS
- **로그 집계**: 워커가 늘어나면 중앙 로깅(예: Loki) 필요
- **모니터링**: `metrics` 테이블에 LLM/tool 메트릭. 워커 메트릭도 같이 수집

---

이 결정은 [docs/changelog.md](changelog.md) 의 변경 이력과 함께 유지됩니다.

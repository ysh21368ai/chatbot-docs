# 모듈 맵

## 1. 백엔드 (`backend/app/`)

### 1.1 최상위 파일

| 파일 | 책임 |
| ---- | ---- |
| `main.py` | FastAPI 앱 생성, 미들웨어, 라우터 등록, 생명주기(`lifespan`) |
| `core/config.py` | `Settings` (pydantic-settings), 모델 카탈로그, 업로드 경로 해석 |
| `db/database.py` | SQLAlchemy 비동기 엔진 · `SessionLocal` · `Base` · `get_db` |
| `core/deps.py` | `get_current_user`, `require_team_admin`, `require_super_admin`, `require_team` |
| `db/models.py` | ORM 엔터티 (도메인별 모듈 패키지) |
| `schemas.py` | Pydantic I/O 스키마 |
| `core/security.py` | bcrypt 해시 · JWT 발급·검증 |
| `error_handlers.py` | 공통 예외 → HTTP 응답 변환 |
| `llm_user_errors.py` | LLM 오류를 사용자용 한국어 메시지로 변환 |
| `logging_setup.py` | 회전 파일 로거 설정 |
| `types_email.py` | 이메일 타입(Pydantic) |
| `middleware/request_logging.py` | 요청 로깅 + `X-Request-Id` |

### 1.2 라우터 (`features/<도메인>/router.py`)

백엔드는 feature-sliced 구조다. 각 도메인은 `backend/app/features/<도메인>/router.py` 로 분리되며 `main.py` 가 등록한다. 운영 핵심 도메인은 항상 등록되고, 개발 전용 도메인은 `FEATURE_*` 플래그가 꺼진 운영(prod)에서 `main.py` 가 등록 자체를 생략해 모든 경로가 404 가 된다(ADR-0009).

| 도메인 | Prefix | 주요 엔드포인트 |
| ---- | ------ | ---------------- |
| `auth` | `/auth` | `POST /register`, `POST /login`, `GET /me`, `GET /team` |
| `team` | `/team` | 멤버 조회/추가, 초대코드 로테이트, 감사 로그 |
| `documents` | `/documents` | 업로드, 목록, 삭제, 개인 문서 공유 |
| `chatbots` (신규) | `/chatbots` | 생성·수정·삭제, 목록(Public/Shared 포함), 문서/도구 연결 |
| `admin` (신규) | `/admin` | 승인 대기 목록, 승인/반려, 팀 생성(super_admin) |
| `conversations` | `/conversations` | 대화 목록·상세·삭제·제목 변경 |
| `chat` | `/chat` | `POST /stream` (SSE), `POST /regenerate` |
| `images` | `/images` | DALL·E 이미지 생성 |
| `models_catalog` | `/models` | 사용 가능한 LLM 목록 |
| `notices` | (prefix 없음) | 공지(`/notices*`) **+ 물어보기/기능요청 게시판 `FaqPost`(`/faq/posts`)** — FaqPost 는 `features/notices/router.py` 소속이다(faqs 도메인 아님). 새 글 등록 시 AI 자동답변(`faq_post_ai`)·관리자 이메일 알림(`graph_mailer`)을 `BackgroundTasks` 로 트리거 |
| `faqs` | `/chatbots/{id}/faqs` · `/faqs` | **챗봇별** Q&A 게시판(`ChatbotFaq`) — 질문/댓글/AI 답글(`faq_ai`)/채택. `/faq/posts` 와 별개 도메인 |
| `themes` · `usage` · `stats` · `metrics` | 각 prefix | 테마·사용량·통계·메트릭 (운영 포함) |
| `tools` *(개발 전용)* | `/tools` | 카탈로그 목록, 자격증명 등록, 커스텀 도구 등록 — `FEATURE_CUSTOM_TOOLS` off 시 미등록(404) |
| `skills` · `workflows` · `schedules` · `claude_code` · `workspace` *(개발 전용)* | 각 prefix | 스킬·워크플로·스케줄·Claude Code·작업공간 — 각 `FEATURE_*` off 시 미등록(404) |

### 1.3 서비스 (`services/`)

| 파일 | 역할 | 공개 함수 |
| ---- | ---- | -------- |
| `document_parser.py` | 원본 파일 → 평문 텍스트 | `extract_text(filename, data)` |
| `chunking.py` | 슬라이딩 윈도우 분할 | `split_text(text)` |
| `embeddings.py` | OpenAI Embedding 배치 호출 | `embed_texts(texts)` |
| `ingest.py` | 문서 파이프라인 오케스트레이션 | `process_document_job(document_id)` |
| `rag.py` | 하이브리드 검색(RRF) | `hybrid_search_chunk_ids()`, `fetch_chunk_contents()`, `build_context_snippets()` |
| `chatbot_rag.py` (신규) | 챗봇 스코프가 적용된 RAG | `search_for_chatbot(chatbot, user, db, query)` |
| `chatbot_service.py` (신규) | 챗봇 로딩/검증, 권한 체크 | `get_for_use()`, `_user_can_access()` |
| `tool_registry.py` *(도구 마켓=개발 전용·운영 미노출)* | 도구 실행 · 시드 카탈로그 | `dispatch(tool_slug, args, ctx)`, `seed_builtin_catalog()` |
| `agent.py` | 다단계 에이전트 루프, 툴 콜 | `run_agent(model, messages, workspace, *, allowed_tools=None, claude_code_options=None, max_steps=6)` |
| `code_sandbox.py` | Docker 파이썬 샌드박스 | `run_code(code, files, timeout)` |
| `web_search.py` | 웹 검색 — SearXNG 1순위·DDG 폴백 | `search_web(q, n)` |
| `stock.py` | 네이버 주식 시세/시세 이력 | `fetch_stock_price`, `fetch_stock_history` |
| `llm_runtime.py` | LiteLLM 래퍼 + 환경 구성 | `stream_chat`, `completion_kwargs` |
| `user_memory.py` | 대화에서 사용자 프로필 추출·저장 | `get_memory`, `update_memory_from_conversation` |
| `title_gen.py` | 대화 제목 자동 생성 | `generate_title` |
| `hr_sync.py` | 인사(HR) DB → 사용자 사전 동기화(매일 배치·SSO 보완) | `run_hr_sync()`, `register_hr_sync_job(scheduler)` |
| `graph_mailer.py` | Microsoft Graph 이메일 발송(물어보기 새 글 알림 등, best-effort) | `send_mail(...)` |
| `faq_post_ai.py` | 물어보기(FaqPost) 게시글 AI 자동 답변 | `generate_faq_post_answer(...)`, `AI_ANSWER_PREFIX` |
| `faq_ai.py` | 챗봇별 FAQ(ChatbotFaq) 게시판 AI 답글 | `generate_ai_answer(...)` |
| `bootstrap_admin.py` | 최초 기동 시 관리자 계정 시드 | `ensure_bootstrap_admin()` |

### 1.4 역할 책임 매트릭스 (신규 기능)

| 기능 | 레이어 | 담당 파일 |
| ---- | ---- | ---- |
| 가입 승인 | auth → admin | `features/auth/router.py`, `features/admin/router.py` |
| 3단계 권한 강제 | deps → 라우터 | `deps.py` (`require_super_admin`), 각 라우터의 Dependency |
| 챗봇 생성·편집 | 라우터 | `features/chatbots/router.py` |
| 챗봇 스코프 기반 RAG | 서비스 | `services/chatbot_rag.py` |
| 문서↔벡터스토어 동기화 | ORM cascade + 라우터 | `db/models.py` (cascade), `features/documents/router.py` |
| 도구 마켓/바인딩 | 라우터 + 서비스 | `features/tools/router.py`, `services/tool_registry.py` |
| HWP/HWPX 파서 | 서비스 | `services/document_parser.py` |
| 전역 로그아웃 | 프론트 | `shared/layout/AppShell.tsx` (+ `shared/lib/api.ts` 의 `logout()`) |

### 1.5 배경 작업 (스케줄러 · 생명주기)

`main.py` 의 `lifespan` 이 기동 시 스케줄러(`schedule_runner.start_scheduler`)와 지표 워커를 띄운다.

| 작업 | 트리거 | 담당 |
| ---- | ---- | ---- |
| HR 사용자 사전 동기화 | APScheduler cron(`HR_SYNC_CRON`, 기본 09:00 KST) — `HR_SYNC_ENABLED` 구성 시에만 등록 | `services/hr_sync.py::register_hr_sync_job()` (`start_scheduler` 안에서 호출) |
| 예약 쿼리(ScheduledQuery) | DB row → cron 잡 재등록 | `services/schedule_runner.py` |
| 물어보기 새 글 AI 자동답변 | 게시글 생성 시 `BackgroundTasks` | `features/notices/router.py` → `services/faq_post_ai.py` |
| 물어보기 새 글 이메일 알림 | 게시글 생성 시 `BackgroundTasks` | `features/notices/router.py` → `services/graph_mailer.py` |

---

## 2. 프론트엔드 (`frontend/`)

### 2.1 디렉토리 구조

```
frontend/
├─ app/
│ ├─ layout.tsx # 최상위 HTML 레이아웃 + Pretendard 폰트
│ ├─ fonts/PretendardVariable.woff2  # 한글 본문 로컬 번들(--font-pretendard)
│ ├─ globals.css # .page-shell(72rem) / .page-shell--narrow(56rem)
│ ├─ page.tsx # 랜딩
│ ├─ login/page.tsx # 로그인
│ ├─ register/page.tsx # 가입 (승인 대기 안내 포함)
│ └─ (workspace)/ # 로그인 이후 영역 (AppShell 씌움)
│ ├─ layout.tsx # AppShell 래퍼
│ ├─ chat/page.tsx # 대화 UI
│ ├─ documents/page.tsx # 문서 관리
│ ├─ chatbots/page.tsx # (신규) 챗봇 목록/생성
│ ├─ chatbots/[id]/page.tsx # (신규) 챗봇 편집·문서·도구 바인딩
│ ├─ admin/page.tsx # (신규) 가입 승인 / 팀 관리
│ ├─ team/page.tsx # 팀 감사
│ ├─ qa·notices·faq·mypage·guide/page.tsx  # Q&A·공지·FAQ·마이·가이드 (운영 포함)
│ ├─ tools/page.tsx # 도구 마켓 (개발 전용)
│ ├─ skills·workflows·schedules·workspaces/page.tsx  # (개발 전용)
│ └─ studio/page.tsx # 그림 만들기(이미지 생성, 개발 전용)
└─ shared/
 ├─ layout/ # AppShell.tsx · NavMenu.tsx · CommandPalette.tsx · AvatarMenu.tsx
 ├─ lib/ # api.ts · features.ts(isRagOnly·DEV_ONLY_PATH_PREFIXES) · theme.ts · userError.ts · track.ts
 ├─ ui/ # Toast · Skeleton · Alert · ConfirmDialog · Celebrate · SuccessFollowup · EmptyState · PageHeader …
 ├─ theme/ # ThemeProvider · BrandThemeProvider · RadialThemePicker …
 └─ providers/ # Providers.tsx
```

### 2.2 인증·상태 관리

- **저장소**: `localStorage['token']` 단일 키.
- **갱신**: `AppShell`이 마운트될 때 `GET /auth/me` 호출. 401/403이면 토큰을 지우고 `/login` 리다이렉트.
- **로그아웃**: 모든 authenticated 페이지의 헤더 우측에 버튼. 클릭 시 `clearToken() → router.replace("/login")`.
 - 헤더가 감춰진 `/chat` 화면에서도 사이드바 하단에 고정 로그아웃 버튼 노출(`ChatSidebarLogout` 보조 컴포넌트).

### 2.3 API 클라이언트 추가 계약

`lib/api.ts`에 다음 헬퍼를 추가합니다(중복 로직 방지):

```ts
export async function logout() {
 clearToken();
 if (typeof window !== "undefined") window.location.href = "/login";
}
```

### 2.4 페이지 ↔ API 매핑 (신규/갱신)

| 페이지 | 호출 API |
| ---- | ---- |
| `/register` | `POST /auth/register` (응답 메시지: "관리자 승인 대기 중") |
| `/admin` | `GET/POST /admin/approvals` · `POST /admin/teams` |
| `/chatbots` | `GET/POST /chatbots` |
| `/chatbots/[id]` | `GET/PATCH /chatbots/{id}` · `PUT /chatbots/{id}/documents` · `PUT /chatbots/{id}/tools` |
| `/tools` | `GET /tools` · `POST /tools/credentials` |
| `/chat` | `POST /chat/stream` (+ `chatbot_id`) |

---

## 3. 데이터 디렉토리 (`data/`)

```
data/
├─ uploads/{team_id}/{doc_id}_{filename} # 원본 파일
└─ logs/backend.log # 회전 로그
```

## 4. 인프라 (`docker-compose.yml` · `backend/sandbox/Dockerfile`)

- `db`: `pgvector/pgvector:pg16` (`5433`에 매핑, `init-db.sql`로 확장 설치)
- `backend/sandbox/Dockerfile`: 코드 실행 샌드박스(개발 전용)의 격리 Python 이미지 — `code_interpreter` 도구 자체는 ADR-0005 로 폐기(claude_code 대체)

## 5. 배포 의존성

| 컴포넌트 | 설치 방법 |
| -------- | -------- |
| PostgreSQL + pgvector | `docker compose up db` |
| Python 의존성 | `pip install -r backend/requirements.txt` |
| Node 의존성 | `npm install --prefix frontend` |
| HWP/HWPX 파서 | `pip install pyhwp hwp5` (또는 `olefile`로 자체 파서) — [document_rag.md](./document_rag.md#hwp-파싱-전략) 참고 |
| 샌드박스 이미지 | `docker build -t chatbot-sandbox backend/sandbox` |

### 실행

```bash
# 1) DB
docker compose up -d db

# 2) 백엔드
cd backend && bash run.sh # uvicorn app.main:app --reload --port 8001

# 3) 프론트
cd frontend && npm run dev # Next.js on :3001
```

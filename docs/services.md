# 서비스 · 기능 카탈로그

> "이 챗봇은 무엇을 할 수 있는가"를 한 페이지로 요약합니다. 상세는 각 참조 문서로 연결.

## 1. 서비스 지도 (Feature Map)

```mermaid
mindmap
 root((장금상선 챗봇))
 대화
 실시간 SSE 스트리밍
 이미지 첨부
 파일 첨부(문서 자동 RAG)
 대화 제목 자동 생성
 사용자 메모리(장기 선호도)
 문서 RAG
 업로드·파싱(PDF/DOCX/PPTX/XLSX/CSV/HWP/HWPX/…)
 pgvector HNSW + tsvector 하이브리드
 팀 문서·개인 문서·공유 문서
 자동 청크/임베딩(text-embedding-3-large)
 챗봇
 Private / Public(팀 내)
 Shared(팀 간 공유) - 구현됨(visibility=shared + chatbot_team_access)
 RAG 스코프 linked_only / owner_visible / team_all
 도구 바인딩
 도구
 builtin: web_search(SearXNG→DDG 폴백), image_generate, stock_price/history — code_interpreter 는 ADR-0005 로 폐기(claude_code 대체)
 oauth: Gmail, Kakao, Google Calendar, M365
 http: 사내 Webhook
 mcp: MCP 서버
 관리
 RBAC 4역할(member/team_auditor/team_admin/super_admin)
 가입 승인 / 비활성화
 팀 생성·초대코드
 대화 감사(super_admin 전용)
 인프라
 PostgreSQL + pgvector
 Docker 샌드박스
 Azure 배포(Container Apps ×3(backend·frontend·searxng) · PostgreSQL/pgvector · Blob · ACR, koreacentral — 라이브)
 CI/CD(GitHub Actions deploy.yml · OIDC · 무중단 롤링)
```

## 2. 제공 기능 요약표

| 영역 | 기능 | 참조 |
| ---- | ---- | ---- |
| 인증 | 이메일/비밀번호, JWT, 초대코드 가입, 승인/비활성화 | 관리자 문서(내부) · [rbac.md](./rbac.md) |
| 대화 | SSE 스트리밍, 멀티모달(이미지·파일), 모델 선택, 제목 자동 생성 | [api.md](./api.md#chat-chat) |
| 문서 RAG | 업로드→파싱→청크→임베딩, 삭제 cascade, 개인/팀/공유 스코프 | [document_rag.md](./document_rag.md) |
| 챗봇 | 시스템 프롬프트·모델·문서·도구 번들링, Private/Public/Shared | [chatbot.md](./chatbot.md) |
| 교차팀 공유 | 특정 팀이 만든 RAG 챗봇을 다른 팀이 사용 (구현됨: visibility=shared + chatbot_team_access) | [chatbot_sharing.md](./chatbot_sharing.md) |
| 도구 *(개발 전용·운영 미노출)* | builtin/oauth/http/mcp 4종, 자격증명 안전 저장 (FEATURE_CUSTOM_TOOLS·isRagOnly 가드) | [tools.md](./tools.md) |
| 코드 실행 *(개발 전용·운영 미노출)* | Docker 기반 Python 샌드박스, 네트워크 차단 | [sandbox.md](./sandbox.md) |
| 감사 | 전 팀 대화 열람 — **super_admin 전용**(2026-07-03 정책) | [rbac.md](./rbac.md) |
| 사용자 동기화 | 인사(HR) DB 를 매일 조회해 사용자 사전 생성/갱신·비활성화(SSO/JIT 보완) — `hr_sync.py`, 스케줄러 `register_hr_sync_job` | 관리자 문서(내부) |
| 물어보기 AI 자동답변 | 시스템 문의/기능요청 게시판(FaqPost) 새 글에 AI 가 1회 자동 답변(운영 매뉴얼·첨부문서 근거) — `faq_post_ai.py` | — |
| 챗봇 FAQ AI 답글 | 챗봇별 Q&A(ChatbotFaq) 게시판에 AI 답글(웹서치·RAG 옵션) — `faq_ai.py` | — |
| 물어보기 새 글 관리자 이메일 알림 | 새 게시글 등록 시 Microsoft Graph 로 관리자에게 이메일 발송(best-effort) — `graph_mailer.py` | — |
| 배포 | Azure + GitHub Actions CI/CD | 배포 문서(내부) |

## 3. 사용자 관점 시나리오

### (a) 팀원이 "인사 규정" 챗봇에 질문
1. 팀장이 `인사 규정 봇` 챗봇 생성(`visibility=public`, `rag_scope=linked_only`).
2. 문서(HWPX) 업로드 → 자동 청크·임베딩.
3. 팀원이 `/chat`에서 챗봇 선택 후 질문 → 팀장이 연결한 문서만 참조.

### (b) 팀장이 직원 가입을 승인
1. 직원이 초대코드로 회원가입 → `approval_status=pending`.
2. 팀장 `/admin` 진입 → "승인 대기" 섹션에 직원 표시.
3. [ 승인 ] 클릭 → `approval_status=approved` → 직원 로그인 가능.
4. 이직 시 [ 비활성화 ] → 로그인 즉시 차단, 이력은 보존.

### (c) 타 팀에서 제작한 챗봇 사용 (교차팀 공유)
1. A팀 팀장이 챗봇을 `shared` 가시성으로 전환 → 공유 대상 팀 선택.
2. B팀 팀원 `/chatbots`에서 해당 챗봇 표시(공유 뱃지).
3. B팀원이 질문 → 응답은 A팀 소유 문서 기반. (문서 직접 열람 권한은 없음 — 챗봇 경유만 허용)

## 4. 백엔드 서비스 모듈 ↔ 기능 맵

| 서비스 (`backend/app/services/`) | 담당 기능 |
| ---- | ---- |
| `ingest.py` | 업로드 문서 파이프라인(파싱→청크→임베딩) |
| `document_parser.py` | PDF/DOCX/HWPX 등 평문 추출 |
| `chunking.py` | 슬라이딩 윈도우 분할 |
| `embeddings.py` | OpenAI `text-embedding-3-large` 배치 호출 |
| `rag.py` | 하이브리드 검색(RRF) — 일반 |
| `chatbot_rag.py` | 챗봇 스코프 적용된 RAG |
| `chatbot_service.py` | 챗봇 CRUD·가시성·권한 체크 |
| `tool_registry.py` | 도구 카탈로그 + `dispatch()` |
| `agent.py` | 에이전트 루프(툴 콜/응답) |
| `code_sandbox.py` | Docker 샌드박스 실행 |
| `web_search.py` | 웹 검색 — 자체호스팅 SearXNG 1순위, DuckDuckGo 폴백 |
| `stock.py` | 네이버 시세/이력 |
| `llm_runtime.py` | LiteLLM 래퍼(모델 프로바이더 추상화) |
| `user_memory.py` | 사용자별 요약 메모리 |
| `title_gen.py` | 대화 제목 자동 생성 |
| `hr_sync.py` | 인사(HR) DB → 사용자 사전 동기화(매일 배치·SSO 보완, 스케줄러 `register_hr_sync_job`) |
| `graph_mailer.py` | Microsoft Graph 이메일 발송(물어보기 새 글 관리자 알림 등, best-effort) |
| `faq_post_ai.py` | 물어보기(FaqPost) 게시글 AI 자동 답변 생성 |
| `faq_ai.py` | 챗봇별 FAQ(ChatbotFaq) 게시판 AI 답글 생성 |
| `bootstrap_admin.py` | 최초 기동 시 super_admin 시드 |
| `schema_upgrade.py` | 기동 시 enum/컬럼 누락 보정 |

## 5. 프론트엔드 페이지 ↔ 기능 맵

| 라우트 | 기능 |
| ---- | ---- |
| `/login` · `/register` | 로그인·가입(초대코드) |
| `/chat` | 실시간 대화, 챗봇 선택, 파일/이미지 첨부 |
| `/documents` | 업로드·삭제·공유 |
| `/chatbots` | 내 챗봇 + 팀 공개 챗봇 + 공유 받은 챗봇(shared) |
| `/chatbots/[id]` | 편집: 기본/문서/도구 3 탭 |
| `/admin` | 승인 대기 / 팀원 관리(비활성화) / 팀 관리 |
| `/team` | 멤버 목록 · 감사 |
| `/qa` · `/notices` · `/faq` · `/mypage` · `/guide` | Q&A·공지·FAQ·마이페이지·가이드 (운영 포함) |
| `/tools` *(개발 전용)* | 도구 마켓, 자격증명 연결 |
| `/skills` · `/workflows` · `/schedules` · `/workspaces` *(개발 전용)* | 스킬·워크플로·스케줄·작업공간 |
| `/studio` *(개발 전용)* | 그림 만들기(이미지 생성) |

## 6. 품질 · SLO 초안 (합의 대상)

| 항목 | 목표(잠정) | 측정 포인트 |
| ---- | ---- | ---- |
| SSE 첫 토큰 지연 | p50 ≤ 1.5s, p95 ≤ 3.5s | `/chat/stream` 첫 이벤트 |
| 문서 업로드 처리 | 5 MB ≤ 30s로 `status=ready` | `process_document_job` 로그 |
| RAG 검색 지연 | p95 ≤ 400 ms (청크 1M) | `rag.py` 타이머 |
| 가용성 | 99.5%/월 | Azure Container Apps 지표 + healthz 외부 핑 |
| 샌드박스 실패율 | < 2% (비-사용자 오류) | `exit_code=-1` 비율 |

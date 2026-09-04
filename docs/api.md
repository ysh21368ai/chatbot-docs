# API 레퍼런스 요약

> 기본 경로: `http://localhost:8000` (개발), CORS는 `localhost:3000`, `localhost:3001`, `10.x.x.x:3001` 허용.
> 운영 도메인은 `BACKEND_ALLOWED_ORIGINS` env(콤마 구분)로 외부화 가능.
> 인증: `Authorization: Bearer <jwt>` 헤더만 사용합니다. (서버는 `HTTPBearer` 만 검사하며 쿠키 인증은 구현되어 있지 않습니다. 프론트는 토큰을 보관했다가 매 요청 헤더에 실어 보냅니다.)
>
> **Swagger UI(`/docs`)·`/openapi.json` 은 운영에서 비공개**입니다(33차 보안 점검 — 인터넷 스캐너가 주간 단위로 스키마를 수확). 개발 환경에선 그대로 열려 있고, 운영에서 한시 개방이 필요하면 `EXPOSE_API_DOCS=true` 로 설정하세요. 외부 연동사에 스펙을 줄 때는 개방보다 `openapi.json` 파일 전달을 권합니다.

## 헬스 (`/health` · `/healthz` · `/readyz`)

| 메서드 | 경로 | 설명 |
| ------ | ---- | ---- |
| GET | `/health` | 호환 — `{"ok": true}` |
| GET | `/healthz` | Liveness — 외부 의존성 검사 없음. K8s livenessProbe 권장 |
| GET | `/readyz` | Readiness — DB `SELECT 1` 핑. 실패 시 503 |

## Auth (`/auth`)

| 메서드 | 경로 | Body | 응답 | 설명 |
| ------ | ---- | ---- | ---- | ---- |
| POST | `/auth/register` | `{email, password, full_name, team_admin_email?, invite_code?, create_team_name?}` | `{access_token, approval_status}` | 가입. 운영은 `FEATURE_SIGNUP=false` 로 403(계정은 HR 동기화/SSO 로 생성). super_admin 발신 초대코드면 즉시 승인(team_admin), 그 외엔 `pending` |
| POST | `/auth/login` | `{email, password}` | `{access_token, refresh_token, approval_status}` | 로그인. **Content-Type 무관**하게 본문 수용(JSON/form-urlencoded/text/plain/헤더 없음 — `_lenient_login_body`). `email` 자리에 **이메일 또는 그룹웨어 아이디**(`@` 유무로 구분) — 비밀번호는 bcrypt 또는 위임(그룹웨어 무염 SHA-256 4회) 대조. 자격 불일치 401, 비활성/반려/승인 대기 403 |
| POST | `/auth/refresh` | `{refresh_token}` (JSON 본문, `RefreshRequest`) | `{access_token, refresh_token}` | 회전식 refresh — 새 refresh 토큰을 응답 본문으로 재발급. 쿠키가 아님. 이미 철회된 토큰 재제시(재사용 감지) 시 사용자 전 세션 철회 |
| POST | `/auth/sso/exchange` | `{token}` | `{access_token, refresh_token}` | 포털 SSO 핸드오프(HS256 공유 시크릿, jti 1회용). JIT 가입 포함 |
| GET | `/auth/config` | — | `{signup_enabled, sso_enabled}` | 프론트 기능 게이트 조회 |
| POST | `/auth/logout` | `{refresh_token?}` (JSON 본문, `LogoutRequest`) | 204 | 본문의 refresh 토큰을 DB 에서 철회(revoke). 본문 없어도(옛 클라이언트) 204 — 멱등. access JWT 는 상태가 없어 만료까지 유효하므로 프론트가 로컬에서 삭제 |
| GET | `/auth/me` | — | `UserOut` | 현재 사용자 |
| GET | `/auth/team` | — | `TeamOut` | 내 팀 |

## Admin (`/admin`)

| 메서드 | 경로 | 권한 | 설명 |
| ------ | ---- | ---- | ---- |
| GET | `/admin/me_role` | 로그인 | 내 역할 / 관리 권한 요약 |
| GET | `/admin/approvals` | team_admin+ | 승인 대기 사용자 목록 |
| POST | `/admin/approvals/{user_id}/approve` | team_admin+ | 승인 |
| POST | `/admin/approvals/{user_id}/reject` | team_admin+ | 반려 |
| GET | `/admin/users` | team_admin+ | 팀 전체 사용자(role·is_active 포함) |
| POST | `/admin/users/{user_id}/deactivate` | team_admin+ | **비활성화** (로그인 차단) |
| POST | `/admin/users/{user_id}/activate` | team_admin+ | **재활성화** |
| POST | `/admin/users/{user_id}/reset_password` | team_admin+ | 임시 비밀번호로 초기화(응답에 1회 노출) + 대상의 활성 refresh 토큰 전부 철회. 본인 계정·(팀장이) super_admin 대상은 400/403 |
| GET | `/admin/teams` | super_admin | 팀 목록 |
| POST | `/admin/teams` | super_admin | 팀 신규 생성 |
| POST | `/admin/teams/{team_id}/rotate_invite` | super_admin | 초대코드 재발급 |

팀장이 다른 팀 사용자를 조작하거나 본인 계정을 비활성화하면 400/403이 반환됩니다. 자세한 플로우는 관리자 문서(내부).

## Team (`/team`)

기존 유지. `POST /team/members`는 `team_admin+` 필요.

대화 감사(**super_admin 전용** — 2026-07-03 정책: 팀장/감사자는 자기 팀 대화도 열람 불가) — 수백 명 대비 *디렉토리 드릴다운*:
- `GET /team/audit/summary` → 대화한 구성원별 `{user_id,user_name,user_email,conversation_count,last_at}` (그룹 집계, 가벼움).
- `GET /team/audit/conversations?user_id=&offset=&limit=` → 특정 구성원의 대화만 lazy 로딩(`user_id` 미지정 시 팀 전체).
- `GET /team/audit/conversations/{id}/messages` → 대화 상세(기존).

## Documents (`/documents`)

| 메서드 | 경로 | 설명 |
| ------ | ---- | ---- |
| POST | `/documents/upload` | 멀티파트. `scope=team\|personal` + 파일 N개 |
| GET | `/documents` | 내게 보이는 팀 문서 목록(팀 공용 + 본인 개인 + 본인에게 공유된 문서만 — 타인 개인 문서는 격리). `q=`/`folder=` 필터 지원 |
| GET | `/documents/by-id/{id}` | 단건 메타(인용 뷰어용). 내 팀 문서이거나 **내가 쓸 수 있는 챗봇에 연결된 문서**면 반환 — 전사 공유 챗봇이 인용한 타 팀 문서를 열기 위함 |
| GET | `/documents/by-name?filename=` | 파일명으로 단건 해석. `document_id` 가 없는 *레거시 인용*, 원본이 삭제·재업로드되어 id 가 바뀐 경우용. 권한 규칙은 `by-id` 와 동일하고 챗봇 연결 문서를 우선 |
| DELETE | `/documents/{id}` | 삭제 → 청크/공유/챗봇링크 cascade 삭제 + 파일 unlink |
| POST | `/documents/{id}/share` | 개인 문서를 팀원에게 공유 |

## 답변 내보내기

| Method | Path | 설명 |
| ------ | ---- | ---- |
| POST | `/chat/export` | `{content, ext, filename}` → 파일 바이트. `docx·pdf·odt·rtf` 는 LibreOffice, `xlsx` 는 Markdown 표 파싱, 그 외 확장자는 텍스트 원문. 내용을 본문으로 받으므로 타인 대화 접근 불가. 상한 40만 자 지원 형식: docx·pdf·odt·rtf·xlsx·pptx·csv·html·md·txt. 원문 첫 줄 `<!-- export: {...} -->` 지시문(summary_row/chart/theme/sheet_title)이 있으면 서식에 반영. |

## i18n (`/i18n`)

해외 법인용 런타임 DOM 번역. 번역은 우리 LLM 이 수행하고 결과는 `translation_cache`(원문 해시 × 언어)에 전사 공유로 남는다.

| Method | Path | 설명 |
| ------ | ---- | ---- |
| GET | `/i18n/languages` | 지원 언어 17종(코드·자국어 표기·RTL 여부·해당 국가). **공개** — 로그인 전 화면에서도 언어를 고를 수 있어야 함 |
| POST | `/i18n/translate` | `{lang, texts[]}` → `{translations[], cached, translated}`. 순서·개수 보존. 로그인 시 캐시 미스는 LLM 번역, **비로그인은 캐시 히트만**(LLM 미호출) |
| POST | `/i18n/language` | 내 표시 언어 저장(`users.ui_language`). 다른 기기에서도 동일 언어로 열림 |
| GET | `/i18n/warmup-status` | 언어별 예열 진행률(화면 문구 중 캐시 보유 수·%). "언어 전환이 느리다"의 원인 = 캐시 미스이므로 이 수치로 판정 |

상한: 한 요청 120개 문구 / 문구당 1,500자 / 합계 24,000자.

## Chatbots (`/chatbots`, 신규)

자세한 내용은 [chatbot.md](./chatbot.md) 참조.

## Tools (`/tools`, 신규 · **개발 전용(운영 미노출, ADR-0009)**)

도구 마켓은 개발 티어 전용입니다. 운영(`APP_ENV=prod`, `FEATURE_CUSTOM_TOOLS` 미설정)에서는 `main.py` 가 라우터 자체를 등록하지 않아 모든 `/tools/*` 가 404 입니다. 자세한 내용은 [tools.md](./tools.md) 참조.

**보안 가드 (2026-05-20 추가)**:
- `POST /tools/custom` 와 `PATCH /tools/{id}` 의 `endpoint` URL 은 SSRF 검사를 통과해야 함. 사설 IP(10/172.16-31/192.168), loopback(127/::1), link-local(169.254 — AWS/GCP/Azure 메타데이터), 메타데이터 호스트명(`metadata.google.internal` 등) 은 자동 차단. 사내 tool-server 가 사설망에 있다면 `SSRF_ALLOWED_HOSTS=host1,host2` env 로 화이트리스트.
- `visibility=team` 또는 `public` 으로 등록·승격하려면 `team_admin` 이상 권한 필요. `private` 가시성은 일반 사용자도 등록 가능 (자기 자신만 호출).
- dispatch 직전에도 한 번 더 가드 — DNS rebinding 1차 방어.

## Conversations (`/conversations`)

| 메서드 | 경로 | 설명 |
| ------ | ---- | ---- |
| GET | `/conversations?q=...` | 내 대화 목록 |
| PATCH | `/conversations/{id}` | 제목 변경 |
| DELETE | `/conversations/{id}` | 삭제 (메시지 cascade) |
| GET | `/conversations/{id}/messages` | 메시지 목록 |

## Chat (`/chat`)

| 메서드 | 경로 | 설명 |
| ------ | ---- | ---- |
| POST | `/chat/stream` | SSE. Body: `{conversation_id?, chatbot_id?, model, message, use_rag, image_base64?, files?, claude_code_mode?, reasoning_effort?, claude_code_options?}`. **`claude_code_mode`/`claude_code_options`/`reasoning_effort` 는 개발 전용** — 운영(`FEATURE_CLAUDE_CODE` OFF)에서 `claude_code_mode=true` 면 403(`이 환경에서는 Claude Code 모드를 사용할 수 없습니다.`). `use_rag` 는 프론트가 *챗봇 선택 여부*로 보냄(일반 채팅=false·모델 지식만, 챗봇 대화=true, 실효값은 챗봇 설정과 AND — 30차) |
| POST | `/chat/regenerate` | 마지막 응답 재생성 |

`files[]` 항목은 `{filename, b64, relpath?}` 입니다. `relpath` 는 **폴더 첨부** 시 상대 경로(예: `src/app/page.tsx`)로, 클로드 코드 모드 워크스페이스에 디렉토리 구조를 그대로 재현하는 데 쓰입니다. 구조 보존은 백엔드 `app/core/pathsafe.safe_relative_path` 게이트를 통과한 경로에만 적용되며, traversal 위험 경로는 basename 으로 격하됩니다. `claude_code_mode=true` 면 LLM/RAG/메모리 매칭을 모두 건너뛰고 `run_claude_code_stream` 으로 직행하고, `reasoning_effort` 는 실제 CLI `--effort`(`low|medium|high|xhigh|max`)로 전달됩니다.

### SSE 이벤트 스키마

`backend/app/schemas_sse.py` 가 단일 진실원. 라우터 · 프론트 · 본 문서가 모두 같은 wire-format 을 참조합니다. 한 메시지는 정확히 하나의 top-level key 를 갖습니다.

| key | payload | 발생 시점 |
| --- | --- | --- |
| `t` | `"…토큰…"` | LLM 토큰이 흘러올 때 |
| `step_start` | `{n, tools}` | 에이전트가 한 step(도구 호출 묶음) 진입 시 |
| `step_end` | `{n, tools}` | step 종료 |
| `sandbox` | `{stdout, stderr, exit_code, image_b64?, files?}` | `code_interpreter` 결과 |
| `image` | `{url, prompt?, model?}` | `image_generate` 결과 |
| `pending_schedule` | `{options:[…]}` | `schedule_query` 도구가 확인 카드 제안 (사용자가 [허용] 클릭 시 등록) |
| `citations` | `[{idx, chunk_id, filename, content, score}]` | RAG 검색이 끝난 직후 1회 |
| `error` | `"사용자 친화 메시지"` | LLM/내부 오류 |
| `done` | `true` | 스트림 종료 |

예시:
```
data: {"t": "안녕하세요"}
data: {"step_start": {"n": 1, "tools": [{"name": "web_search", "args": {"q": "오늘 환율"}}]}}
data: {"sandbox": {"stdout": "1350.21\n", "exit_code": 0}}
data: {"citations": [{"idx": 1, "chunk_id": "abc", "filename": "policy.pdf", "content": "발췌…", "score": 8.4}]}
data: {"error": "외부 도구 응답 시간 초과"}
data: {"done": true}
```

새 이벤트를 추가할 때는 (1) `schemas_sse.py` 에 모델·`serialize_*` 추가, (2) `chat/router.py:_event_to_sse` 에 분기 추가, (3) 본 표 업데이트, (4) `tests/test_sse_events.py` 에 케이스 추가.

## Models (`/models`)

| 메서드 | 경로 | 설명 |
| ------ | ---- | ---- |
| GET | `/models` | 사용 가능한 LLM 카탈로그 |

## Notices (`/notices`)

| 메서드 | 경로 | 권한 | 설명 |
| ------ | ---- | ---- | ---- |
| GET | `/notices/latest?limit=3` | 로그인 | 채팅 사이드바용 최신 공지 요약 |
| GET | `/notices?include_inactive=false` | 로그인 | 공지사항 목록 |
| POST | `/notices` | team_admin+ | 공지 생성. `title`, `summary`, `body`, `is_pinned`, `is_active` |
| PATCH | `/notices/{notice_id}` | team_admin+ | 공지 수정/비활성화 |
| DELETE | `/notices/{notice_id}` | team_admin+ | 공지 삭제 |

공지는 `notices` 테이블에 저장되며, 팀 공지와 전역 공지를 함께 조회합니다.

## FAQ / Feature Requests (`/faq/posts`)

| 메서드 | 경로 | 권한 | 설명 |
| ------ | ---- | ---- | ---- |
| GET | `/faq/posts?status=&mine=false` | 로그인 | FAQ/기능 요청 게시글 목록 |
| POST | `/faq/posts` | 로그인 | 질문 또는 기능 요구사항 작성 |
| PATCH | `/faq/posts/{post_id}` | 작성자 또는 team_admin+ | 본문 수정, 관리자 답변, 상태 변경 |
| DELETE | `/faq/posts/{post_id}` | 작성자 또는 team_admin+ | 게시글 삭제 |

상태값은 `open`, `in_review`, `planned`, `answered`, `closed`입니다. 기존 챗봇별 FAQ(`/chatbots/{id}/faqs`)와 별도의 운영/요구사항 게시판입니다(핸들러는 `features/notices/router.py`).

## Usage (`/usage`)

| 메서드 | 경로 | 설명 |
| ------ | ---- | ---- |
| GET | `/usage/me/summary?window=24h\|7d\|30d\|month` | 내 LLM/임베딩/OCR 사용량, 토큰, USD 비용, KRW 환산액 요약 |

KRW 환산은 USD 환율 조회 성공 시 최신 값을 쓰고, 실패하면 `USD_KRW_FALLBACK_RATE` 환경값을 사용합니다.

## Stats (`/stats`, 신규)

| 메서드 | 경로 | 인증 | 설명 |
| ------ | ---- | ---- | ---- |
| GET | `/stats/public` | **없음** | 로그인 화면용 누적 통계: `total_chats`(=user 메시지 수)·`total_users`(활성)·`total_documents` + `top_contributors`(이름·대화수 Top 5). 프로세스 내 30초 TTL 캐시. 실패 시 0 폴백. |
| GET | `/stats/highlights` | 로그인 | 공지/대시보드 하이라이트: `recent_registrations`(최근 등록 도구/스킬/**챗봇** — kind·name·by·at, 최신 12) + `today_top_chatter`(오늘 KST 가장 활발한 사용자 또는 null). |
| GET | `/stats/leaderboard` | 로그인 | 이번 주(월 09:00 KST~) 리더보드. `week_start` + `categories[]`(대화왕/열정왕(비용)/자료왕) 각 Top 3(`rank·name·value·display`). 60초 캐시. |
| GET | `/stats/me/level` | 로그인 | 내 활동량(XP) 기반 40레벨: `tier`(초급/중급/고급/전문가)·`sublevel`(1~10)·`overall`(1~40)·`label`("고급 Lv.3")·`emoji`·`xp`·`chats`·`next_label`·`next_at`·`progress`(0~1). XP = 대화 + 제작물×15 + 받은 좋아요×8 + 문서×3. |

`/stats/highlights` 응답에 `achievements`(좋아요 받은 제작물·최고 활동가 레벨 등 사전 포맷 문구) 가 추가됩니다.

문서 목록 검색: `GET /documents?q=` 가 파일명·폴더·설명 부분일치(ILIKE)로 필터합니다(폴더 picker·자료 페이지 공용).

`/stats/public` 은 *읽기 전용 집계*만 노출하며 DB 스키마 변경이 없습니다. 사내 도구라 우수 기여자 이름을 노출하지만, 민감 데이터(이메일·내용)는 포함하지 않습니다.

## 공통 오류

| 상태 | 상황 |
| ---- | ---- |
| 400 | 검증 실패 / 잘못된 입력 |
| 401 | 토큰 누락/만료/위조 |
| 403 | 역할 불일치 / 승인 대기 / 비활성 계정 |
| 404 | 리소스 없음 (대화/문서/챗봇) |
| 429 | 레이트리밋(운영 추가 시) |
| 500 | 서버 내부 오류 (`expose_internal_errors=true`인 경우 상세 메시지 포함) |

## 인증 메모

- **access 토큰 60분** (`ACCESS_TOKEN_EXPIRE_MINUTES=60`) + **회전식 refresh 14일** (`REFRESH_TOKEN_EXPIRE_DAYS=14`). refresh 토큰은 **JSON 본문**으로 주고받습니다(쿠키 아님) — 서버는 opaque 랜덤 문자열의 sha256 해시만 DB(`refresh_tokens`)에 저장. refresh 재사용(철회된 토큰 재제시)이 감지되면 해당 사용자의 전 세션을 철회합니다.
- 로그인 식별자: 이메일 또는 그룹웨어 아이디(`users.login_id`). 비밀번호 검증은 자체 bcrypt **또는** 그룹웨어 위임(무염 SHA-256 4회 반복, `external_pass_sha256` — HR 동기화가 매일 갱신, 공식은 `core/security.py:groupware_password_hash`) 중 하나가 통과하면 성공.
- SSO: 포털이 발급한 HS256 토큰을 `/auth/sso/exchange` 로 교환(수명 상한 `SSO_TOKEN_MAX_AGE_SECONDS`, jti 1회용). 최초 진입 사용자는 JIT 로 생성됩니다.
- 로그아웃(`/auth/logout`)은 본문의 refresh 토큰을 DB 에서 철회(revoke) + 프론트 토큰 삭제로 완료됩니다. access JWT 자체는 상태가 없어 만료까지 유효합니다.

### 팀 감사

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/team/audit/chatbots` · `/team/audit/chatbots/{id}/users` · `/team/audit/chatbots/{id}/conversations?user_id=` · `/team/audit/chatbots/{id}/conversations/{conv}/messages` | 챗봇별 대화 감사(2026-09-04). 자기 팀의 팀 공개(public/shared) 챗봇 대화를 그 팀 구성원 모두가 열람(개인용 private 은 소유자·super_admin 만), super_admin 은 전체. 다른 팀 챗봇 403, 챗봇 밖 대화 404. |

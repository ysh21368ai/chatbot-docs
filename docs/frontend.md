# 프론트엔드 가이드

## 1. 라우팅 개요

```
app/
├─ (public) -- 인증 불필요 (파일 구조상 그룹 없음, 그냥 루트)
│ ├─ page.tsx /
│ ├─ login/page.tsx /login
│ └─ register/page.tsx /register
└─ (workspace) -- AppShell 공통 레이아웃
 ├─ layout.tsx
 ├─ chat/page.tsx /chat
 ├─ documents/page.tsx /documents
 ├─ chatbots/page.tsx /chatbots (신규)
 ├─ chatbots/[id]/page.tsx /chatbots/{id} (신규)
 ├─ admin/page.tsx /admin (신규)
 ├─ team/page.tsx /team
 ├─ qa/page.tsx /qa
 ├─ notices/page.tsx /notices
 ├─ faq/page.tsx /faq
 ├─ mypage/page.tsx /mypage
 ├─ guide/page.tsx /guide
 ├─ tools/page.tsx /tools (개발 전용)
 ├─ skills/page.tsx /skills (개발 전용)
 ├─ workflows/page.tsx /workflows (개발 전용)
 ├─ schedules/page.tsx /schedules (개발 전용)
 ├─ workspaces/page.tsx /workspaces (개발 전용)
 └─ studio/page.tsx /studio (개발 전용)
```

> **운영/개발 티어(ADR-0009)** — `/tools`·`/skills`·`/workflows`·`/schedules`·`/studio`·`/workspaces` 는 개발 전용이다. 운영 빌드(`NEXT_PUBLIC_FEATURE_SET=rag`)에서는 `shared/lib/features.ts` 의 `DEV_ONLY_PATH_PREFIXES` 와 `isRagOnly` 가드로 메뉴·명령 팔레트·가이드에서 숨겨지고, 백엔드도 해당 라우터를 등록하지 않아 차단된다.

> **레이아웃 표준** — 본문은 두 컨테이너로 통일한다: `.page-shell`(72rem · 목록/관리/그리드)와 `.page-shell--narrow`(56rem · 읽기/폼/상세). AppShell 헤더도 72rem 폭에 가장자리 정렬한다(`app/globals.css`).

## 2. AppShell

- `(workspace)` 하위 페이지 공통 헤더를 렌더.
- 헤더 요소: 브랜드 → 네비게이션(`NavMenu`) → 명령 팔레트(⌘K, `CommandPalette`) → 테마 피커 → 사용자 메뉴(`AvatarMenu` — 마이페이지·관리·**로그아웃**). 네비 항목은 `shared/layout/NavMenu.tsx` 의 `VISIBLE_GROUPS` 가 `isRagOnly` 로 분기 — 운영(rag) 빌드는 도구·스킬·워크플로·스케줄 등 개발 전용 항목을 숨긴다(`soon` 더미 포함).
- `/chat` 경로에서는 전폭 레이아웃을 위해 헤더가 감춰지지만, 좌측 사이드바 하단에 **공지 미니 목록**, **사용량 미니 카드**, **마이페이지 링크**, **로그아웃 버튼**을 배치합니다.

### 로그아웃 동작
`shared/layout/AppShell.tsx` 의 사용자 메뉴(`AvatarMenu`)와 `/chat` 사이드바에 공통으로 `shared/lib/api.ts` 의 `logout()` 콜백을 사용합니다.

```tsx
const handleLogout = () => {
 clearToken();
 router.replace("/login");
};
```

세션 종료 후 `/login`으로 즉시 리다이렉트됩니다.

## 3. 상태 관리

- **전역 상태 없음** — 각 페이지가 `useEffect`에서 필요한 API를 호출하고 `useState`로 보관.
- **인증 토큰**: `localStorage.token`. `shared/lib/api.ts` 의 `apiFetch()` 가 자동으로 `Authorization: Bearer` 헤더에 주입.
- **테마**: `ThemeProvider`가 `localStorage.theme` 관리.

## 4. 챗봇 사용 UI (채팅 페이지 변경점)

- 사이드바 상단에 "챗봇 선택" 드롭다운을 추가.
- 선택된 `chatbot_id`는 새 대화 생성 시 서버에 전달.
- 선택한 챗봇의 프로필(이름, 설명, 도구 뱃지)은 입력창 위 배너에 축약 표시.

## 5. 챗봇 편집 (`/chatbots/[id]`)

탭이 아니라 **단일 스크롤 폼**이며, 3개 `stitch-card` 섹션으로 나뉜다(`app/(workspace)/chatbots/[id]/page.tsx`):

1. **기본** — 이름, 설명, 모델, 시스템 프롬프트, 가시성, 근거 범위
   - **가시성**: 3값(`나만`/`우리 팀`/`선택 팀`) 라디오 카드 `VisibilityChooser` (`features/chatbots/VisibilityChooser.tsx`). `선택 팀`(=`shared`) 선택 시 바로 아래에 회사→팀 트리+검색 `TeamPicker` 가 펼쳐지고, 고른 팀은 저장 시 `extra_team_ids` 로 전달된다.
   - **근거 범위**: `RagScopeChooser` 가 `use_rag`(bool) + `rag_scope`(`linked_only`/`owner_visible`/`team_all`)를 한 컨트롤(안 봄/고른 문서만/내가 올린 문서 전체/우리 팀 문서 전체)로 통합한다.
2. **고를 문서** — `use_rag && rag_scope === 'linked_only'` 일 때만 렌더된다. 인라인 체크박스 대신 `DocumentPickerModal`(검색·폴더 트리·무한 스크롤 페이지네이션, 수만 건 대응)로 지정 문서를 고른다.
3. **장착된 도구** — 활성화 토글. 도구는 개발 전용(ADR-0009)이라 운영(rag) 빌드에선 미노출.

"저장" 시 각각 `PATCH /chatbots/{id}`(가시성·`extra_team_ids`·`use_rag`·`rag_scope` 포함) · `PUT /chatbots/{id}/documents` · `PUT /chatbots/{id}/tools`.

## 6. 도구 마켓 (`/tools`)

- 카드 그리드, 카테고리 탭 (communication / search / data / productivity / all).
- 각 카드: 아이콘, 이름, 카테고리, "연결" 버튼 또는 "연결됨" 뱃지.
- 연결이 OAuth 기반이면 서버가 반환한 consent URL로 새 창 이동.

## 7. 관리 콘솔 (`/admin`)

- 탭 1: 승인 대기 사용자 목록 → 승인/반려
- 탭 2: 팀 목록 (super_admin 전용) → 팀 신규 생성, 초대코드 복사
- 탭 3: 도구 카탈로그 (super_admin 전용, **개발 전용·운영 미노출**) → 도구 추가/비활성화

## 8. 공지 · FAQ · 마이페이지

- `/notices`: 작게 보이던 공지 영역을 별도 페이지로 분리. 관리자는 생성/수정/비활성화 가능.
- `/faq`(물어보기): 사용자가 기능 요구사항/질문을 작성하면 **AI 가 자동으로 1회 답변**(`[AI 자동답변]` 칩으로 사람 답변과 시각 구분)하고, 관리자가 이어서 답변/상태를 관리한다. 상단에는 **"특정 챗봇에 대한 문의는 그 챗봇 담당자에게 먼저"** 콜아웃(① 담당자 → ② 관리자 동선 분리)이 있다.
- `/mypage`: 계정 정보(역할·**소속=회사·팀** 표시) + **AI 개인화 메모리** 섹션(`/auth/me/memory` 조회·초기화) + LLM 사용량 요약(USD 비용과 KRW 환산액).
- `/chat`: 좌측 하단에 최신 공지 3개와 7일 사용량 요약을 축약 표시.

## 9. 로그아웃 UX 요구사항 (체크리스트)

- [x] AppShell 헤더에 로그아웃 버튼
- [x] 채팅 페이지 사이드바에 로그아웃 버튼 (헤더 미표시 상태에서도)
- [ ] (후속) 세션 만료 자동 감지 후 토스트로 안내 → `/login` 이동

## 10. 에러 UX

- `ApiUserError`로 모든 API 오류를 통일. 401/403은 AppShell이 `/login`으로 리다이렉트, 그 외는 페이지 상단 에러 배너로 표시.
- 자격증명 만료 같은 도구 실패는 채팅 메시지 안에 에러 블록으로 표시(에이전트 출력 경로).
- **채팅 스트림 실패** — user 말풍선은 지우지 않고 그대로 유지하며 말풍선 아래 "⚠ 전송 실패" 상태를 표시(`Msg.failed`, 클라이언트 전용). `FailureBanner` 재시도가 같은 내용으로 들어오면 새 말풍선을 추가하지 않고 실패 표시만 해제한다(중복 방지). 배너를 닫아도 내가 보낸 내용이 대화에 남는다.
- **첨부 경고는 비차단** — 폴더 첨부 상한(300개) 초과 등은 native `alert()` 대신 컴포저 위 인라인 경고 바(8초 자동 소거 + 수동 닫기). 이미지 교체 확인은 `confirmDialog` 사용(native `confirm()` 금지).

## 11. 피드백 · 로딩 프리미티브 (`shared/ui/`)

모두 "모듈 싱글톤 + 전역 호스트" 패턴 — 호스트는 AppShell 에 1회 마운트, 호출은 어디서든.

| 프리미티브 | 용도 | 호출 |
| --- | --- | --- |
| `toast()` | 일상 액션 피드백("저장했어요"/"삭제했어요"/조용한 실패). 우하단 비차단 스택, 자동 소거(성공 3.5s·에러 6s) | `toast("저장했어요")`, `toast(msg, "error")` |
| `celebrate()` | 첫 챗봇 생성 같은 *달성* 순간. 컨페티 + 격려. 드물게 | `celebrate("만들었어요! 🎉")` |
| `successFollowup()` | 성공 직후 *다음 행동* 제안(액션 링크 포함) | `successFollowup({title, actions})` |
| `confirmDialog()` | 파괴적/되돌리기 어려운 액션 확인. native `confirm()` 금지 | `await confirmDialog({title, danger:true})` |
| `Alert` | 페이지 상단 인라인 오류/안내(머무르는 상태) | `<Alert variant="error">…</Alert>` |
| `Skeleton` / `SkeletonList` | 로딩 — "불러오는 중…" 텍스트 대신 콘텐츠 형태 예고 | `<SkeletonList rows={4} />` |

**중복 금지 규칙**: 같은 순간에 toast+celebrate, toast+successFollowup, toast+인라인 Alert(같은 에러) 를 동시에 쓰지 않는다. 인라인 `Alert`/`setErr` 로 이미 표시되는 에러에 toast 를 또 띄우지 않는다. `<option>` 내부 로딩 텍스트는 스켈레톤 불가 — 텍스트 유지.

## 12. 의존성

- `next@15`, `react@19`, `tailwindcss@3.4`, `react-markdown`, `rehype-highlight`, `remark-gfm`, `@xyflow/react`(워크플로 2D 캔버스 · 개발 전용), `three`(3D 캔버스).
- 한글 본문 폰트는 Pretendard Variable 로컬 번들(`app/fonts/PretendardVariable.woff2`, `--font-pretendard`) — 라틴/숫자는 기존 폰트, 한글 글리프만 Pretendard 가 받친다.
- 레이아웃은 `.page-shell`(72rem) / `.page-shell--narrow`(56rem) 표준(`app/globals.css`).
- 새 페이지는 `use client` 지시어 필요(모두 인터랙티브).

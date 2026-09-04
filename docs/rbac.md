# RBAC · 권한 체계 · 가입 플로우

## 1. 역할 계층

```
super_admin (최종관리자)
 ↓ 팀을 생성하고 각 팀장을 임명할 수 있음
team_admin (팀장)
 ↓ 팀원을 초대/승인, 팀 리소스 감독
team_auditor (감사자, 레거시)
 ↓ (2026-07-03 정책 변경으로 대화 감사 권한 회수 — 열람 권한은 super_admin 전용)
team_member (팀원)
 - 일반 사용자: 본인 대화·개인 문서·개인 챗봇
```

- 상위 역할은 하위 역할의 작업을 수행할 수 있습니다.
- `team_auditor`는 역할 enum 은 유지되나, **대화 감사 권한이 회수되어**(2026-07-03) 현재 고유 권한이 없는 레거시 역할입니다.
- `super_admin`은 **팀 미소속 전역 계정**입니다(2026-07-03). 팀이 없어도 채팅·문서·챗봇 등
  제품 기능을 사용할 수 있으며, 이때 콘텐츠는 `team_id = NULL`(관리자 전역 버킷)로 저장되어
  어떤 팀에도 노출되지 않습니다. 일반 사용자는 팀이 없으면 종전대로 400 차단(`require_team`).
- 팀은 **회사(법인)별로 분리**됩니다 — `teams.company` / `users.company`(그룹웨어 법인명, models.py)로,
  같은 팀명이라도 회사가 다르면 별개 팀입니다(HR 동기화가 `회사+팀명` 키로 find-or-create).
  팀 격리(tenant isolation)는 회사 경계까지 포함합니다.

## 2. 계정 생성 시나리오

> 운영 계정은 **자체 가입이 아니라 인사(HR) 동기화 / 포털 SSO** 로 만들어집니다. 자체 회원가입
> (`POST /auth/register`)은 운영에서 `FEATURE_SIGNUP=false` 로 **403 차단**되며(auth/router.py:94-100),
> 과거의 "초대코드·팀장 이메일 승인 → 승인 대기" 플로우와 승인 대기 UI 는 제거되었습니다.

### (a) 최초 부팅 — super_admin 자동 생성
`config.BOOTSTRAP_ADMIN_ENABLED=true` 일 때 `services/bootstrap_admin.py`가 super_admin 계정을 생성합니다. 운영에서는 시드 후 `false`로 끕니다.

### (b) 인사(HR) DB 동기화 — 계정 사전 생성 (운영 기본)
매일 09:00 KST 사내 그룹웨어 인사 DB 를 조회해 사용자를 미리 생성/갱신합니다(`services/hr_sync.py`).
신규는 `role=member`·`approval_status=approved`로 생성되고, 이름·팀·회사(company)·그룹웨어 로그인 자격
(`login_id`·비밀번호 해시)이 인사 DB 정본으로 채워집니다. 동작 상세는 operations.md §11(사내 운영 문서).

### (c) 포털 SSO(JIT) — 첫 진입 시 생성
사내 포털에서 넘어오는 SSO 토큰으로 첫 진입 시 계정이 자동 생성(JIT)됩니다(operations.md §10(사내 운영 문서)). HR 동기화와 상호보완합니다.

### (d) 팀장이 팀원을 직접 등록 (예외 온보딩)
HR/SSO 로 커버되지 않는 예외는 팀장이 `POST /team/members` 로 이메일/비밀번호를 직접 설정해 생성 — 이 경우 `approval_status=approved`로 즉시 로그인 가능합니다.

### (e) 팀장이 팀원을 비활성화 / 재활성화
1. `/admin` → "팀원 관리" 섹션에서 [ 비활성화 ] 클릭
2. `POST /admin/users/{id}/deactivate` → `is_active=false`
3. 비활성 계정은 로그인 시 **403 "비활성화된 계정입니다."**
4. 이력(대화/문서 소유권)은 그대로 보존. 재활성화는 [ 재활성화 ] 버튼으로 즉시 복구.

제약:
- 본인 계정은 비활성화 불가(서버 400).
- `super_admin` 계정은 팀장이 비활성화 불가(서버 403).
- 다른 팀 사용자는 `super_admin`만 조작 가능.

## 3. 권한 매트릭스

| 작업 | super_admin | team_admin | team_auditor | team_member |
| ---- | :---------: | :--------: | :----------: | :---------: |
| 팀 생성 | | ❌ | ❌ | ❌ |
| 팀 삭제 / 초대코드 로테이트 | | (본인 팀) | ❌ | ❌ |
| 팀원 가입 승인/반려 | | (본인 팀) | ❌ | ❌ |
| 팀원 비활성화/재활성화 | | (본인 팀) | ❌ | ❌ |
| 팀 멤버 목록 보기 | | | | |
| 팀원 생성(직접) | | | ❌ | ❌ |
| 대화 감사(전체 인원) | | ❌ | ❌ | ❌ |
| 개인 대화/문서 | | | | |
| 팀 공용 문서 업로드 | | | ❌ | |
| 팀 공용 문서 삭제(타인) | | | ❌ | ❌ |
| 챗봇 생성 | | | | |
| Public 챗봇 공개 | | | | (본인 생성만) |
| 타인의 챗봇 수정 | (전체) | (본인 팀) | ❌ | ❌ |
| 타팀 챗봇 사용(대화) | (전체 — private 포함, RAG 는 연결 문서만) | shared 로 열린 것만 | shared 로 열린 것만 | shared 로 열린 것만 |
| 도구 카탈로그 관리 | | ❌ | ❌ | ❌ |
| 도구 자격증명 등록(본인 계정) | | | | |

## 4. 강제 지점 (enforcement)

- **의존성 주입**: `backend/app/core/deps.py`
 ```python
 async def require_super_admin(user = Depends(get_current_user)):
 if user.role != UserRole.super_admin:
 raise HTTPException(403, "최종관리자만 접근할 수 있습니다.")

 async def require_team_admin(user = Depends(get_current_user)):
 if user.role not in (UserRole.super_admin, UserRole.team_admin):
 raise HTTPException(403, "팀 관리자 이상만 접근할 수 있습니다.")
 ```

- **로그인 차단**: `POST /auth/login` 응답 시 `user.approval_status != approved` 이면 403 반환.

- **라우트 예시**:
 ```python
 @router.post("/admin/teams", dependencies=[Depends(require_super_admin)])
 @router.post("/admin/approvals/{user_id}/approve", dependencies=[Depends(require_team_admin)])
 ```

## 5. 프론트엔드 가드

- `AppShell.tsx`는 `/auth/me` 응답을 기반으로 역할에 따라 네비게이션 항목을 표시/숨김 처리합니다.
- `admin`, `tools` 관리 페이지는 역할 미충족 시 `/chat`으로 리다이렉트.
- 자체 회원가입이 꺼져 있어(`FEATURE_SIGNUP=false`) 승인 대기 화면(`/register/pending`)은 제거되었습니다.
  계정은 HR 동기화/SSO 로 `approved` 상태로 생성되며, 드물게 `pending`/`rejected`/비활성 계정이
  로그인하면 `/login`의 에러 메시지로 안내됩니다.

## 6. JWT 클레임

```json
{
 "sub": "<user_id>",
 "role": "team_admin",
 "team_id": "<team_id>",
 "approval": "approved",
 "exp": 1800000000
}
```

- 서버는 매 요청마다 DB에서 `User`를 다시 로드하여 상태를 재검증합니다(토큰 내 값은 UI 힌트 용도).

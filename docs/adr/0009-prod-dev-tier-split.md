# ADR-0009: 운영/개발 티어 분리 — 운영(Azure)은 RAG 챗봇 전용, 개발은 풀 기능 트렁크

- 상태: **승인**
- 날짜: 2026-06-10
- 관련: `CONTRIBUTING.md`(브랜치 전략·운영 보장 범위), ADR-0008(Azure 운영 배포), `infra/main.bicep`(태그·비용)

## 배경

회사 방침으로 챗봇 **운영 티어만 Azure 에 배포**한다(ADR-0008). 운영의 목적은 명확하다 — **팀 격리 문서 RAG 챗봇**. 반면 이 리포의 개발 라인은 Claude Code 모드·커스텀 도구·워크플로·스킬·스케줄·샌드박스 등 실험 기능을 계속 확장한다. 두 용도가 한 형상에 섞이면 운영 공격면·과금·매뉴얼이 전부 비대해진다.

## 결정

### 1. 티어 정의

| | 운영 (prod) | 개발 (dev) |
|---|---|---|
| 위치 | Azure `ai-part-rg` (koreacentral) | 로컬 dev 박스 (현행) |
| 목적 | **RAG 챗봇 전용** | 풀 기능 디벨롭 트렁크 |
| 기능 범위 | 운영 보장 4기능 — 채팅 대화 · 자료 업로드 · 챗봇 만들기 · RAG 문서 검색 | 전체 (Claude Code · 도구 · 워크플로 · 스킬 · 스케줄 · 샌드박스 · studio …) |
| 배포 단위 | Container Apps 3개(BE/FE/SearXNG) + PG Flexible + Blob + ACR (시크릿은 CA secrets — Key Vault 미도입) | docker compose(db) + uvicorn/next dev |
| 미배포 컴포넌트 | worker · tool-server · 샌드박스 docker · claude CLI — **운영 이미지에 미포함** | 전부 로컬 구동 |

### 2. 코드 분리 방식 — 포크 금지, 기능 플래그 (CONTRIBUTING 재확인)

코드베이스는 **하나**다. 운영 전용 브랜치에서 기능 코드를 물리적으로 들어내는 방식(영구 분기)은 머지 비용 때문에 금지(CONTRIBUTING.md 기존 결정 유지).

```
main      운영(stable) — 운영 보장 4기능 검증 코드만 승격. 태그 vX.Y.Z 로 Azure 배포
develop   개발 트렁크 — 모든 기능 PR 머지 대상 (Claude Code 포함 전 기능)
feature/* develop 분기 → PR → develop
hotfix/*  main 분기 → main + develop 동시 머지
```

- 운영 반영은 `develop → main` 승격 PR 로만. 승격 조건: CI green + **운영 4기능 스모크** + changelog 갱신.
- 운영에서 dev 기능은 **기능 플래그 OFF** 로 차단:
  - 백엔드 `core/config.py`: `APP_ENV=prod` 면 기본 OFF 인 플래그 군 신설 — `FEATURE_CLAUDE_CODE` `FEATURE_CUSTOM_TOOLS` `FEATURE_WORKFLOWS` `FEATURE_SKILLS` `FEATURE_SCHEDULES` `FEATURE_SANDBOX`. OFF 인 도메인의 라우터는 등록 자체를 생략(`main.py` include_router 분기) — 404 가 곧 차단이라 권한 구멍 원천 제거.
  - 프론트: `NEXT_PUBLIC_FEATURE_SET=rag|full` 1개로 단순화 — NavMenu·CommandPalette·guide(매뉴얼)·온보딩 미션이 이 값으로 항목을 필터.
- **Claude Code 모드는 개발 전용** — 운영 컨테이너에 claude CLI·docker 샌드박스를 설치하지 않으므로 플래그 우회로도 동작 불가(이중 안전).

### 3. 데이터 — 운영 DB 는 신규로 시작

- 운영 PostgreSQL Flexible 은 **빈 스키마로 신규 생성**. dev 박스의 대화 이력·테스트 계정·실험 데이터는 **이관하지 않는다** (개인 실험·더미가 섞여 있어 운영 데이터로 부적절).
- 운영에 필요한 것은 **문서(자료) 뿐** — 운영 개시 시 공식 문서를 새로 업로드·인덱싱한다(임베딩 소스 일관성도 확보, ADR-0008 §1).
- dev DB 는 로컬에 그대로 유지(개발 계속). 추후 특정 대화/챗봇 정의를 운영으로 가져가야 하면 선별 export 스크립트로(전체 이관 금지).

### 4. 매뉴얼/문서 분리

운영 사용자가 보는 매뉴얼에 Claude Code·워크플로 설명이 있으면 안 된다(존재하지 않는 기능 안내 = 신뢰 훼손, 감사에서 이미 같은 계열 결함 확인 — guide 의 /qa 폐기 화면 설명 등).

- **인앱 가이드**(`frontend/app/(workspace)/guide`): GROUPS 항목에 `feature` 키를 달고 `NEXT_PUBLIC_FEATURE_SET=rag` 면 RAG 4기능 항목만 렌더.
- **공개 docs**(`SinokorOfficial/chatbot-docs`): nav 를 "사용자 매뉴얼(운영)" / "개발자 문서(전 기능)" 로 2분할. `user_manual.md` 는 운영 4기능 기준으로 재작성.
- 운영 전용 절차(배포·계정·장애)는 기존 규칙대로 `operations.md`/`deployment.md`(공개 sync 제외)에.

### 5. 비용 관리 — 공유 RG `ai-part-rg` 에서 프로젝트별 분리 (ADR-0008 §7 보강)

공식 문서 재검증(2026-06-10) 결과를 명시한다:

- **가능하다**: Cost Analysis 에서 `Group by: Tag → project` 로 같은 RG 안 여러 프로젝트 비용을 분리 집계. 예산(Budgets)도 태그 필터로 프로젝트별 알림 설정 가능.
- **태그는 소급되지 않는다**: 태그 적용 *이후* 보고되는 사용량부터 비용 데이터에 붙는다. 과거 비용은 재집계되지 않으며, 태그를 나중에 바꾸면 변경 시점부터 새 값으로 집계된다(이전 기간은 옛 값/Untagged 로 남음). → **리소스 생성 시점부터 태그 부착**이 원칙(현 bicep 이 전 리소스에 `project/env/costCenter/managedBy` 부착 — 유지).
- **RG 태그 자체는 비용 데이터에 안 잡힌다**: 리소스에 직접 붙은 태그만 사용량 레코드에 들어간다. 일부 리소스는 태그를 사용량에 안 실어 보내기도 함. → **Cost Management 의 Tag inheritance 설정을 켤 것**(EA/MCA 지원): 구독·RG 태그가 하위 사용량 레코드에 자동 적용되어 누락 보완. 켜면 당월 1일 치부터 8~24시간 내 반영.
- 운영 절차: ① bicep `tags` 파라미터 유지(단일 소스, 재배포로 갱신) ② 구독 스코프에서 Tag inheritance Edit→활성화 ③ Cost Analysis 저장 뷰 "project=chatbot" 생성 ④ 월 예산 + 80%/100% 알림을 태그 필터로 설정.

## 결과

- 운영 공격면 최소화(도메인 라우터 미등록 + 바이너리 미포함), 과금 예측 가능(RAG 경로만), 매뉴얼-실기능 일치.
- 개발 속도 유지 — develop 은 기능 플래그 신경 안 쓰고 전 기능 개발, 승격 시점에만 4기능 스모크.
- 후속 구현 체크리스트:
  - [x] `core/config.py` FEATURE_* 플래그 + `main.py` include_router 분기 (2026-06-10 구현 — prod 기본 OFF·명시 오버라이드, claude_code 직행/도구 노출은 403/제외, 회귀 테스트 `tests/test_feature_flags.py`)
  - [x] `NEXT_PUBLIC_FEATURE_SET` — NavMenu/CommandPalette/guide/온보딩 퀘스트 필터 + 채팅의 클로드 코드 진입점 3곳(슬래시·?cc=1·복귀 CTA) 제거 (2026-06-10 구현, `shared/lib/features.ts`)
  - [x] 운영 Dockerfile 슬림화 — backend/Dockerfile 에 claude CLI·docker·샌드박스 의존이 원래 없음을 확인(설치는 dev 호스트에만)
  - [ ] `user_manual.md` 운영 4기능 기준 재작성 + chatbot-docs nav 2분할
  - [ ] Azure: Tag inheritance 활성화 + project=chatbot 예산 알림 (포털 작업)

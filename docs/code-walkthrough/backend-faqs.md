# faqs — 챗봇 Q&A + AI 답글

`features/faqs/router.py` 는 **챗봇별 Q&A 게시판(`ChatbotFaq` + `FaqComment`)만** 책임집니다.

> 흔한 오해: 이 파일이 공개 FAQ/문의 게시판(`FaqPost`, `/faq/posts`)까지 담당한다고 착각하기 쉽습니다.
> **`FaqPost` 는 `features/notices/router.py` 에 있습니다** (공지와 같은 파일). 아래 3장 참고.

- **챗봇별 Q&A** (`ChatbotFaq` + `FaqComment`) — 각 챗봇 페이지에서 "이 챗봇에 대한 질문" 을 모음. AI 자동 답글 가능. 라우트 prefix `/chatbots/{id}/faqs`, `/faqs/{id}`.

## 1. 챗봇 Q&A — 라우트 (`features/faqs/router.py`)

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/chatbots/{cb_id}/faqs` | 챗봇별 Q&A 목록 (정렬·필터 쿼리) |
| POST | `/chatbots/{cb_id}/faqs` | 질문 생성 (open 상태) — 챗봇 소유자에게 이메일 알림(아래 1-1) |
| GET | `/faqs/{id}` | 단건 + 댓글 묶음 |
| PATCH / DELETE | `/faqs/{id}` | 작성자/관리자만 |
| GET | `/faqs/{id}/comments` | 댓글 목록 |
| POST | `/faqs/{id}/comments` | 댓글 추가 (사람 답변) |
| POST | `/faqs/{id}/upvote` | 좋아요 토글 |
| POST | `/faqs/comments/{cid}/upvote` | 댓글 좋아요 토글 |
| POST | `/faqs/{id}/accept/{cid}` | 작성자가 채택 → status='answered' |
| POST | `/faqs/{id}/ai-reply` | AI 자동 답글 트리거 |

### 1-1. 새 질문 → 챗봇 소유자 이메일 알림 (2026-07-14)

`POST /chatbots/{id}/faqs` 는 질문을 저장한 뒤 `BackgroundTasks` 로 `_notify_chatbot_faq(faq.id)`
를 enqueue 합니다 — 챗봇 **소유자(담당자)** 에게 Microsoft Graph(`services/graph_mailer.send_mail`)
로 새 질문 알림을 보냅니다.

- 수신자: `chatbot.owner_user_id` 의 이메일 (`send_mail(..., to=[owner.email])`).
- 소유자가 곧 질문자 본인이면(자기 챗봇에 자문) 생략.
- best-effort — graph_mailer 미설정/발송 실패는 로그만 남기고 질문 등록을 막지 않음.

## 2. AI 답글 (`services/faq_ai.py`) — 챗봇 Q&A 전용

챗봇 Q&A(`ChatbotFaq`) 용 답글 생성기. 시그니처는 `ChatbotFaq` + 이전 댓글 트레일(prior comments)
기반이라 게시판(`FaqPost`)에는 그대로 못 씁니다 (그쪽은 3장의 `faq_post_ai.py`).

```python
async def generate_ai_answer(
    db: AsyncSession,
    faq: ChatbotFaq,
    user: User,
    use_web_search: bool = False,
    use_rag: bool = True,
) -> dict:
    ...
    model = os.environ.get("FAQ_AI_MODEL", "openai/gpt-5.4-mini")
    resp = await litellm.acompletion(model=model, messages=msgs, tools=tools or None, ...)
    return {"answer": answer, "ai_meta": {"model": model, "citations": ..., ...}}
```

- 챗봇 RAG(`use_rag`)·웹서치(`use_web_search`) 옵션, 이전 댓글 트레일은 마지막 12개만 사용.
- 답글은 새 `FaqComment` 로 insert + `author_kind='ai'` 표시. `ai_meta` 는 댓글 extra JSONB 에
  저장 — UI 가 "이 답변은 AI" 라벨 + 인용 카드 표시.

## 3. 공개 FAQ / 문의 게시판 (`FaqPost`) — **`features/notices/router.py`**

전사 문의·기능 요청 게시판. 라우트·핸들러가 **공지(notices)와 같은 파일**에 있습니다.

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/faq/posts` | 목록 — **회사·팀 무관 전체 공개**(3-1) |
| POST | `/faq/posts` | 문의 작성 — 게시 시 백그라운드 2건(3-2) |
| PATCH | `/faq/posts/{id}` | 작성자는 title/body, answer·status 는 관리자만(3-3) |
| DELETE | `/faq/posts/{id}` | 작성자/관리자 |

`FaqPostStatus` enum — open / answered / closed.

### 3-1. 열람 범위 — 전체 공개

`list_faq_posts` (notices/router.py:250-282) 는 예전엔 팀 스코프였으나, 현재는 **회사·팀과
무관하게 모든 로그인 사용자가** 목록/본문/답변을 열람합니다 (2026-07-14). `team_id` 쿼리
파라미터는 (주로 super_admin 이) 특정 팀만 골라 보려 할 때의 **선택 필터**일 뿐 강제 격리가
아닙니다. `status`, `q`(제목/본문 검색), `mine`, `limit` 도 필터로 지원.

### 3-2. 게시 시 백그라운드 작업 2건

`create_faq_post` (notices/router.py:363-384) 는 글 저장 후 `BackgroundTasks` 로 두 작업을 enqueue 합니다.

1. **AI 자동답변** — `_auto_answer_faq_post(post.id)` (notices/router.py:285-316). 전용
   `SessionLocal` 세션을 새로 열어 `faq_post_ai.generate_faq_post_answer` 로 답변 생성 후 저장
   (아래 4장). LLM 호출 중 관리자가 먼저 답/상태를 바꿨으면 덮어쓰지 않음. 실패해도 글은
   open(미답변)으로 남고 에러는 로그만.
2. **관리자 이메일 알림** — `_notify_new_faq_post(post.id)` (notices/router.py:319-360).
   Microsoft Graph(`services/graph_mailer.send_mail`)로 새 문의를 관리자에게 통지. best-effort.

### 3-3. 수정 권한 (notices/router.py:396-426)

`patch_faq_post`:

- **작성자 또는 관리자**만 수정 가능 (그 외 403).
- 작성자는 `title` / `body` 만 수정 가능. `status` / `answer` 필드가 본문에 있으면
  **관리자(team_admin 이상)가 아닌 한 403**.
- 관리자가 `answer` 를 채우면 `answer_by_user_id = user.id`, open 이면 status→answered.
  (AI 자동답변은 이와 달리 `answer_by_user_id=None` — 4장 규약.)

## 4. FaqPost 전용 AI 자동답변 (`services/faq_post_ai.py`)

게시판(`FaqPost`) 전용 답변 서비스. 게시글 `(title, body)` 단건만으로 1회성 답변을 생성합니다.

- **모델**: `FAQ_POST_AI_MODEL` > `FAQ_AI_MODEL` > `openai/gpt-5.4-mini` (faq_post_ai.py:214-215).
- **근거 자료**: 팀 업무문서 RAG 가 아니라 **서비스 사용 설명서**(`docs/user_manual.md`, 공개
  Pages 와 동일 원본)를 통째로 프롬프트에 포함(~4KB, 6시간 메모리 캐시). 본문의
  `/documents/{id}/download` 첨부 링크가 있으면 그 파싱 텍스트(OCR 포함)도 첨부.
- **표기 규약**: 저장 시 `AI_ANSWER_PREFIX = "[AI 자동답변] "` 프리픽스 + `answer_by_user_id=None`
  (사람 답변자 없음 → UI 가 AI 답변으로 구분). 상태는 answered 로 전환.
- 톤: 1인 운영이라 틀린 단정 금지 — 불확실하면 "관리자가 확인 후 답변" 폴백, 기능 요청은
  "접수되었습니다" 류(구현 시점·가능 여부 약속 금지).

> 2장 `faq_ai.py`(챗봇 Q&A용)와 별개 서비스입니다 — 시그니처(단건 게시글 vs 챗봇+댓글 트레일)와
> 근거(서비스 매뉴얼 vs 챗봇 RAG)가 다릅니다.

## 5. 좋아요 dedup

`ChatbotFaq.upvote_count` integer 컬럼 + 좋아요 토글 시 row-level 증감 패턴.

```python
@router.post("/faqs/{id}/upvote")
async def upvote(...):
    res = await db.execute(
        update(ChatbotFaq).where(ChatbotFaq.id == id).values(upvote_count=ChatbotFaq.upvote_count + 1)
    )
```

**개선 여지**: 사용자별 dedup 은 `UserSkillUpvote` / `UserToolUpvote` 같은 join 테이블 패턴이
따로 있음 ([models 워크스루](backend-models.md)). FAQ 좋아요도 동일 패턴으로 옮기는 게 다음 작업.

## 6. 함정·결정

- **두 게시판은 서로 다른 파일** — 챗봇 Q&A(`ChatbotFaq`)는 `features/faqs/router.py`,
  공개 문의 게시판(`FaqPost`, `/faq/posts`)은 `features/notices/router.py`. `/faq/posts` 를
  faqs 라우터에서 찾다 헤매기 쉬움. UI 도 `/faq`(게시판) vs `/qa`(허브) vs
  `/chatbots/{id}/faqs`(챗봇별)로 분리됨.
- **답글 채택은 작성자만** — `accept/{cid}` 가드로 작성자 또는 team_admin/super_admin 만.

## 관련

- 챗봇 모델 — [models 워크스루](backend-models.md)
- 챗봇 RAG 검색 — `services/chatbot_rag.py`
- AI 답글 메타 표시 — frontend `chatbots/[id]/faqs/page.tsx`

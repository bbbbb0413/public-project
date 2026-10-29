---
id: SPEC-086
title: 대화 세션 검색 범위 본문(turns.content) 확장 및 검색 스니펫 제공
status: done
targets: [python-server]
stages: [backend, qa]
priority: normal
---

## 배경 / 문제
현재 RAG 대화 세션 검색 기능은 첫 질문의 앞부분 제목으로만 검색이 제한되어 있어, 사용자가 과거 대화에서 확인했던 핵심 지식이나 후속 질문·답변 내용을 키워드로 검색할 수 없는 심각한 사용성 결함이 존재한다.

`public-python-server/src/ai_service/rag/schemas.py:144-147` (`ConversationSession.create`)을 보면 세션 생성 시 `title = first_question[:MAX_TITLE_LENGTH]`로 첫 번째 질문의 앞 50자만 제목으로 고정 저장된다. 따라서 50자를 넘어가는 질문 뒷부분, 2번째 턴 이후의 후속 질문, 그리고 AI가 제공한 상세 답변 내용은 제목(`title`)에 전혀 반영되지 않는다.

그러나 `public-python-server/src/ai_service/rag/repository.py:52-62` (`ConversationSessionRepository.find_by_user_id`)에서는 키워드 검색 시 `query["title"] = {"$regex": re.escape(keyword.strip()), "$options": "i"}`와 같이 오직 세션의 `title` 필드에 대해서만 정규식 부분 일치를 수행하고 있다. 이로 인해 `public-python-server/tests/unit/rag/test_session_repository_ownership.py:79-99` (`test_목록조회_키워드가_있으면_title_regex조건을_추가한다`)에서도 `title` 단일 필드 매칭만을 전제로 검증이 고정되어 있다.

실제 사용 흐름에서 `public-front/src/components/AiService.tsx:1047-1054` (`AiService`)의 사이드바 검색창("대화 검색...")에 과거 대화에서 얻었던 답변 키워드(예: "환불 기한", "배송 지연 규정")를 입력하면, 제목에 해당 키워드가 포함되지 않은 세션은 본문에 분명히 해당 내용이 존재함에도 불구하고 검색 결과에서 완전히 누락된다(`session-empty`, "검색 결과가 없습니다").

또한 프론트엔드 UI `public-front/src/components/AiService.tsx:1097-1102` (`AiService`) 및 `public-front/src/api/ai.ts:274-279` (`SessionOut`)에는 이미 검색 결과 스니펫을 렌더링하기 위한 `s.snippet` 인터페이스와 스타일(`public-front/src/components/AiService.css:1666-1673`)이 마련되어 있으나, 백엔드 `public-python-server/src/ai_service/rag/schemas.py:392-407` (`SessionOut`)에서 `snippet` 필드를 제공하지 않아 검색어와 일치하는 본문 문맥을 전혀 보여주지 못하고 있다.

이번 분기 제품 방향인 "답변을 믿을 수 있게 만든다 — 사용자가 스스로 확인할 수 있어야 한다"를 달성하기 위해서는, 이전에 검증하고 확인했던 지식 답변을 본문 키워드로도 신속하게 다시 찾아 재확인할 수 있도록 세션 검색 범위를 대화 턴 본문(`turns.content`)까지 확장하고 매칭된 문맥 스니펫을 응답에 담아 제공해야 한다.

## 요구사항
- [ ] `ConversationSessionRepository.find_by_user_id`에서 유효한 검색 키워드(`keyword`)가 주어졌을 때, 세션 제목(`title`)뿐만 아니라 대화 턴의 본문(`turns.content`)까지 검색 대상에 포함하는 MongoDB `$or` 조건을 생성하여 쿼리한다.
- [ ] 검색 조건 확장 시에도 최상위 소유자 격리(`userId: user_id`) 조건이 반드시 유지되어야 하며, 타인의 세션 또는 턴 본문이 검색 결과에 노출되지 않아야 한다.
- [ ] 검색 키워드가 `None`이거나 양쪽 공백 제거 후 빈 문자열(`""`)인 경우 검색 조건(`$or`)을 부가하지 않고 소유자의 전체 세션 목록을 최신 수정일(`updatedAt desc`) 순으로 반환한다.
- [ ] 특수문자가 포함된 검색어가 입력되더라도 정규식 이스케이프(`re.escape`)를 적용하여 안전하게 검색되고 정규표현식 오류(500)가 발생하지 않아야 한다.
- [ ] `SessionOut` Pydantic 모델에 선택적 필드 `snippet: str | None = None`을 추가하고, `SessionOut.from_domain` 매핑 시 검색 키워드(`keyword`)가 주어지면 매칭된 대화 턴(`turns.content`)의 전후 문맥을 최대 100자 내외로 추출하여 `snippet` 필드에 바인딩한다.
- [ ] 검색어 매칭 턴이 없거나 키워드가 주어지지 않은 일반 목록 조회 시 `snippet`은 `None`으로 반환하거나 안전한 기본값으로 처리한다.
- [ ] 스니펫 문자열 내의 개행 문자(`\n`, `\r`)는 단일 공백으로 치환하고 연속된 공백을 정리하여 한 줄 텍스트 형태로 제공한다.
- [ ] `public-python-server/src/ai_service/rag/router.py`의 `get_sessions` 엔드포인트에서 유효 검색어(`effective_keyword`)를 `SessionOut.from_domain`에 전달하여 클라이언트에 `snippet`이 포함된 응답을 반환한다.
- [ ] `test_session_repository_ownership.py` 및 `test_rag_session_routes.py`에 본문 키워드 매칭 검색, 소유자 격리 검증, 스니펫 추출 단위/E2E 테스트를 추가 및 갱신한다.

## 비요구사항 (Out of scope)
- Elasticsearch, OpenSearch, Atlas Search 등 별도의 외부 전문 검색 엔진 도입이나 인프라 변경은 수행하지 않는다.
- 기존 게이트웨이 및 프론트엔드의 세션 검색 쿼리 파라미터(`q`, `keyword`) 전달 규약이나 API 엔드포인트 경로를 변경하지 않는다.
- 대화 세션 상세 조회(`GET /ai/rag/sessions/:sessionId`)나 대화 턴 삭제/수정 등 세션 검색 목록 외의 다른 세션 API 로직은 수정하지 않는다.
- 프론트엔드 컴포넌트(`AiService.tsx`)의 사이드바 레이아웃이나 검색 폼 UI 구조를 새로 변경하지 않는다.

## 백엔드
- `public-python-server/src/ai_service/rag/repository.py`
  - `ConversationSessionRepository.find_by_user_id`: 키워드가 존재할 때 `query["$or"] = [{"title": regex_cond}, {"turns.content": regex_cond}]` 형태로 쿼리 딕셔너리를 구성한다.
- `public-python-server/src/ai_service/rag/schemas.py`
  - `SessionOut`: `snippet: str | None = Field(default=None, alias="snippet")` 필드 추가.
  - `SessionOut.from_domain`: `keyword: str | None = None` 매개변수를 지원하여, 키워드가 존재하고 세션의 `turns` 중 `content`에 키워드가 포함된 턴이 있으면 해당 키워드 주변 텍스트(예: 앞 20자, 뒤 60자 내외, 최대 100자)를 추출하여 `snippet`에 할당.
- `public-python-server/src/ai_service/rag/router.py`
  - `get_sessions`: `[SessionOut.from_domain(s, effective_keyword) for s in sessions]` 형태로 스니펫 매핑 파라미터 전달.
- `public-python-server/tests/unit/rag/test_session_repository_ownership.py`
  - 키워드 검색 시 쿼리에 `userId` 격리와 함께 `title` 및 `turns.content` `$or` 조건이 올바르게 전달되는지 검증하는 단위 테스트 갱신.
- `public-python-server/tests/e2e/test_rag_session_routes.py`
  - 첫 질문 제목에 키워드가 없더라도 2번째 턴 이후의 질문이나 AI 답변 본문에 키워드가 포함된 세션이 정상 검색되고 `snippet` 필드가 포함되어 응답되는지 검증하는 E2E 테스트 추가.

## 수용 기준 (Acceptance Criteria)
- Given 사용자 `user-1`의 대화 세션에 첫 질문 제목은 "회사 복지 제도"이고 2번째 턴의 AI 답변 본문에 "연차 유급휴가는 입사 1년 후 15일 부여됩니다"가 포함되어 있을 때
  When `user-1`이 `keyword="유급휴가"`로 세션 목록을 조회하면
  Then 해당 세션이 검색 결과 목록에 1건 포함되어 반환되어야 하고, 반환된 `SessionOut`의 `snippet`에 "유급휴가" 키워드 및 주변 문맥이 포함되어야 한다.
- Given 사용자 `user-2`의 대화 세션 본문에 "유급휴가" 키워드가 포함되어 있을 때
  When 사용자 `user-1`이 `userId="user-1"`, `keyword="유급휴가"`로 세션 목록을 조회하면
  Then 사용자 `user-2`의 세션은 검색 결과에 절대로 노출되지 않아야 한다.
- Given 키워드가 `None`, 빈 문자열 `""`, 또는 공백으로만 이루어진 `"   "`일 때
  When 세션 목록 조회를 요청하면
  Then `$or` 조건 없이 소유자의 전체 세션이 최신 `updatedAt` 역순으로 반환되어야 하고, `snippet` 필드는 `None`으로 안전하게 반환되어야 한다.
- Given 검색어에 정규표현식 특수문자(예: `"매출 (Q3)"`, `"[긴급] 배송"`, `"$100"`)가 포함되어 있을 때
  When 세션 목록 조회를 요청하면
  Then 정규식 파싱 오류(500 Internal Server Error) 없이 이스케이프 처리되어 정확히 부분 일치하는 세션을 반환해야 한다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/repository.py:52-62` (`ConversationSessionRepository.find_by_user_id`)
- 세션 제목 생성 로직: `public-python-server/src/ai_service/rag/schemas.py:144-147` (`ConversationSession.create`)
- 세션 목록 DTO 매핑부: `public-python-server/src/ai_service/rag/schemas.py:392-407` (`SessionOut.from_domain`)
- 세션 목록 조회 라우터: `public-python-server/src/ai_service/rag/router.py:30-41` (`get_sessions`)
- 리포지토리 소유권 단위 테스트: `public-python-server/tests/unit/rag/test_session_repository_ownership.py:79-99` (`test_목록조회_키워드가_있으면_title_regex조건을_추가한다`)
- 프론트엔드 스니펫 렌더링부: `public-front/src/components/AiService.tsx:1097-1102` (`AiService`)

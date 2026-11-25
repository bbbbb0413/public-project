---
id: SPEC-115
title: 대화 세션 삭제 시 연관 답변 북마크 및 피드백 데이터 연쇄 정리와 고아 보관 항목 방지
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
대화 세션 삭제 기능(SPEC-058)이 배송된 이후, 사용자가 세션을 삭제해도 해당 세션에 종속된 답변 북마크와 피드백 데이터가 백엔드 데이터베이스에 그대로 방치되는 결함이 존재한다.
`public-python-server/src/ai_service/rag/service.py:43-47` (`SessionService.delete_session`)에서는 세션 삭제 시 `ConversationSessionRepository.delete_by_id_for_user`만 호출하여 `conversation_sessions` 컬렉션의 세션 도큐먼트만 삭제한다.
`public-python-server/src/ai_service/rag/repository.py:89-99` (`ConversationSessionRepository.delete_by_id_for_user`)는 세션 컬렉션 단건 삭제만 수행하며, `BookmarkRepository` 및 `AnswerFeedbackRepository`(`public-python-server/src/ai_service/feedback/repository.py:55-64` (`AnswerFeedbackRepository.delete`))에는 특정 세션에 속한 모든 데이터를 한 번에 정리하는 연쇄 삭제 인터페이스가 구현되어 있지 않다.
이로 인해 `public-python-server/src/ai_service/rag/router.py:124-135` (`delete_session`) 엔드포인트를 통해 세션이 정상 삭제되더라도, `bookmarks` 컬렉션과 `feedbacks` 컬렉션에는 존재하지 않는 세션을 참조하는 고아(Orphan) 레코드가 영구적으로 남게 된다.
결과적으로 사용자가 사이드바 보관함(`bookmarks`) 탭을 열었을 때 이미 삭제한 세션의 북마크가 계속 노출되며, `public-front/src/components/AiService.tsx:1750-1758` (`bookmarks.map`)에서 해당 보관 항목을 클릭하면 `handleLoadSession`이 호출되어 404 세션 없음 오류(`SessionNotFoundError`)가 발생하고 "세션을 불러오는 데 실패했습니다."라는 오류 메시지가 나타난다.
또한 `public-front/src/components/AiService.tsx:703-718` (`handleConfirmDeleteSession`)에서도 세션 삭제 성공 시 `sessions` 상태만 갱신하고 `bookmarks` 상태는 정리하지 않아, 세션 삭제 직후 보관함 탭에 삭제된 세션의 북마크 항목이 실시간으로 잔류한다.
이는 이미 배송된 대화 세션 삭제(SPEC-058), 답변 평가 피드백(SPEC-024, SPEC-068), AI 답변 북마크(SPEC-082, SPEC-102, SPEC-109) 기능의 정합성을 깨뜨리고 사용자 경험을 저해하므로, 세션 삭제 시 종속 데이터를 안전하게 연쇄 정리하고 화면 상태를 동기화해야 한다.

## 요구사항
- [ ] `BookmarkRepository`에 특정 사용자의 특정 세션에 속한 모든 북마크 레코드를 삭제하는 `delete_by_session_for_user(session_id: str, user_id: str) -> int` 메서드를 추가한다.
- [ ] `AnswerFeedbackRepository`에 특정 사용자의 특정 세션에 속한 모든 피드백 레코드를 삭제하는 `delete_by_session_for_user(session_id: str, user_id: str) -> int` 메서드를 추가한다.
- [ ] `SessionService`의 생성자 또는 의존성 주입에 `BookmarkRepository`와 `AnswerFeedbackRepository`를 연계하여, `delete_session` 호출 시 세션 삭제가 성공한 후 해당 세션의 북마크와 피드백을 연쇄 삭제하도록 구현한다.
- [ ] 세션이 존재하지 않거나 요청자의 소유가 아닌 경우 `SessionNotFoundError`가 발생해야 하며, 타인의 북마크나 피드백 데이터가 삭제되는 일이 없도록 사용자 격리(`userId: user_id`)를 엄격히 보장한다.
- [ ] 프론트엔드 `AiService.tsx`의 `handleConfirmDeleteSession`에서 세션 삭제 성공 시, 로컬 `bookmarks` 상태에서도 삭제된 `sessionId`에 해당하는 보관 항목들을 즉시 필터링 제거(`prev.filter((b) => b.sessionId !== sid)`)한다.
- [ ] 보관함 탭에서 세션 상세 조회(`getSessionDetail`)가 404 에러로 실패하는 경우(레거시 고아 북마크 클릭 등), 안내 메시지("존재하지 않거나 삭제된 대화의 보관 항목입니다.")를 노출하고 해당 고아 북마크를 보관함 목록 상태에서 정리 제거한다.

## 비요구사항 (Out of scope)
- 다른 사용자의 데이터 정리나 관리자 전용 세션 일괄 삭제 기능 구현
- MongoDB 다중 도큐먼트 분산 트랜잭션(Replica Set Transaction) 설정 강제 또는 샤딩 구성 변경
- 보관함 탭 내 다중 선택 일괄 삭제 UI 추가
- 피드백 통계 집계 대시보드 신규 개발

## 백엔드
- `public-python-server/src/ai_service/rag/repository.py`:
  - `BookmarkRepository` 클래스에 `delete_by_session_for_user(session_id: str, user_id: str) -> int` 메서드 추가
  - `{"sessionId": session_id, "userId": user_id}` 조건으로 `delete_many` 수행 후 삭제된 도큐먼트 수 반환
- `public-python-server/src/ai_service/feedback/repository.py`:
  - `AnswerFeedbackRepository` 클래스에 `delete_by_session_for_user(session_id: str, user_id: str) -> int` 메서드 추가
  - `{"sessionId": session_id, "userId": user_id}` 조건으로 `delete_many` 수행 후 삭제된 도큐먼트 수 반환
- `public-python-server/src/ai_service/rag/service.py`:
  - `SessionService` 생성자에 선택적 또는 필수 의존성으로 `BookmarkRepository` 및 `AnswerFeedbackRepository`(또는 관련 콜백/포트) 주입 지원
  - `delete_session` 메서드에서 세션 삭제 성공 시 `bookmark_repo.delete_by_session_for_user` 및 `feedback_repo.delete_by_session_for_user` 호출
- `public-python-server/src/ai_service/rag/dependencies.py`:
  - `get_session_service` 팩토리 함수에서 `BookmarkRepository` 및 `AnswerFeedbackRepository`를 주입받아 `SessionService` 생성

## 프론트엔드
- `public-front/src/components/AiService.tsx`:
  - `handleConfirmDeleteSession`: 세션 삭제 API 호출 성공 시 `setBookmarks((prev) => prev.filter((b) => b.sessionId !== sid))`를 호출하여 현재 보관함 목록에서 삭제된 세션 항목 즉시 제거
  - `handleLoadSession`: 세션 상세 조회 실패 시 타깃 턴 인덱스나 북마크 클릭 컨텍스트를 고려하여 세션이 없으면 해당 북마크 항목을 `bookmarks` 상태에서 안전하게 제외 처리하고 사용자 안내 제공

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 북마크와 답변 피드백이 등록된 대화 세션을 보유하고 있을 때
  When 사용자가 해당 대화 세션을 삭제하면
  Then `conversation_sessions` 컬렉션에서 세션이 삭제되고, 연관된 `bookmarks`와 `feedbacks` 도큐먼트도 함께 데이터베이스에서 삭제된다.
- Given 사용자가 세션을 삭제했을 때
  When 사이드바 보관함 탭을 확인하면
  Then 삭제된 세션에 속해 있던 답변 북마크가 보관함 목록에서 즉시 사라져 노출되지 않는다.
- Given 북마크나 피드백이 하나도 없는 대화 세션이 존재할 때 (경계 케이스)
  When 사용자가 해당 세션을 삭제하면
  Then 오류 없이 세션이 정상 삭제되고 204 No Content가 반환된다.
- Given 존재하지 않는 세션 ID이거나 다른 사용자가 소유한 세션 ID로 삭제를 요청했을 때 (경계 케이스)
  When `DELETE /rag/sessions/{session_id}?userId={user_id}`를 호출하면
  Then 404 SessionNotFoundError가 발생하고 타인의 세션, 북마크, 피드백 도큐먼트는 일체 삭제되지 않는다.

## 참고
- `public-python-server/src/ai_service/rag/service.py:43-47` (`SessionService.delete_session`)
- `public-python-server/src/ai_service/rag/repository.py:89-99` (`ConversationSessionRepository.delete_by_id_for_user`)
- `public-python-server/src/ai_service/rag/router.py:124-135` (`delete_session`)
- `public-python-server/src/ai_service/feedback/repository.py:55-64` (`AnswerFeedbackRepository.delete`)
- `public-front/src/components/AiService.tsx:703-718` (`handleConfirmDeleteSession`)
- `public-front/src/components/AiService.tsx:1750-1758` (`bookmarks.map`)

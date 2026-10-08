---
id: SPEC-068
title: 답변 평가(피드백) 삭제(철회) 지원 및 UI 연동
status: ready
targets: [python-server, server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
이 제품은 "답변을 믿을 수 있게 만든다"를 핵심 목표로 하며, 사용자가 답변의 정확도와 유용성을 직접 평가하여 시스템에 피드백을 되돌려줄 수 있는 기능을 제공하고 있다.
그러나 현재 피드백 시스템은 등록(`POST /ai/feedback`) 및 조회(`GET /ai/feedback`)만 지원할 뿐, 등록된 평가를 삭제하거나 철회하는 기능이 구현되어 있지 않다.
- `public-python-server/src/ai_service/feedback/repository.py:10-68` (`AnswerFeedbackRepository`) 및 `public-python-server/src/ai_service/feedback/service.py:16-66` (`FeedbackService`), `public-python-server/src/ai_service/feedback/router.py:10-40` (`router`)에 피드백 삭제(`delete`) 메서드와 `DELETE /rag/feedback` 엔드포인트가 부재하다.
- `public-server/apps/gateway/src/ai/proxy/answer-feedback-proxy.controller.ts:20-46` (`AnswerFeedbackProxyController`)에 피드백 삭제를 ai-service-py로 중계하는 `DELETE` 프록시 핸들러가 없다.
- `public-front/src/api/ai.ts:392-405` (`submitAnswerFeedback`, `getSessionFeedback`)에 평가 삭제 클라이언트 API가 없고, `public-front/src/components/AnswerFeedback.tsx:72-184` (`AnswerFeedback`) 컴포넌트의 수정 폼 내에 [평가 삭제] 또는 [평가 취소] 액션이 없어 사용자가 실수로 남긴 부정확한 평가를 되돌릴 방법이 없다.
- 이로 인해 사용자가 잘못 남긴 평가 데이터가 DB에 영구히 남아 답변 품질 평가 지표가 왜곡되며, 사용자는 평가를 취소하고 싶어도 수정 외에 다른 선택지가 없는 불편을 겪는다.

## 요구사항
- [ ] `public-python-server`의 `AnswerFeedbackRepository`에 `delete(session_id, turn_index, user_id)` 메서드를 추가하고, MongoDB 컬렉션에서 조건에 일치하는 단일 평가 레코드를 삭제한다.
- [ ] `public-python-server`의 `FeedbackService`에 `delete(session_id, turn_index, user_id)` 메서드를 추가하고 세션 및 턴 소유권(`_assert_answer_is_readable`)을 검증한 후 리포지토리 삭제를 수행한다.
- [ ] `public-python-server`의 `router.py`에 `DELETE /rag/feedback` 엔드포인트를 추가하고, `sessionId`, `turnIndex`, `userId` 쿼리 파라미터를 받아 삭제를 처리하며 성공 시 HTTP 204 No Content를 반환한다.
- [ ] `public-server` 게이트웨이의 `AnswerFeedbackProxyController`에 `DELETE /ai/feedback` 엔드포인트를 추가하고, 세션 인증 유저의 `userId`(`req.session.uuid`)와 함께 ai-service-py로 삭제 요청을 프록시한다.
- [ ] `public-front/src/api/ai.ts`에 `deleteAnswerFeedback(sessionId: string, turnIndex: number): Promise<void>` 함수를 추가한다.
- [ ] `public-front/src/components/AnswerFeedback.tsx`에 `onDelete?: () => Promise<void>` prop을 지원하고, 기존 평가가 있는 상태(`existing !== undefined`)에서 수정 폼을 열었을 때 [평가 삭제] 버튼을 노출한다.
- [ ] `public-front/src/components/AnswerFeedback.tsx`에서 [평가 삭제] 버튼 클릭 시 `onDelete` 콜백을 호출하고, 성공 시 컴포넌트가 접힌 기본 상태("이 답변을 평가하기")로 초기화된다.
- [ ] `public-front/src/components/AiService.tsx`에 `handleDeleteFeedback(turnIndex)` 핸들러를 연결하여, 삭제 완료 시 상위 `feedbackByTurn` 상태에서 해당 턴의 평가를 제거하고 "답변 평가가 삭제되었습니다." 피드백을 반영한다.

## 비요구사항 (Out of scope)
- 다른 사용자가 남긴 평가 목록을 관리자 화면 등에서 일괄 삭제하거나 조회하는 기능
- 평가 삭제 이력을 별도의 감사 로그(Audit Log) 테이블이나 이벤트 스트림으로 영속화하는 작업
- 대화 세션 자체를 삭제할 때의 연쇄 삭제(Cascade Delete) 최적화 (세션 수명주기 TTL로 관리됨)

## 백엔드
- `public-python-server/src/ai_service/feedback/repository.py`: `AnswerFeedbackRepository.delete(session_id: str, turn_index: int, user_id: str) -> bool` 추가
- `public-python-server/src/ai_service/feedback/service.py`: `FeedbackService.delete(session_id: str, turn_index: int, user_id: str) -> None` 추가
- `public-python-server/src/ai_service/feedback/router.py`: `DELETE /rag/feedback` 엔드포인트 정의 (`status_code=204`, `Query` 파라미터 `sessionId`, `turnIndex`, `userId`)
- `public-server/apps/gateway/src/ai/proxy/answer-feedback-proxy.controller.ts`: `@Delete() @HttpCode(204)` 데코레이터가 적용된 `deleteMine` 메서드 추가 (`sessionId`, `turnIndex` 쿼리 및 `req.session.uuid` 전달)

## 프론트엔드
- `public-front/src/api/ai.ts`: `deleteAnswerFeedback(sessionId: string, turnIndex: number): Promise<void>` 추가 (`DELETE /ai/feedback?sessionId=...&turnIndex=...`)
- `public-front/src/components/AnswerFeedback.tsx`:
  - `Props` 인터페이스에 `onDelete?: () => Promise<void>` 선택적 prop 추가
  - `existing`이 존재하고 `isOpen === true`일 때 `feedback-actions` 영역에 [평가 삭제] 버튼 렌더링
  - 삭제 진행 중일 때 버튼 비활성화(`disabled`) 및 "삭제 중…" 피드백 표시
  - 삭제 실패 시 `setError('평가를 삭제하지 못했습니다. 잠시 후 다시 시도해 주세요.')` 안내
- `public-front/src/components/AiService.tsx`:
  - `handleDeleteFeedback = async (turnIndex: number) => { ... }` 정의 및 `AnswerFeedback`의 `onDelete`로 바인딩
  - 삭제 성공 시 `feedbackByTurn` 객체에서 해당 턴 키 제거

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 특정 답변(turnIndex: 1)에 대해 이미 평가(정확도 4, 유용성 5)를 등록한 상태에서
  When 해당 답변의 피드백 수정 폼을 열고 [평가 삭제] 버튼을 클릭하면
  Then 백엔드 `DELETE /ai/feedback`이 호출되어 저장된 평가가 삭제되고, UI가 평가 등록 전의 "이 답변을 평가하기" 버튼 상태로 복원되어야 한다.
- Given 다른 사용자의 세션 ID 또는 존재하지 않는 turnIndex로 피드백 삭제를 요청할 경우
  When 백엔드 `DELETE /ai/feedback` API가 호출되면
  Then 404 Not Found 또는 400 Bad Request 에러를 반환하고 타인의 평가 레코드는 삭제되지 않아야 한다.
- Given 평가가 등록되지 않은 답변(existing === undefined)인 경우
  When 사용자가 "이 답변을 평가하기"를 눌러 평가 등록 폼을 열면
  Then [평가 삭제] 버튼은 노출되지 않고 [취소]와 [평가 제출] 버튼만 표시되어야 한다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/feedback/repository.py:10-68` (`AnswerFeedbackRepository`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/feedback/service.py:16-68` (`FeedbackService`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/feedback/router.py:10-43` (`router`)
- 고쳐야 할 자리: `public-server/apps/gateway/src/ai/proxy/answer-feedback-proxy.controller.ts:20-47` (`AnswerFeedbackProxyController`)
- 고쳐야 할 자리: `public-front/src/api/ai.ts:392-405` (`submitAnswerFeedback`, `getSessionFeedback`)
- 고쳐야 할 자리: `public-front/src/components/AnswerFeedback.tsx:72-186` (`AnswerFeedback`)
- 관련 정의 및 컴포넌트: `public-front/src/components/AiService.tsx:309-316` (`handleSubmitFeedback`), `:1153-1160` (`AiService`)

---
id: SPEC-095
title: 답변 평가(피드백) 불만족 사유 빠른 선택 및 질문 보완 재질문 연계
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
현재 RAG 답변 시스템은 사용자가 답변을 그대로 믿지 않고 스스로 검증할 수 있도록 추론 과정, 근거 출처, 신뢰도 배지를 제공하며, 답변이 잘못되었을 때 사용자가 평가를 시스템에 되돌려 줄 수 있도록 답변 평가(피드백, SPEC-024, SPEC-051, SPEC-064, SPEC-068) 기능을 지원한다. 그러나 부정확한 답변을 마주한 사용자가 피드백을 남기고 문제를 해결하는 흐름에서 중대한 단절과 결함이 존재한다.
첫째, `public-front/src/components/AnswerFeedback.tsx:85-133` (`AnswerFeedback`)을 보면 정확도(1~5점)와 유용성(1~5점)을 선택한 뒤 의견을 남길 수 있지만, 정확도를 1점("전혀 정확하지 않다") 또는 2점("정확하지 않다")으로 낮게 평가했을 때 왜 답변이 잘못되었는지 구체적인 사유를 입력할 수단이 긴 텍스트 입력 필드(`comment`) 하나뿐이다. 사용자가 모바일이나 데스크톱 환경에서 "환각/사실 오류", "근거 문서 내용과 다름", "질문 의도와 다른 답변" 등의 사유를 매번 타이핑하기 번거로워 의견 작성을 건너뛰거나 평가 자체를 포기하게 되며, 시스템 입장에서도 어떤 유형의 답변 오류가 자주 발생하는지 정형화된 피드백을 수집하기 어렵다.
둘째, `public-front/src/components/AiService.tsx:404-413` (`handleSubmitFeedback`) 및 `public-front/src/components/AiService.tsx:1672-1679` (`AiService`)에서 사용자가 부정확한 답변에 대해 평가를 제출하면 단순히 "답변 평가가 저장되었습니다." 토스트 알림만 노출되고 대화 흐름이 그대로 종료된다. 사용자는 틀린 답변을 마주하고 피드백을 제출했음에도 불구하고 올바른 답변을 얻기 위해 질문을 어떻게 다시 작성해야 할지 안내받지 못하며, 직전 질문 내용을 다시 복원하여 조건을 보완하거나 재질문할 수 있는 액션 연계가 전혀 없다.
셋째, `public-front/src/components/AiService.tsx:1672-1679` (`AiService`)에서 피드백 컴포넌트에 넘기는 인덱스(`idx`)는 `chatLog.map((msg, idx) => ...)`의 단순 배열 인덱스다. 하지만 스트리밍 실패 인라인 에러(SPEC-073)가 발생했을 때 `public-front/src/components/AiService.tsx:811-819` (`startQuestionStreaming` 내 에러 콜백)에서 `isError: true` 메시지가 `chatLog`에 추가되는데, 이 에러 버블은 백엔드 세션의 `turns`에 영속화되지 않는다. 이로 인해 에러 버블 이후에 생성된 정상 답변의 `idx`는 실제 백엔드 세션 턴 인덱스보다 커지게 되며, 평가 제출 시 `public-python-server/src/ai_service/feedback/service.py:61-71` (`FeedbackService._assert_answer_is_readable`)에서 `turn_index >= len(turns)` 조건에 걸려 HTTP 404("대상 답변을 찾을 수 없습니다.") 에러가 발생하고 평가 저장이 영구 차단되는 결함이 발생한다.
따라서 부정확한 답변(정확도 1~2점) 평가 시 빠른 불만족 사유 선택 칩을 제공하여 피드백 수집 품질을 높이고, 평가 완료 후 직전 질문을 입력창에 복원하여 즉시 보완 재질문할 수 있는 액션을 연계하며, 에러 버블로 인한 `turnIndex` 불일치를 보정해야 한다.

## 요구사항
- [ ] `AnswerFeedback` 컴포넌트에서 정확도 점수가 1점 또는 2점(`accuracy <= 2`)으로 선택되었을 때, "어떤 점이 아쉬웠나요?" 불만족 사유 칩 버튼 그룹을 노출한다.
- [ ] 불만족 사유 칩에는 `환각/사실 오류`, `근거 문서와 다름`, `질문 의도와 무관`, `내용 불완전/부족`, `최신 정보 아님` 최소 5개 기본 선택지를 제공한다.
- [ ] 사유 칩은 다중 선택(토글)이 가능해야 하며, 사유 칩 클릭 시 선택된 사유 태그(예: `[환각/사실 오류] [근거 문서와 다름]`)가 `comment` 입력창의 접두사 텍스트로 자동 동기화되어 백엔드 계약 변경 없이 온전히 저장된다.
- [ ] 정확도가 3점 이상(`accuracy >= 3`)으로 변경되거나 초기화될 때 불만족 사유 칩 영역을 화면에서 접고 숨긴다.
- [ ] 기존 평가 수정 모달이 열렸을 때(`effectiveExisting`), 기존 `comment` 본문에 포함된 사유 태그(대괄호 형태 `[사유]`)를 파싱하여 해당 사유 칩들의 활성화 상태를 자동으로 복원한다.
- [ ] 사용자가 정확도 1~2점(`accuracy <= 2`)으로 평가를 제출 완료했을 때, 해당 AI 메시지 버블 하단 평가 영역에 `[질문 보완하여 다시 질문]` 액션 버튼(`btn-feedback-refine`)을 노출한다.
- [ ] `[질문 보완하여 다시 질문]` 버튼 클릭 시, 해당 답변의 직전 사용자 질문 텍스트를 하단 질문 입력창(`question`)에 복원하고 입력창 ref(`chatInputRef`)에 포커스를 주어 즉시 수정 후 전송할 수 있도록 지원한다.
- [ ] 답변 스트리밍 진행 중(`isStreaming === true`)일 때는 모든 `[질문 보완하여 다시 질문]` 버튼을 비활성화(`disabled`)하여 입력 충돌 및 중복 요청을 방지한다.
- [ ] `chatLog` 내에 인라인 에러 메시지(`msg.isError === true`)가 존재하더라도, 세션 영속화 대상인 실제 어시스턴트 턴 인덱스(`turnIndex`)를 올바르게 계산하여 `AnswerFeedback` 및 `handleBookmarkAnswer`에 전달함으로써 백엔드 404 오류를 방지한다.

## 비요구사항 (Out of scope)
- 백엔드 `answer_feedback` MongoDB 컬렉션 스키마나 게이트웨이 DTO의 변경(DB 마이그레이션)은 포함하지 않는다. 사유 칩은 기존 `comment` 필드의 태그 규격으로 호환 저장한다.
- RAG 질의 처리 엔진의 프롬프트나 에이전틱 루프 알고리즘 자체의 변경은 포함하지 않는다.
- 관리자 화면의 피드백 통계 대시보드 신규 구현은 포함하지 않는다.

## 프론트엔드
`public-front/src/components/AnswerFeedback.tsx`
- 불만족 사유 상수 목록 선언: `FEEDBACK_REASONS = ['환각/사실 오류', '근거 문서와 다름', '질문 의도와 무관', '내용 불완전/부족', '최신 정보 아님']`
- `selectedReasons: string[]` 상태 관리 및 정확도 점수 변경(`accuracy <= 2`) 시 사유 선택 영역 조건부 렌더링
- 사유 칩 토글 핸들러: 선택된 사유 태그들을 문자열 태그(`[사유]`)로 포맷팅하여 `comment` 텍스트 앞부분에 자동 반영
- 기존 평가 복원 시 `existing.comment` 내 대괄호 태그 파싱 및 `selectedReasons` 초기화 동기화

`public-front/src/components/AiService.tsx`
- 에러 버블(`isError: true`)을 제외하고 실제 세션 턴 인덱스를 매핑하는 유틸 함수 또는 계산 로직 적용
- `AnswerFeedback`의 `existing` 평가 결과 중 정확도가 1~2점인 경우, 메시지 버블 하단에 `btn-feedback-refine` 버튼 렌더링
- `handleRefineFeedbackQuestion` 핸들러: 해당 답변의 직전 사용자 질문 내용을 찾아 `question` 상태에 복원하고 입력창 포커스 처리
- 스트리밍 중 비활성화 상태 바인딩(`disabled={isStreaming}`)

`public-front/src/components/AiService.css`
- `.feedback-reasons-group`: 불만족 사유 칩 컨테이너 레이아웃 스타일
- `.feedback-reason-chip`: 사유 선택 칩 버튼 스타일 및 `.selected` 활성화 스타일
- `.btn-feedback-refine`: 질문 보완하여 다시 질문 버튼 스타일 및 hover/disabled 스타일

`public-front/src/components/AnswerFeedback.test.tsx` 및 `AiService.test.tsx`
- 정확도 1~2점 선택 시 불만족 사유 칩 노출 및 클릭 시 comment 동기화 단위 테스트
- 기존 평가 복원 시 comment 내 태그 파싱을 통한 사유 칩 활성화 상태 복원 단위 테스트
- 저평가(1~2점) 제출 후 `[질문 보완하여 다시 질문]` 버튼 노출 및 클릭 시 질문 입력창 복원 검증 단위 테스트
- 에러 버블이 포함된 대화 내역에서 정확한 `turnIndex`가 전달되어 피드백이 정상 제출되는지 검증하는 단위 테스트

## 수용 기준 (Acceptance Criteria)
- Given 답변 평가 폼이 열려 있는 상태에서 When 사용자가 정확도를 1점 또는 2점으로 선택하면 Then "어떤 점이 아쉬웠나요?" 안내와 함께 5개의 불만족 사유 칩(`환각/사실 오류`, `근거 문서와 다름`, `질문 의도와 무관`, `내용 불완전/부족`, `최신 정보 아님`)이 노출된다.
- Given 불만족 사유 칩이 노출된 상태에서 When 사용자가 `환각/사실 오류`와 `근거 문서와 다름` 칩 2개를 클릭하면 Then 의견(comment) 입력 필드에 `[환각/사실 오류] [근거 문서와 다름]` 접두사가 자동으로 채워지고 해당 칩들이 선택된 시각 스타일(`.selected`)을 갖는다.
- Given 불만족 사유를 선택한 뒤 추가 의견을 적은 상태에서 When 사용자가 정확도를 4점(정확하다)으로 변경하면 Then 불만족 사유 칩 영역이 화면에서 사라지고 사유 태그 입력이 초기화된다.
- Given 이전에 `[환각/사실 오류] 문서 3페이지 내용과 다릅니다.`로 평가를 저장했던 답변에서 When 사용자가 [수정] 버튼을 클릭해 폼을 열면 Then `환각/사실 오류` 칩이 선택된 상태로 렌더링되고 본문 입력창에 기존 의견이 그대로 복원된다.
- Given 정확도 1점으로 평가가 등록되어 접힌 평가 요약이 표시된 AI 답변 버블에서 When 사용자가 `[질문 보완하여 다시 질문]` 버튼을 클릭하면 Then 직전 사용자 질문 내용이 하단 질문 입력창에 복원되고 입력창에 포커스가 맞춰진다.
- Given 질문 생성 도중 에러가 발생하여 인라인 에러 버블(`isError: true`)이 생성된 후 새 질문을 전송해 정상 답변을 받은 상황(경계 케이스)에서 When 해당 정상 답변에 대해 평가를 제출하면 Then 에러 버블 인덱스가 아닌 실제 세션의 어시스턴트 턴 인덱스가 백엔드로 전달되어 404 에러 없이 정상적으로 평가가 저장된다.
- Given 답변 스트리밍이 진행 중(`isStreaming === true`)인 상황에서 When 저평가 답변 버블을 확인하면 Then `[질문 보완하여 다시 질문]` 버튼이 `disabled` 상태로 비활성화되어 중복 클릭이 방지된다.

## 참고
- 고쳐야 할 자리: `public-front/src/components/AnswerFeedback.tsx:85-133` (`AnswerFeedback`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:404-413` (`handleSubmitFeedback`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1672-1679` (`AiService`)
- 에러 버블 발생부: `public-front/src/components/AiService.tsx:811-819` (`startQuestionStreaming` 내 에러 콜백)
- 백엔드 턴 인덱스 검증부: `public-python-server/src/ai_service/feedback/service.py:61-71` (`FeedbackService._assert_answer_is_readable`)

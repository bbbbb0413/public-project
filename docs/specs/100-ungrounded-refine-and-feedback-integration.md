---
id: SPEC-100
title: 근거 미참조·저신뢰도 RAG 답변의 피드백 연계 개선 및 불만족 사유 기반 재질문 연동
status: done
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품의 핵심 목표는 지식베이스에 등록된 문서를 근거로 질문에 답하되, 답을 그대로 믿지 않고 사용자가 스스로 확인하고 검증할 수 있도록 돕는 것이다. 이를 위해 RAG 답변에 대해 근거 문서 미참조 경고 배너(SPEC-059), 확인하지 못한 항목(missing) 질문 연계(SPEC-075), 저신뢰도(신뢰도 60% 미만) 답변 경고 배너 및 질문 구체화(SPEC-093), 답변 불만족 사유 빠른 선택(SPEC-095) 등이 순차적으로 배송되었다. 그러나 근거가 부족하거나 신뢰도가 낮은 답변을 마주했을 때 사용자가 이를 시스템에 알리고 질문을 보완하는 핵심 흐름에서 여러 단절과 편의성 결함이 발생하고 있다.
첫째, `public-front/src/components/AiService.tsx:1552-1560` (`AiService`)을 보면 지식베이스에서 관련 문서를 전혀 찾지 못해 모델의 일반 지식(환각 위험이 가장 높은 상태)으로 생성된 답변(`isUngrounded`)의 경우 상단에 `[근거 문서 미발견 (모델 일반 지식 기반 답변)]` 경고 배너만 노출되고 질문을 구체화하거나 보완할 수 있는 액션 버튼이 전혀 제공되지 않는다. 바로 아래의 저신뢰도 답변(`isLowConfidence`, `public-front/src/components/AiService.tsx:1561-1588` (`AiService`))에는 `[질문 구체화]`(`btn-refine-question`) 버튼이 제공되는 것과 대조적이며, 사용자는 근거가 전혀 없는 답변에 대해 질문을 다시 보완하려면 이전 질문을 수동으로 전부 재입력해야 한다.
둘째, 근거 미발견 답변 배너(`ungrounded-warning`)와 저신뢰도 답변 배너(`low-confidence-warning`) 모두, 사용자가 답변의 부정확성을 인지했을 때 즉각 시스템에 불만족 피드백을 전달할 수 있는 바로가기 진입점이 부재하다. 사용자가 평가를 등록하려면 긴 마크다운 답변 텍스트, 메타데이터 배지, 참고 문서 목록, 보관/복사 액션 바를 지나 메시지 버블 맨 하단에 접혀 있는 `이 답변을 평가하기` 버튼(`public-front/src/components/AnswerFeedback.tsx:226-254` (`AnswerFeedback`))을 직접 스크롤하여 찾아야 하므로 피드백 수집률이 저하된다.
셋째, `public-front/src/components/AiService.tsx:918-928` (`handleRefineFeedbackQuestion`)을 보면 사용자가 1~2점 불만족 피드백을 제출하고 화면에 노출된 `[질문 보완하여 다시 질문]`(`btn-feedback-refine`, `public-front/src/components/AiService.tsx:1775-1786` (`AiService`)) 버튼을 클릭했을 때, 사용자가 정성껏 선택한 불만족 사유(예: `[근거 문서와 다름]`, `[환각/사실 오류]`)가 질문 입력창에 전혀 반영되지 않고 직전 사용자 질문 원문(`previousUserMsg?.text`)만 복원된다. 피드백 폼에서 사유 태그를 선택했음에도 불구하고 재질문 텍스트에 보완 요청 맥락이 전달되지 않아 사용자가 수동으로 요구사항을 덧붙여야 하는 불편이 크다.
따라서 근거 미발견 답변에도 질문 구체화 버튼을 제공하고, 경고 배너에서 피드백 폼을 원클릭으로 열 수 있는 진입점을 연계하며, 피드백 제출 후 재질문 시 선택한 불만족 사유 태그를 질문 텍스트에 자동으로 포함하여 질문 보완 편의성을 완성해야 한다. 이 개선은 사용자가 의심스러운 답변을 즉시 알아채고 시스템에 되돌려주며 신뢰할 수 있는 답을 다시 얻도록 돕는 제품 방향에 부합한다.

## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 근거 미발견 경고 배너(`ungrounded-warning`) 내에 `[질문 구체화]` 액션 버튼(`btn-refine-question`)을 추가한다. 클릭 시 직전 사용자 질문 내용을 하단 질문 입력창(`question`)에 복원하고 입력창 ref에 포커스한다.
- [ ] 근거 미발견 경고 배너(`ungrounded-warning`) 및 저신뢰도 경고 배너(`low-confidence-warning`) 내에 `[답변 평가]` 바로가기 버튼(`btn-warning-feedback`)을 추가한다.
- [ ] `[답변 평가]` 버튼 클릭 시 해당 AI 메시지 버블의 `AnswerFeedback` 평가 폼이 즉시 펼쳐진(열린) 상태로 전환되고 평가 영역으로 포커스를 이동한다.
- [ ] `AnswerFeedback` 컴포넌트(`public-front/src/components/AnswerFeedback.tsx`)가 외부에서 열림 상태를 트리거할 수 있도록 `isOpen` 제어 prop(예: `forceOpen?: boolean` 또는 동등한 제어 인터페이스)을 지원한다.
- [ ] `handleRefineFeedbackQuestion` 실행 시, 해당 턴의 피드백 데이터(`feedbackByTurn[turnIndex]`)에서 사유 태그(예: `[근거 문서와 다름]`, `[환각/사실 오류]`) 또는 코멘트가 존재하는 경우 이를 추출하여 `"${questionText} (보완 요청: ${extractedReasons})"` 형식으로 질문 입력창에 자동 바인딩한다.
- [ ] 피드백에 사유 태그가 없고 자유 의견 텍스트만 존재하는 경우 `"${questionText} (보완 요청: ${comment})"` 형식으로 바인딩하며, 피드백 내용이 전혀 없거나 공백인 경우에는 기존과 동일하게 `questionText`만 복원한다.
- [ ] 스트리밍 진행 중(`isStreaming === true`)일 때는 근거 미발견 배너 및 저신뢰도 배너의 `[질문 구체화]` 및 `[답변 평가]` 버튼을 모두 비활성화(`disabled`)하여 입력 충돌 및 중복 요청을 방지한다.
- [ ] 세션 복원(`handleLoadSession`)으로 불러온 이전 대화 세션의 AI 메시지 버블에서도 근거 미발견 배너의 질문 구체화, 경고 배너의 답변 평가 바로가기, 불만족 사유 태그가 연동된 재질문 동작이 동일하게 정상 동작해야 한다.
- [ ] 변경된 UI 렌더링, 경고 배너 피드백 폼 연동, 피드백 사유 태그 포함 질문 복원 동작을 검증하는 프론트엔드 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 백엔드 피드백 데이터베이스 스키마나 REST API 규격(`POST/GET/DELETE /rag/feedback`)을 변경하지 않는다.
- RAG 질의 비평(Critique) 프롬프트 내용이나 신뢰도 판정 수식(0.6 임계치)을 수정하지 않는다.
- 실시간 채팅(`ChatRoom.tsx`)이나 관리자 화면 컴포넌트의 동작을 수정하지 않는다.
- 지식베이스 청크 열람 모달이나 원본 파일 다운로드 처리 방식을 변경하지 않는다.

## 프론트엔드
`public-front/src/components/AnswerFeedback.tsx`
- 외부 열림 제어 prop 추가:
  - `forceOpen?: boolean` 선택적 prop을 추가한다.
  - `forceOpen`이 `true`로 전달되면 컴포넌트 내부 `isOpen` 상태를 `true`로 설정하고 폼 초기화(`resetForm()`)를 수행한다.
- 기존 접힘/펼침 토글 및 평가 제출/삭제/취소 동작은 기존 규칙을 온전히 유지한다.

`public-front/src/components/AiService.tsx`
- 개별 메시지별 피드백 폼 열림 상태 관리:
  - `const [activeFeedbackTurns, setActiveFeedbackTurns] = useState<Record<number, boolean>>({});` 상태를 추가한다.
  - 경고 배너 내 `[답변 평가]` 클릭 핸들러(`handleOpenFeedbackFromWarning(turnIndex: number)`)를 구현하여 `activeFeedbackTurns[turnIndex] = true`로 갱신한다.
- 근거 미발견 배너(`ungrounded-warning`, 라인 1552~1560) 개선:
  - 배너 우측 액션 영역에 `[질문 구체화]`(`btn-refine-question`) 및 `[답변 평가]`(`btn-warning-feedback`) 버튼을 렌더링한다.
  - `[질문 구체화]` 클릭 시 직전 사용자 질문을 복원하고 포커스한다.
  - `[답변 평가]` 클릭 시 `handleOpenFeedbackFromWarning(turnIndex)`를 호출한다.
- 저신뢰도 배너(`low-confidence-warning`, 라인 1561~1588) 개선:
  - 기존 `[질문 구체화]` 버튼 옆에 `[답변 평가]`(`btn-warning-feedback`) 버튼을 추가 렌더링한다.
  - `[답변 평가]` 클릭 시 `handleOpenFeedbackFromWarning(turnIndex)`를 호출한다.
- `handleRefineFeedbackQuestion` 핸들러 개선:
  - `msgIndex`와 `turnIndex`를 전달받는다.
  - 직전 사용자 질문 원문(`previousUserMsg?.text || ''`)을 획득한다.
  - `feedbackByTurn[turnIndex]`에 저장된 평가가 있고 `comment` 필드가 존재하는 경우, 코멘트 내 사유 태그(예: `[근거 문서와 다름]`) 또는 정리된 코멘트를 추출한다.
  - 사유가 존재하면 `"${questionText} (보완 요청: ${reasons})"` 형식으로 포맷팅하여 `setQuestion`에 전달하고 질문 입력창 ref에 포커스한다.
- `AnswerFeedback` 바인딩부 개선 (라인 1768~1774):
  - `forceOpen={Boolean(activeFeedbackTurns[turnIndex])}` prop을 전달하여 배너 바로가기 클릭 시 즉시 폼이 열리도록 연동한다.
- `btn-feedback-refine` 호출부(라인 1781):
  - `handleRefineFeedbackQuestion(idx, turnIndex)`로 `turnIndex`를 함께 넘기도록 수정한다.

`public-front/src/components/AiService.css`
- `.ungrounded-warning`: 배너 내부 레이아웃을 flex 컨테이너로 정돈하여 안내 문구와 우측 액션 버튼 그룹(`ungrounded-warning-actions`)을 배치한다.
- `.btn-warning-feedback`: 경고 배너 내부의 보조 액션 버튼 스타일을 정의하여 `btn-refine-question`과 시각적 통일성을 부여한다.
- `.low-confidence-warning-actions`: 저신뢰도 배너 액션 버튼 그룹 스타일을 보강한다.

`public-front/src/components/AiService.test.tsx`
- 근거 미발견(`isUngrounded === true`) AI 답변 버블에 `[질문 구체화]` 버튼이 렌더링되고, 클릭 시 직전 사용자 질문이 입력창에 복원되는지 검증하는 단위 테스트
- 근거 미발견 배너 및 저신뢰도 배너의 `[답변 평가]` 버튼 클릭 시 해당 답변의 `AnswerFeedback` 폼이 즉시 열리는지 검증하는 단위 테스트
- 불만족 피드백에 사유 태그(예: `[환각/사실 오류] [근거 문서와 다름]`)가 저장된 상태에서 `[질문 보완하여 다시 질문]` 클릭 시 `"${질문} (보완 요청: [환각/사실 오류] [근거 문서와 다름])"` 형식으로 질문 입력창에 복원되는지 검증하는 단위 테스트
- 스트리밍 진행 중(`isStreaming === true`)일 때 경고 배너 내 버튼 및 피드백 재질문 버튼이 비활성화(`disabled`)되는지 검증하는 단위 테스트
- 세션 복원 대화에서 근거 미발견 및 저신뢰도 답변의 액션 버튼들이 동일하게 정상 동작함을 검증하는 단위 테스트

## 수용 기준 (Acceptance Criteria)
- Given 지식베이스에서 출처를 찾지 못해 `isUngrounded`로 판정된 AI 답변 버블이 렌더링되었을 때 When 상단 경고 배너를 확인하면 Then "근거 문서 미발견 (모델 일반 지식 기반 답변)" 문구와 함께 `[질문 구체화]` 및 `[답변 평가]` 버튼이 모두 표시된다.
- Given 근거 미발견 답변 배너의 `[질문 구체화]` 버튼이 표시된 상태에서 When 사용자가 해당 버튼을 클릭하면 Then 직전 사용자 질문 텍스트가 하단 질문 입력창(`question`)에 복원되고 입력창에 포커스가 맞춰진다.
- Given 근거 미참조 또는 저신뢰도 AI 답변 버블이 표시되어 있을 때 When 사용자가 경고 배너의 `[답변 평가]` 버튼을 클릭하면 Then 해당 답변 버블 하단의 `AnswerFeedback` 평가 입력 폼이 즉시 펼쳐진 상태로 화면에 렌더링된다.
- Given 사용자가 특정 답변에 대해 정확도 1점과 불만족 사유 `[근거 문서와 다름]`을 선택하여 평가를 제출한 상태에서 When 해당 답변 하단의 `[질문 보완하여 다시 질문]` 버튼을 클릭하면 Then 하단 질문 입력창에 `"${직전질문} (보완 요청: [근거 문서와 다름])"` 형식으로 보완 요청 사유가 포함되어 복원되고 포커스된다.
- Given 피드백에 사유 태그 없이 자유 의견("수치 정보가 맞지 않습니다")만 입력되어 저장된 상태에서 When 사용자가 `[질문 보완하여 다시 질문]`을 클릭하면 Then 하단 질문 입력창에 `"${직전질문} (보완 요청: 수치 정보가 맞지 않습니다)"` 형식으로 복원된다.
- Given 피드백에 코멘트가 전혀 없거나 공백만 저장된 상태에서 When 사용자가 `[질문 보완하여 다시 질문]`을 클릭하면 Then 질문 입력창에 괄호 없이 기존 직전 질문 원문만 온전히 복원된다.
- Given 질문 스트리밍이 진행 중인(`isStreaming === true`) 상황에서 When 이전 답변 버블의 근거 미참조 배너, 저신뢰도 배너 및 피드백 재질문 영역을 확인하면 Then `[질문 구체화]`, `[답변 평가]`, `[질문 보완하여 다시 질문]` 버튼이 모두 비활성화(`disabled`) 상태여야 한다.

## 참고
- 고쳐야 할 자리 (근거 미발견 경고 배너 렌더링부): `public-front/src/components/AiService.tsx:1552-1560` (`AiService`)
- 고쳐야 할 자리 (저신뢰도 경고 배너 렌더링부): `public-front/src/components/AiService.tsx:1561-1588` (`AiService`)
- 고쳐야 할 자리 (피드백 기반 재질문 핸들러): `public-front/src/components/AiService.tsx:918-928` (`handleRefineFeedbackQuestion`)
- 피드백 재질문 버튼 렌더링 및 AnswerFeedback 연동부: `public-front/src/components/AiService.tsx:1768-1786` (`AiService`)
- 답변 평가 컴포넌트 접힘/펼침 제어부: `public-front/src/components/AnswerFeedback.tsx:226-254` (`AnswerFeedback`)
- 확인하지 못한 항목(missing) 질문 연계 참조 구현: `public-front/src/components/AiService.tsx:930-933` (`handleAskMissingItem`)

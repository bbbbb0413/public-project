---
id: SPEC-107
title: 답변 평가 불만족 사유 태그 요약 표시 및 부정확 답변 경고 상태 시각화 연동
status: done
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품의 핵심 목표는 지식베이스에 등록된 문서를 근거로 질문에 답하되, 사용자가 답변을 그대로 믿지 않고 스스로 확인하고 검증할 수 있도록 돕는 도구다. 이를 위해 답변 완료 시 신뢰도 전달(SPEC-010), 단발성 질의 신뢰도 산출 및 근거 미참조 경고(SPEC-059), 저신뢰도 답변 주의 경고(SPEC-093), 답변 불만족 사유 빠른 선택(SPEC-095), 근거 미참조·저신뢰도 답변의 피드백 연계 개선(SPEC-100) 등이 순차적으로 배송되었다.
그러나 사용자가 틀린 답변을 발견하고 시스템에 부정적인 평가를 전달한 이후, 그 결과가 화면에 유지되고 반영되는 핵심 피드백 루프에서 다음과 같은 심각한 시각적 단절과 상태 누락이 존재한다:
첫째, `public-front/src/components/AnswerFeedback.tsx:15-28` (`parseReasonsFromComment`)에는 사용자가 작성한 코멘트에서 불만족 사유 태그(예: `[환각/사실 오류]`, `[근거 문서와 다름]`)를 추출하는 파싱 로직이 이미 구현되어 있다. 하지만 `public-front/src/components/AnswerFeedback.tsx:260-275` (`AnswerFeedback`)의 접힌 평가 요약 영역(`feedback-summary`)을 보면, 오직 `내 평가 · 정확도 {accuracy}/5 · 유용성 {helpfulness}/5`라는 숫자 텍스트만 렌더링된다. 사용자가 정성껏 선택한 구체적인 불만족 사유 태그들이 접힌 화면에서는 완전히 은닉되어, 사용자가 자신이 이 답변에 대해 어떤 문제점(환각인지, 근거 불일치인지 등)을 지적했는지 다시 확인하려면 매번 [수정] 버튼을 눌러 모달 폼을 다시 열어야만 한다.
둘째, `public-front/src/components/AiService.tsx:1655-1690` (`AiService` 내 메시지 버블 렌더링부)을 보면 시스템이 자체 산출한 `hasMissing`(`[추가 확인 필요]`), `isLowConfidence`(`[답변 신뢰도 낮음]`), `isUngrounded`(`[근거 문서 미발견]`) 배지는 렌더링되지만, 사용자가 직접 평가를 통해 "정확하지 않다(정확도 1~2점)"고 시스템에 피드백한 상태(`feedbackByTurn[turnIndex]?.accuracy <= 2`)를 반영하는 시각적 배지나 스타일이 전무하다. 이로 인해 사용자가 오류를 발견하고 평가를 완료했음에도 불구하고, 해당 답변 버블은 여전히 아무런 문제가 없는 일반 정상 답변과 완전히 동일한 외형으로 대화창에 방치된다.
셋째, `public-front/src/components/AiService.tsx:2055-2076` (`AiService` 내 AI 메시지 액션 및 피드백 바인딩부)에서 정확도 1~2점 불만족 피드백이 등록되면 `[질문 보완하여 다시 질문]`(`btn-feedback-refine`) 버튼이 화면에 나타나지만, 메시지 버블 상단에는 아무런 경고 표시가 없어 스크롤을 내리거나 이전 대화를 다시 열었을 때 해당 답변이 검증에 실패한 답변인지 한눈에 인지하기 어렵다. 또한 평가를 삭제하거나 점수를 3점 이상으로 수정했을 때 답변 버블의 시각적 경고 상태가 안전하게 초기화되어야 한다.
넷째, `public-front/src/api/ai.ts:383-391` (`AnswerFeedbackOut`) 인터페이스와 `public-front/src/components/AiService.tsx:480-487` (`loadFeedback`)에서 세션 복원 시 각 턴의 피드백 데이터(`turnIndex`, `accuracy`, `helpfulness`, `comment`)가 이미 클라이언트에 정상적으로 동기화되고 있으므로, 추가적인 백엔드 스키마 변경 없이 프론트엔드 단독으로 평가 사유 요약 노출 및 답변 버블 경고 배지 시각화를 구현할 수 있다.
따라서 접힌 평가 요약 영역에 선택된 불만족 사유 태그들을 배지 형태로 함께 노출하고, 1~2점 불만족 평가를 받은 답변 버블에 `[사용자 평가: 부정확]` 시각적 경고 배지와 전용 스타일을 적용하여 사용자가 틀린 답변을 즉시 알아채고 신뢰성 검증 상태를 대화 전체에서 유지할 수 있도록 개선해야 한다. 이 개선은 "답이 틀렸을 때 사용자가 그것을 알아챌 수 있는가", "사용자가 틀렸다고 판단한 답을 시스템에 되돌려 줄 수단이 있는가"라는 이번 분기 제품 방향에 직결된다.
## 요구사항
- [ ] `public-front/src/components/AnswerFeedback.tsx`의 접힌 평가 요약 영역(`!isOpen`)에서 `effectiveExisting`이 존재할 때 `parseReasonsFromComment(effectiveExisting.comment ?? '')`로 추출된 불만족 사유 목록(`reasons`)이 1개 이상 존재하면, `feedback-summary` 내부 또는 우측에 사유 태그 목록 컨테이너(`feedback-summary-reasons`)를 렌더링한다.
- [ ] 각 사유 태그는 `feedback-summary-reason-tag` CSS 클래스를 부여하고 `[${reason}]` (또는 `${reason}`) 텍스트 배지 형태로 표시하여, 사용자가 모달 폼을 펼치지 않고도 접힌 상태에서 어떤 불만족 사유로 평가했는지 즉시 확인할 수 있도록 한다.
- [ ] 추출된 사유 태그가 없거나 코멘트가 빈 문자열/공백인 평가 건(`reasons.length === 0`)에 대해서는 기존과 동일하게 점수 텍스트(`내 평가 · 정확도 N/5 · 유용성 N/5`)만 깔끔하게 노출한다.
- [ ] `public-front/src/components/AiService.tsx`의 메시지 버블 렌더링부(`chatLog.map`)에서 해당 턴의 피드백(`feedbackByTurn[turnIndex]`)이 존재하고 정확도가 1점 또는 2점(`accuracy <= 2`)인 경우, 메시지 버블(`message-bubble`)에 `has-negative-feedback` CSS 클래스를 적용한다.
- [ ] 정확도 1~2점 피드백이 등록된 AI 메시지 버블 상단(메타 배지 영역)에 `[사용자 평가: 부정확]` 경고 배지(`negative-feedback-badge`, `data-testid="negative-feedback-badge"`)를 렌더링하여 이미 오류가 지적된 답변임을 한눈에 식별할 수 있도록 한다.
- [ ] 사용자가 피드백 폼에서 [평가 삭제]를 실행(`handleDeleteFeedback`)하거나 정확도를 3점 이상으로 수정 제출(`handleSubmitFeedback`)한 경우, `has-negative-feedback` 클래스 및 `negative-feedback-badge`가 즉시 화면에서 제거되고 일반 상태 스타일로 정상 복귀되어야 한다.
- [ ] 세션 복원(`handleLoadSession`)으로 불러온 대화 세션에서도 저장된 피드백 데이터(`loadFeedback`)에 따라 정확도 1~2점 답변 버블에 `has-negative-feedback` 클래스와 `[사용자 평가: 부정확]` 배지가 온전히 복원되어 렌더링되어야 한다.
- [ ] 스트리밍 진행 중(`isStreaming === true`)인 최신 답변 버블이나 오류 메시지 버블(`msg.isError`), 중단된 메시지 버블(`msg.isCancelled`)에는 부정확 평가 배지가 잘못 표시되지 않도록 격리한다.
- [ ] `public-front/src/components/AiService.css`에 접힌 요약 사유 태그 컨테이너(`.feedback-summary-reasons`), 개별 사유 태그(`.feedback-summary-reason-tag`), 부정확 평가 답변 버블 스타일(`.message-bubble.has-negative-feedback`), 부정확 배지 스타일(`.negative-feedback-badge`)을 추가한다.
- [ ] 변경된 접힌 요약 내 사유 태그 표시, 정확도 1~2점 답변 버블의 부정확 배지 렌더링, 평가 삭제/수정 시 상태 갱신 동작을 검증하는 프론트엔드 단위 테스트를 작성한다.
## 비요구사항 (Out of scope)
- 백엔드 피드백 데이터베이스 스키마나 REST API 규격(`POST/GET/DELETE /ai/feedback`, `POST/GET/DELETE /rag/feedback`)을 변경하지 않는다.
- `AnswerFeedback.tsx`의 내부 평가 입력 폼(별점 라디오, 불만족 사유 선택 칩, 글자 수 제한 등)의 유효성 검증 규칙을 변경하지 않는다.
- 실시간 채팅(`ChatRoom.tsx`)이나 결제 내역(`PaymentHistory.tsx`), 관리자 화면의 동작을 수정하지 않는다.
- RAG 질의 비평(Critique) 프롬프트 템플릿이나 신뢰도 산출 수식을 변경하지 않는다.
## 프론트엔드
`public-front/src/components/AnswerFeedback.tsx`
- 접힌 평가 요약부(`!isOpen`, 라인 260~275):
  - `effectiveExisting`이 존재할 때 `const reasons = parseReasonsFromComment(effectiveExisting.comment ?? '');`를 추출한다.
  - `btn-feedback-toggle.rated` 내부의 `feedback-summary` 영역 또는 우측에 `reasons.length > 0`인 경우:
    ```tsx
    {reasons.length > 0 && (
      <div className="feedback-summary-reasons" data-testid="feedback-summary-reasons">
        {reasons.map((reason) => (
          <span key={reason} className="feedback-summary-reason-tag">
            {reason}
          </span>
        ))}
      </div>
    )}
    ```
    를 추가 렌더링한다.
- 불만족 사유 태그 클릭 시 모달이 정상적으로 열리도록 이벤트 버블링을 유지한다.

`public-front/src/components/AiService.tsx`
- 메시지 버블 렌더링 판정부 (`chatLog.map`, 라인 1655~1675):
  - 각 턴의 피드백 상태를 조회한다:
    `const currentFeedback = feedbackByTurn[turnIndex];`
    `const hasNegativeFeedback = isAi && currentFeedback?.accuracy !== undefined && currentFeedback.accuracy !== null && currentFeedback.accuracy <= 2;`
  - 메시지 버블 컨테이너 클래스에 반영한다:
    `className={`message-bubble${hasMissing ? ' has-missing' : ''}${isLowConfidence ? ' has-low-confidence' : ''}${hasNegativeFeedback ? ' has-negative-feedback' : ''}${turnIndex === highlightedTurnIndex ? ' highlighted-turn' : ''}`}`
  - 상단 경고 배지 영역(라인 1680~1690 부근)에 부정확 평가 배지를 조건부 렌더링한다:
    ```tsx
    {hasNegativeFeedback && (
      <div className="negative-feedback-badge" data-testid="negative-feedback-badge">
        [사용자 평가: 부정확]
      </div>
    )}
    ```
- 평가 수정 및 삭제 시 상태 동기화:
  - `handleSubmitFeedback` 및 `handleDeleteFeedback` 실행 시 `feedbackByTurn` 상태가 즉시 갱신되므로, 해당 턴의 `hasNegativeFeedback` 및 배지가 리렌더링을 통해 즉각 반응한다.

`public-front/src/components/AiService.css`
- `.feedback-summary-reasons`: 접힌 피드백 요약 내부에서 사유 태그들을 가로로 나열하는 flex 컨테이너 (인라인 flex, flex-wrap: wrap, gap: 4px, margin-left: 6px).
- `.feedback-summary-reason-tag`: 접힌 요약에 표시되는 개별 사유 태그 스타일 (11px 폰트 크기, 패딩 1px 6px, 연한 주황/적색 배경 `#fef2f2`, 테두리 `#fca5a5`, 텍스트 `#b91c1c`, 둥근 테두리 10px).
- `.message-bubble.has-negative-feedback`: 사용자가 부정확하다고 평가한 답변 버블 스타일 (적색 계열 테두리 `1px solid rgba(239, 68, 68, 0.4)`, 은은한 붉은 틴트 배경 또는 하이라이트).
- `.negative-feedback-badge`: `[사용자 평가: 부정확]` 시각적 배지 스타일 (배경 `#fee2e2`, 텍스트 `#dc2626`, 폰트 크기 11px, 폰트 굵기 600, 둥근 모서리 4px, 패딩 2px 6px, margin-bottom 6px).

`public-front/src/components/AnswerFeedback.test.tsx` 및 `public-front/src/components/AiService.test.tsx`
- 불만족 사유 태그(예: `[환각/사실 오류] [근거 문서와 다름]`)가 포함된 피드백 데이터가 주어졌을 때, 접힌 요약 영역에 `feedback-summary-reason-tag` 배지가 올바르게 렌더링되는지 검증하는 단위 테스트.
- 사유 태그가 없는 피드백 데이터(일반 코멘트 또는 빈 코멘트)가 주어졌을 때(경계 케이스), 사유 태그 영역 없이 기본 점수 요약만 정상 렌더링되는지 검증하는 단위 테스트.
- 정확도 1점 또는 2점 피드백이 등록된 답변 버블에 `has-negative-feedback` 클래스와 `negative-feedback-badge`가 렌더링되는지 검증하는 단위 테스트.
- 피드백 삭제(`handleDeleteFeedback`) 또는 3점 이상으로 수정 시 `has-negative-feedback` 클래스와 경고 배지가 즉시 제거되는지 검증하는 단위 테스트.
- 세션 복원(`handleLoadSession`) 시 저장된 피드백 데이터에 따라 해당 답변 버블에 부정확 경고 배지가 정상 복원되는지 검증하는 단위 테스트.
## 수용 기준 (Acceptance Criteria)
- Given 불만족 사유 태그(예: `[환각/사실 오류] [근거 문서와 다름]`)가 포함된 피드백(`existing = { accuracy: 1, helpfulness: 1, comment: '[환각/사실 오류] [근거 문서와 다름] 사실과 다릅니다.' }`)이 존재할 때,
  When `AnswerFeedback` 컴포넌트가 접힌 상태(`!isOpen`)로 렌더링되면,
  Then 접힌 요약 영역에 `내 평가 · 정확도 1/5 · 유용성 1/5` 점수 텍스트와 함께 `환각/사실 오류`, `근거 문서와 다름` 사유 태그 배지(`feedback-summary-reason-tag`)가 나열되어 표시되어야 한다.
- Given 사유 태그가 포함되지 않은 일반 의견만 있는 피드백(`existing = { accuracy: 2, helpfulness: 2, comment: '내용이 조금 부족해요' }`)이거나 코멘트가 없는 피드백(`comment: null` 또는 빈 문자열)이 주어졌을 때 (경계 케이스),
  When `AnswerFeedback` 컴포넌트가 접힌 상태로 렌더링되면,
  Then 사유 태그 배지 영역(`feedback-summary-reasons`)은 렌더링되지 않고 기본 점수 요약(`내 평가 · 정확도 2/5 · 유용성 2/5`)만 안전하게 표시되어야 한다.
- Given 대화창에 세션이 활성화되어 있고 AI 답변 버블에 대해 사용자가 정확도 1점 또는 2점으로 평가를 제출 완료했을 때,
  When 피드백 저장 API가 성공적으로 완료되면,
  Then 해당 AI 메시지 버블에 `has-negative-feedback` CSS 클래스가 즉시 부여되고 상단에 `[사용자 평가: 부정확]` 배지(`negative-feedback-badge`)가 렌더링되어야 한다.
- Given 정확도 1~2점으로 부정확 배지가 노출된 AI 답변 버블에서 사용자가 평가 폼을 다시 열어 [평가 삭제] 버튼을 클릭했을 때,
  When 삭제 API 호출이 완료되면,
  Then 해당 답변 버블에서 `has-negative-feedback` 클래스와 `[사용자 평가: 부정확]` 배지가 즉시 사라지고 일반 답변 버블 외형으로 복귀해야 한다.
- Given 사용자가 사이드바 세션 목록에서 이전에 1~2점 불만족 평가를 남겼던 세션을 클릭하여 복원했을 때(`handleLoadSession`),
  When 세션 상세 턴 및 피드백 데이터(`loadFeedback`) 조회가 완료되면,
  Then 평가가 등록되어 있던 해당 답변 턴 메시지 버블에 `has-negative-feedback` 스타일과 `[사용자 평가: 부정확]` 배지, 그리고 접힌 피드백 요약 내 사유 태그가 올바르게 렌더링되어야 한다.
## 참고
- 고쳐야 할 자리: `public-front/src/components/AnswerFeedback.tsx:15-28` (`parseReasonsFromComment`)
- 고쳐야 할 자리: `public-front/src/components/AnswerFeedback.tsx:260-275` (`AnswerFeedback`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1655-1690` (`AiService` 내 메시지 버블 렌더링부)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:2055-2076` (`AiService` 내 AI 메시지 액션 및 피드백 렌더링부)
- 관련 피드백 인터페이스: `public-front/src/api/ai.ts:383-391` (`AnswerFeedbackOut`)
- 관련 게이트웨이 프록시 컨트롤러: `public-server/apps/gateway/src/ai/proxy/answer-feedback-proxy.controller.ts:16-50` (`AnswerFeedbackProxyController`)

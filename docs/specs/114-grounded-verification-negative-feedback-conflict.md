---
id: SPEC-114
title: 사용자 부정확 평가 시 답변 근거 검증 완료 배지 오표시 방지 및 상호 배타성 보장
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
이 제품의 핵심 목표는 지식베이스 문서를 근거로 질문에 답하되, 사용자가 답변을 그대로 믿지 않고 스스로 검증할 수 있도록 돕는 도구다. 이를 위해 답변 평가 및 불만족 사유 태그 요약(SPEC-095, SPEC-107), AI 답변 근거 충분 검증 배지(SPEC-108), 불만족 피드백 기반 재질문 편의성 개선(SPEC-110) 등이 지속해서 배송되었다.
그러나 "사용자가 틀렸다고 판단한 답을 시스템에 되돌려 줄 수 있는가" 및 "답변을 믿을 수 있게 만드는가"라는 제품 방향과 관련하여, 이미 배송된 검증 배지와 사용자 피드백 사이에 다음과 같은 심각한 상태 모순과 화면 결함이 존재한다:
첫째, `public-front/src/components/AiService.tsx:1844-1852` (`AiService` 내 `isGroundedVerified` 판정부)를 보면 신뢰도가 80% 이상(`confidence >= 0.8`)이고 출처가 존재하며 결여 항목이 없으면 `isGroundedVerified`를 무조건 참으로 판정한다. 반면 `public-front/src/components/AiService.tsx:1853-1860` (`AiService` 내 `hasNegativeFeedback` 판정부)에서는 사용자가 해당 턴에 대해 정확도 1점 또는 2점으로 평가를 등록한 경우 `hasNegativeFeedback`을 참으로 판정한다. 문제는 `isGroundedVerified` 판정 조건에 `!hasNegativeFeedback`이 누락되어 있어, 사용자가 사실과 다르거나 환각이 발생했다고 지적한 답변임에도 불구하고 시스템의 `isGroundedVerified`가 여전히 참으로 유지된다.
둘째, `public-front/src/components/AiService.tsx:1869-1891` (`AiService` 내 메시지 버블 클래스 및 배지 렌더링부)를 보면, `isGroundedVerified`와 `hasNegativeFeedback`이 모두 참일 때 상단에 `✓ 지식베이스 근거 검증 완료` 배지와 `[사용자 평가: 부정확]` 배지가 한 버블 안에 나란히 동시에 렌더링된다. 또한 버블 컨테이너에도 `is-grounded-verified`와 `has-negative-feedback` 클래스가 함께 부여된다. `public-front/src/components/AiService.css:1640-1664` (`AiService.css` 내 배지 및 버블 스타일 정의부)에 따르면 검증 완료는 녹색 테두리(`rgba(74, 222, 128, 0.35)`), 부정확 평가는 빨간색 테두리(`rgba(239, 68, 68, 0.4)`) 및 붉은 배경을 적용하도록 되어 있어, 시각적 스타일 충돌뿐만 아니라 시스템은 "검증 완료"를 주장하고 사용자는 "부정확"을 지적하는 극단적인 모순이 노출된다.
셋째, `public-front/src/components/AiService.tsx:2001-2077` (`AiService` 내 인라인 주의 배너 렌더링부)를 보면 근거 문서가 없거나(`isUngrounded`) 저신뢰도인 경우(`isLowConfidence`)에는 상단에 경고 배너와 [질문 구체화] 액션이 제공되지만, 사용자가 직접 부정확하다고 명시한 답변(`hasNegativeFeedback`)에 대해서는 상단에 안내 배너가 전혀 노출되지 않는다. `public-front/src/components/AiService.tsx:2286-2299` (`AiService` 내 피드백 재질문 버튼부)에 배치된 [질문 보완하여 다시 질문] 버튼은 긴 답변 본문과 참고 문서, 피드백 컴포넌트 하단에 위치하여, 사용자가 상단에서 오류를 인지한 뒤 즉시 재질문으로 이어가기 어렵다.
이는 SPEC-107(부정확 답변 경고 시각화)과 SPEC-108(근거 충분 검증 배지)이 서로 배타적으로 조화되지 못하여 발생한 결함이며, 시스템이 보증한 답변이라도 사용자가 오류를 발견하면 시스템 보증 배지를 억제하고 부정확 주의 상태로 전환되어야 한다는 신뢰 원칙을 훼손한다.
따라서 사용자 부정확 피드백이 존재하는 답변에서는 `isGroundedVerified` 배지를 즉시 억제하고, 상단에 부정확 안내 배너와 상단 질문 보완 액션을 제공하여 답변 신뢰 체계를 정상화해야 한다.

## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 `isGroundedVerified` 판정 조건에 `!hasNegativeFeedback`을 추가하여, 해당 답변 턴에 부정확 피드백(정확도 1~2점)이 등록된 경우 `isGroundedVerified`가 거짓이 되도록 상호 배타성을 보장한다.
- [ ] `public-front/src/components/AiService.tsx`의 메시지 버블 클래스에서 `hasNegativeFeedback === true`인 경우 `is-grounded-verified` 클래스가 적용되지 않고 `has-negative-feedback` 클래스만 단독 적용되어 스타일 충돌을 방지한다.
- [ ] 사용자가 답변 평가를 3점 이상으로 수정하거나 [평가 삭제]를 통해 부정확 피드백을 철회한 경우, 원래의 고신뢰도 조건을 만족한다면 `isGroundedVerified` 상태 및 `[✓ 지식베이스 근거 검증 완료]` 배지가 즉시 화면에 복원된다.
- [ ] `public-front/src/components/AiService.tsx`의 인라인 배너 렌더링 영역에 `hasNegativeFeedback === true`일 때 노출되는 부정확 답변 주의 안내 배너(`negative-feedback-warning`, `data-testid="negative-feedback-warning"`)를 추가한다.
- [ ] `negative-feedback-warning` 배너 내에 `AnswerFeedback`의 코멘트에서 추출된 불만족 사유 태그(`parseReasonsFromComment`)가 있는 경우 해당 사유(예: "사유: 환각/사실 오류, 근거 문서와 다름")를 포함하고, 사유 태그가 없는 경우 "사용자가 부정확한 답변으로 평가한 내용입니다. 답변 내용을 확인하고 질문을 보완해 보세요." 기본 안내 문구를 렌더링한다.
- [ ] `negative-feedback-warning` 배너 내에 [질문 보완] 액션 버튼(`btn-refine-negative-question`, `data-testid="btn-refine-negative-question"`)을 제공하여, 클릭 시 하단까지 스크롤할 필요 없이 `handleRefineFeedbackQuestion`을 호출해 질문 입력창에 피드백 기반 보완 질문을 즉시 바인딩하도록 한다.
- [ ] `public-front/src/components/AiService.css`에 `negative-feedback-warning` 배너 및 배너 내부 액션 버튼 스타일을 추가한다.
- [ ] 피드백 등록/수정/삭제 시 `isGroundedVerified` 배지와 `hasNegativeFeedback` 배지의 상호 배타적 렌더링 및 상단 부정확 안내 배너 노출 동작을 검증하는 프론트엔드 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 백엔드 피드백 API(`POST /ai/feedback`, `DELETE /ai/feedback`, `GET /ai/feedback/session/:sessionId`)의 계약이나 DB 스키마를 변경하지 않는다.
- `AnswerFeedback.tsx` 컴포넌트의 5단계 별점 입력 방식이나 불만족 사유 칩(`FEEDBACK_REASONS`) 목록을 변경하지 않는다.
- 출처 청크 열람 모달(`chunks-modal`)이나 지식베이스 문서 관리 테이블의 UI/동작을 변경하지 않는다.
- 실시간 채팅(`ChatRoom.tsx`)이나 결제 내역(`PaymentHistory.tsx`) 컴포넌트를 수정하지 않는다.

## 프론트엔드
`public-front/src/components/AiService.tsx`
- `isGroundedVerified` 판정식 정정:
  ```tsx
  const currentFeedback = feedbackByTurn[turnIndex];
  const hasNegativeFeedback =
    isAi &&
    !msg.isError &&
    !msg.isCancelled &&
    currentFeedback?.accuracy !== undefined &&
    currentFeedback.accuracy !== null &&
    currentFeedback.accuracy <= 2;
  const isGroundedVerified =
    isAi &&
    !isNoSources &&
    !isUngrounded &&
    !hasMissing &&
    !msg.isCancelled &&
    !msg.isError &&
    !hasNegativeFeedback &&
    msg.confidence !== undefined &&
    msg.confidence >= 0.8;
  ```
- 인라인 주의 배너 렌더링부 (`isUngrounded`, `isLowConfidence` 분기 체인):
  - `hasNegativeFeedback`인 경우 `negative-feedback-warning` 배너를 우선 또는 독립적으로 렌더링한다.
  - 피드백 코멘트 파싱:
    ```tsx
    const feedbackReasons = currentFeedback?.comment
      ? parseReasonsFromComment(currentFeedback.comment)
      : [];
    ```
  - 배너 렌더링 구조:
    ```tsx
    {hasNegativeFeedback ? (
      <div className="negative-feedback-warning" data-testid="negative-feedback-warning">
        <div className="negative-feedback-warning-content">
          <span className="negative-feedback-warning-icon" aria-hidden="true">⚠</span>
          <span className="negative-feedback-warning-text">
            {feedbackReasons.length > 0
              ? `사용자가 부정확한 답변으로 평가했습니다. (사유: ${feedbackReasons.join(', ')})`
              : '사용자가 부정확한 답변으로 평가한 내용입니다. 답변 내용을 확인하고 질문을 보완해 보세요.'}
          </span>
        </div>
        <div className="negative-feedback-warning-actions">
          <button
            type="button"
            className="btn-refine-negative-question"
            data-testid="btn-refine-negative-question"
            onClick={() => handleRefineFeedbackQuestion(idx, turnIndex)}
            disabled={isStreaming}
          >
            질문 보완
          </button>
        </div>
      </div>
    ) : isUngrounded ? (
      ...
    ) : isLowConfidence ? (
      ...
    ) : null}
    ```

`public-front/src/components/AiService.css`
- `.negative-feedback-warning`: 부정확 평가 인라인 경고 배너 컨테이너 스타일 (적색 테마 배경 `rgba(239, 68, 68, 0.08)`, 테두리 `1px solid rgba(239, 68, 68, 0.3)`, 둥근 모서리, 패딩).
- `.negative-feedback-warning-content`: 경고 아이콘 및 안내 텍스트 플렉스 레이아웃.
- `.negative-feedback-warning-icon`: 경고 아이콘 스타일 (적색 계열 `#ef4444`).
- `.negative-feedback-warning-text`: 안내 문구 스타일 (연한 적색 텍스트 `#fca5a5`, 13px, 줄간격 1.4).
- `.btn-refine-negative-question`: 상단 보완 질문 액션 버튼 스타일 (적색 테마 서브 버튼, 호버 피드백).

## 수용 기준 (Acceptance Criteria)
- Given 신뢰도 0.85, 출처 2건, 결여 항목 없는 AI 답변이 생성되어 `[✓ 지식베이스 근거 검증 완료]` 배지가 표시되어 있을 때
  When 사용자가 해당 답변에 대해 정확도 1점(사유: "환각/사실 오류")으로 피드백을 제출하면
  Then AI 메시지 버블에서 `[✓ 지식베이스 근거 검증 완료]` 배지(`grounded-verified-badge`)가 즉시 사라지고, `[사용자 평가: 부정확]` 배지와 함께 상단 `negative-feedback-warning` 배너가 렌더링된다.
- Given 정확도 2점으로 부정확 평가된 답변 버블에서
  When 사용자가 [수정]을 통해 정확도를 4점으로 변경 제출하거나 [평가 삭제]를 실행하면
  Then 부정확 배지와 상단 경고 배너가 즉시 제거되고, 원래의 `[✓ 지식베이스 근거 검증 완료]` 배지와 `is-grounded-verified` 클래스가 정상 복원된다.
- Given 신뢰도 0.85인 정상 답변에 대해 사용자가 3점 이상(3, 4, 5점)으로 피드백을 등록한 경우 (경계 케이스)
  When 평가 저장이 완료되면
  Then `hasNegativeFeedback`이 거짓이므로 `[✓ 지식베이스 근거 검증 완료]` 배지가 사라지지 않고 계속 유지된다.
- Given 세션 목록에서 과거 세션을 불러왔을 때(세션 복원 `handleLoadSession`), 신뢰도 0.85인 답변에 대해 이미 1점 피드백이 영속화되어 있는 경우 (경계 케이스)
  When 대화 화면이 렌더링되면
  Then 검증 완료 배지 없이 부정확 평가 배지와 상단 `negative-feedback-warning` 배너만 노출된다.
- Given 답변에 사유 태그 없이 자유 의견만 작성된 1점 피드백이 등록되었을 때 (경계 케이스)
  When 상단 `negative-feedback-warning` 배너를 확인하면
  Then 오류 없이 "사용자가 부정확한 답변으로 평가한 내용입니다. 답변 내용을 확인하고 질문을 보완해 보세요." 기본 안내 문구가 표시되고 [질문 보완] 버튼이 정상 작동한다.

## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1844-1860` (`AiService` 내 `isGroundedVerified` 및 `hasNegativeFeedback` 상태 판정부)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1869-1891` (`AiService` 내 메시지 버블 클래스 및 메타 배지 렌더링부)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:2001-2077` (`AiService` 내 인라인 주의 배너 렌더링부)
- 관련 피드백 액션 구현: `public-front/src/components/AiService.tsx:2286-2299` (`AiService` 내 하단 피드백 재질문 버튼부)
- 관련 스타일 정의부: `public-front/src/components/AiService.css:1640-1675` (`AiService.css` 내 검증 완료 배지 및 부정확 평가 배지 스타일 정의부)
- 관련 유틸 정의부: `public-front/src/components/AnswerFeedback.tsx:19-30` (`parseReasonsFromComment` 함수 정의부)

---
id: SPEC-117
title: 사용자 부정확 평가 시 경고 배너 중복 노출 방지 및 상호 배타적 배너 정돈
status: draft
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품의 핵심 목표는 지식베이스 문서를 근거로 질문에 답하되, 사용자가 답변을 그대로 믿지 않고 스스로 검증할 수 있도록 돕는 도구다. 이를 위해 답변 평가 불만족 사유 빠른 선택(SPEC-095), 근거 미참조·저신뢰도 답변의 피드백 연계(SPEC-100), AI 답변 근거 충분 검증 배지(SPEC-108), 사용자 부정확 평가 시 검증 완료 배지 억제 및 상단 부정확 안내 배너 연동(SPEC-114) 등이 단계적으로 배송되었다.
그러나 최근 배송된 기능들이 상호 배타적으로 조화되지 못하여, 사용자가 답변에 대해 부정확 평가(1~2점)를 등록했을 때 화면에 다음과 같은 심각한 상태 모순과 배너 중복 렌더링 결함이 발생한다:
첫째, `public-front/src/components/AiService.tsx:2005-2028` (`AiService` 내 `negative-feedback-warning` 렌더링부)와 `public-front/src/components/AiService.tsx:2030-2106` (`AiService` 내 `ungrounded-warning` 및 `low-confidence-warning` 렌더링부)를 보면, 사용자가 근거가 없거나(`isUngrounded`) 저신뢰도인(`isLowConfidence`) 답변에 대해 정확도 1점 또는 2점으로 평가를 제출하여 `hasNegativeFeedback`이 `true`가 되었을 때, 상단에 `negative-feedback-warning` 배너("사용자가 부정확한 답변으로 평가했습니다.")가 렌더링된다. 그러나 그 바로 아래에 기존의 `ungrounded-warning` 또는 `low-confidence-warning` 배너가 독립적인 분기로 평가되어 화면에 동시에 함께 렌더링된다. 이로 인해 한 AI 메시지 버블 안에 경고성 배너가 2개씩 수직으로 쌓이는 심각한 화면 중복이 발생한다.
둘째, `public-front/src/components/AiService.tsx:2057-2065, 2095-2103` (`AiService` 내 `btn-warning-feedback` 렌더링부)를 보면, 하단에 중복 렌더링된 `ungrounded-warning`과 `low-confidence-warning` 배너 내부에는 여전히 `[답변 평가]` 바로가기 버튼(`btn-warning-feedback`)이 그대로 활성화되어 노출된다. 사용자는 이미 방금 평가를 완료하여 부정확 평가 상태가 되었음에도 불구하고, 화면은 여전히 평가를 다시 하라는 안내 버튼을 노출하고 있어 사용자를 혼란스럽게 만든다. 사용자가 부정확 평가를 등록한 시점에는 이미 평가가 완료되었으므로, 평가 유도 목적의 기존 배너는 억제되고 최상위 부정확 평가 배너(`negative-feedback-warning`)만 단독으로 표시되어야 한다.
셋째, `public-front/src/components/AiService.tsx:2018-2027` (`AiService` 내 `btn-refine-negative-question` 렌더링부)와 `public-front/src/components/AiService.tsx:2315-2328` (`AiService` 내 `btn-feedback-refine` 렌더링부)를 보면, SPEC-114에서 상단 배너 내에 `[질문 보완]` 버튼이 신설되었음에도 불구하고, 답변 최하단 평가 영역 바로 아래에 과거 SPEC-095에서 추가되었던 `[질문 보완하여 다시 질문]` 버튼이 여전히 남아있다. 이로 인해 하나의 메시지 버블 안에서 동일한 동작(`handleRefineFeedbackQuestion`)을 수행하는 액션 버튼이 상단과 하단에 중복 노출되어 UI 정돈도가 떨어진다.
넷째, `public-front/src/components/AiService.tsx:1881-1895` (`AiService` 내 메시지 버블 상단 배지 렌더링부)를 보면, `hasNegativeFeedback === true`일 때 `[사용자 평가: 부정확]` 배지(`negative-feedback-badge`)가 노출되는 동시에, `isLowConfidence === true` 조건도 함께 만족하는 경우 `[답변 신뢰도 낮음]` 배지(`low-confidence-badge`)가 나란히 함께 노출된다. 사용자가 이미 부정확하다고 평가한 답변에서 시스템의 저신뢰도 배지가 함께 뜨는 것은 정보 과잉이며, 사용자 평가 상태가 최우선 권위를 가지도록 배지 표시 규칙을 정돈해야 한다.
이 개선은 "답변을 믿을 수 있게 만든다"는 이번 분기 집중 과제 아래, 사용자가 틀렸다고 판단한 답을 시스템에 되돌려 주었을 때 화면이 일관되고 신뢰할 수 있는 단일 피드백 상태로 즉각 정돈되도록 보장하는 필수 작업이다.
## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 인라인 주의 배너 렌더링 영역에서, 해당 답변에 부정확 평가가 등록된 경우(`hasNegativeFeedback === true`), 하위의 `ungrounded-warning` 및 `low-confidence-warning` 배너가 렌더링되지 않도록 상호 배타성을 보장한다.
- [ ] `hasNegativeFeedback === true`일 때 오직 상단의 `negative-feedback-warning` 배너만 단독 노출되어 배너가 중복 누적 렌더링되지 않도록 한다.
- [ ] 근거 문서가 미참조된 답변(`isUngrounded === true`)에 대해 부정확 평가가 등록된 경우, `negative-feedback-warning` 배너 텍스트에 근거 미참조 정보가 자연스럽게 포함되도록 한다. (예: 사유 태그가 있는 경우 "사용자가 부정확한 답변으로 평가했습니다. (사유: {reasons} · 근거 문서 미참조)", 사유 태그가 없는 경우 "사용자가 부정확한 답변으로 평가했습니다. (근거 문서 미참조)")
- [ ] 사용자가 답변 평가를 3점 이상으로 수정하거나 [평가 삭제]를 통해 부정확 피드백을 철회한 경우(`hasNegativeFeedback === false`), 해당 답변이 원래 근거 미참조(`isUngrounded`)이거나 저신뢰도(`isLowConfidence`)였다면 기존의 `ungrounded-warning` 또는 `low-confidence-warning` 배너(및 내부의 `[답변 평가]`, `[질문 구체화]` 버튼)가 화면에 정상 복원되도록 한다.
- [ ] `public-front/src/components/AiService.tsx`의 메시지 버블 상단 배지 영역에서 `hasNegativeFeedback === true`인 경우 `[답변 신뢰도 낮음]` 배지(`low-confidence-badge`)를 억제하여 `negative-feedback-badge`와 중복 노출되지 않도록 처리한다.
- [ ] 상단 `negative-feedback-warning` 배너 내에 `[질문 보완]` 버튼이 이미 존재하는 점을 고려하여, 하단 피드백 영역 아래의 중복된 `btn-feedback-refine` 버튼 렌더링을 깔끔하게 정리하거나 일관된 조건으로 조화롭게 정돈한다.
- [ ] `hasNegativeFeedback` 상태에 따른 배너 단독 노출, 평가 수정/삭제 시 배너 복원, 근거 미참조 문구 병합을 검증하는 프론트엔드 단위 테스트를 추가 및 갱신한다.
## 비요구사항 (Out of scope)
- 백엔드 피드백 API(`POST /ai/feedback`, `GET /ai/feedback`, `DELETE /ai/feedback`)의 스키마나 DB 구조를 변경하지 않는다.
- `AnswerFeedback.tsx` 컴포넌트 내부의 점수 선택 라디오 버튼이나 불만족 사유 태그 목록(`FEEDBACK_REASONS`)을 수정하지 않는다.
- 단순 RAG 및 에이전틱 RAG 백엔드의 비평 생성기(`CritiqueGeneratorService`)나 검색 알고리즘을 변경하지 않는다.
- 지식베이스 문서 관리 테이블이나 실시간 채팅(`ChatRoom.tsx`), 결제 내역 화면을 수정하지 않는다.
## 프론트엔드
`public-front/src/components/AiService.tsx`
- 인라인 배너 렌더링 조건문 정돈:
  - `hasNegativeFeedback`를 최우선 단독 분기로 분리하거나, 기존 `isUngrounded` 및 `isLowConfidence` 렌더링 가드에 `!hasNegativeFeedback` 조건을 추가한다.
  - 배너 렌더링 구조:
    ```tsx
    {hasNegativeFeedback ? (
      <div className="negative-feedback-warning" data-testid="negative-feedback-warning">
        <div className="negative-feedback-warning-content">
          <span className="negative-feedback-warning-icon" aria-hidden="true">
            ⚠
          </span>
          <span className="negative-feedback-warning-text">
            {feedbackReasons.length > 0
              ? `사용자가 부정확한 답변으로 평가했습니다. (사유: ${feedbackReasons.join(', ')}${isUngrounded ? ' · 근거 문서 미참조' : ''})`
              : isUngrounded
                ? '사용자가 부정확한 답변으로 평가한 내용입니다. (근거 문서 미참조) 답변 내용을 확인하고 질문을 보완해 보세요.'
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
      <div className="ungrounded-warning" data-testid="ungrounded-warning">
        ...
      </div>
    ) : isLowConfidence ? (
      <div className="low-confidence-warning" data-testid="low-confidence-warning">
        ...
      </div>
    ) : null}
    ```
- 상단 배지 영역 배타성 보장:
  - `isLowConfidence && !hasNegativeFeedback && (<div className="low-confidence-badge" ...>[답변 신뢰도 낮음]</div>)` 형태로 조건 가드 보강.
- 하단 피드백 재질문 버튼 정리:
  - 상단 배너에서 이미 `[질문 보완]` 버튼이 제공되므로, `!hasNegativeFeedback`이거나 상단 배너와 중복되지 않도록 하단 `btn-feedback-refine` 노출을 정돈.
## 수용 기준 (Acceptance Criteria)
- Given 사용자가 지식베이스와의 관련도가 낮아 `low-confidence-warning` 배너(신뢰도 60% 미만)가 표시된 답변을 보고 있을 때
  When 사용자가 `[답변 평가]`를 눌러 정확도 1점과 불만족 사유를 선택하고 평가를 제출하면
  Then 상단에 `negative-feedback-warning` 배너만 단독으로 표시되고, 기존의 `low-confidence-warning` 배너 및 내부의 `[답변 평가]` 버튼은 즉시 사라진다.
- Given 근거 문서가 발견되지 않아 `ungrounded-warning` 배너가 표시된 답변에 대해
  When 사용자가 정확도 2점으로 부정확 평가를 등록하면
  Then `ungrounded-warning` 배너가 닫히고 `negative-feedback-warning` 배너에 "근거 문서 미참조" 안내가 함께 결합되어 렌더링된다.
- Given 부정확 평가(1점)가 등록되어 `negative-feedback-warning` 배너가 표시된 상태에서
  When 사용자가 평가 폼에서 [평가 삭제]를 클릭하여 피드백을 철회하면
  Then `negative-feedback-warning` 배너가 제거되고, 원래의 `low-confidence-warning` 또는 `ungrounded-warning` 배너가 화면에 다시 복원된다.
- Given 근거가 충분하여 검증 완료(`isGroundedVerified === true`) 상태였던 답변에서
  When 사용자가 정확도 1점으로 부정확 평가를 등록하면
  Then `isGroundedVerified` 배지가 억제되고 오직 `negative-feedback-badge` 및 `negative-feedback-warning` 배너만 노출된다.
- Given 평가가 등록되지 않았거나 3점 이상(보통/만족)으로 평가된 일반 답변인 경우 (경계 케이스)
  When 화면을 확인하면
  Then `negative-feedback-warning` 배너가 노출되지 않으며 기존의 정상 근거 배너 및 배지가 유지된다.
## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:2005-2028` (`negative-feedback-warning` 렌더링부)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:2030-2106` (`ungrounded-warning` 및 `low-confidence-warning` 렌더링부)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1881-1895` (`missing-warning-badge`, `low-confidence-badge`, `negative-feedback-badge` 배지 렌더링부)
- 관련 기존 구현: `public-front/src/components/AiService.tsx:1845-1861` (`hasNegativeFeedback` 및 `isGroundedVerified` 판정부)
- 관련 기존 구현: `public-front/src/components/AiService.tsx:2315-2328` (`btn-feedback-refine` 렌더링부)
- 관련 기존 구현: `public-front/src/components/AiService.tsx:1250-1285` (`handleRefineFeedbackQuestion`)

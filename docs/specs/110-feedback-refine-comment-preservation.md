---
id: SPEC-110
title: 답변 평가(불만족 피드백) 기반 재질문 시 사용자 상세 의견 유실 수정 및 재질문 편의성 개선
status: done
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품의 핵심 목표는 지식베이스 문서를 근거로 질문에 답하되, 사용자가 답변을 그대로 믿지 않고 스스로 검증할 수 있도록 돕는 도구다. 이번 분기의 핵심 집중 과제는 "사용자가 틀렸다고 판단한 답을 시스템에 되돌려 줄 수단이 있는가"이며, 이를 위해 답변 정확도/유용성 평가(SPEC-024), 평가 상태 비동기 동기화(SPEC-051), 평가 삭제 지원(SPEC-068), 불만족 사유 빠른 선택 및 질문 보완 재질문 연계(SPEC-095), 근거 미참조/저신뢰도 답변의 피드백 연계 개선(SPEC-100), 불만족 사유 요약 노출 및 부정확 답변 경고 배지 시각화(SPEC-107)가 단계적으로 배송되었다.
그러나 사용자가 틀리거나 부족한 답변에 대해 정성껏 피드백을 남기고 이를 바탕으로 후속 질문을 이어가는 핵심 흐름에서 다음과 같은 심각한 사용자 의견 유실 결함과 편의성 결여가 존재한다:
첫째, `public-front/src/components/AiService.tsx:1157-1185` (`handleRefineFeedbackQuestion`)를 보면, 사용자가 답변 평가 폼에서 1~2점 불만족 평가를 남긴 뒤 노출되는 `[질문 보완하여 다시 질문]`(`btn-feedback-refine`) 버튼 클릭 시 실행되는 핸들러다. 여기서 피드백 코멘트(`rawComment`)를 파싱할 때 `const tagMatches = rawComment.match(/\[([^\]]+)\]/g);`로 대괄호 태그를 검사한 뒤, `if (tagMatches && tagMatches.length > 0)`이면 오직 추출된 사유 태그들만 `extractedReason = tagMatches.join(' ')`로 바인딩한다. 이로 인해 사용자가 사유 칩(예: `[근거 문서와 다름]`)을 누르고 입력창에 `2024년 2분기 영업이익 기준이 아닙니다`라고 구체적인 수정 요청을 직접 작성했음에도, 태그 뒤에 작성한 상세 의견 본문이 통째로 증발(유실)되고 질문창에는 `원래 질문 (보완 요청: [근거 문서와 다름])`만 남는다. 정작 모델이 답변을 바로잡는 데 가장 중요한 사용자의 구체적 지시사항이 누락되어, 모델은 무엇이 다른지 알지 못한 채 동일한 실수를 반복하게 된다.
둘째, `public-front/src/components/AnswerFeedback.tsx:15-37` (`parseReasonsFromComment`, `stripReasonTags`)를 보면, 사용자가 작성한 코멘트에서 사유 태그 목록을 추출하는 `parseReasonsFromComment`와 사유 태그를 제거하여 순수 사용자 의견만 분리하는 `stripReasonTags` 함수가 이미 구현되어 있다. 하지만 이 유용한 유틸리티 함수들과 허용 사유 목록 상수(`FEEDBACK_REASONS`)가 모듈 내부에 비공개(unexported)로 닫혀 있다. 이 때문에 `AiService.tsx`가 검증된 파싱 유틸리티를 재사용하지 못하고 임시 정규식으로 태그만 추출하면서 사용자 상세 의견을 유실시키는 원인이 되었다.
셋째, `public-front/src/components/AiService.tsx:1152-1155` (`handleRefineQuestion`) 및 `handleRefineFeedbackQuestion`에서는 질문 입력창에 보완된 질문 텍스트를 채운 뒤 `chatInputRef.current?.focus()`만 호출하고 있다. 대화가 여러 턴 누적되어 스크롤이 길어진 화면에서 상단 답변 버블의 `[질문 보완하여 다시 질문]` 버튼을 눌렀을 때, 화면 스크롤이 이전 답변 위치에 그대로 멈춰 있어 사용자는 질문창에 내용이 채워졌는지 즉시 인지하지 못하고 직접 아래로 스크롤해야 하는 불편을 겪는다.
넷째, `public-front/src/components/AiService.tsx:2172-2183` (`AiService` 내 메시지 버블 액션 렌더링부)에서 `btn-feedback-refine` 버튼에 접근성 라벨(`aria-label`)과 명확한 테스트 식별자(`data-testid`)가 결여되어 있으며, 버튼 클릭 시 질문창에 보완 요청이 정상 복원되었음을 알리는 시각적 피드백이 부족하다.
따라서 사유 태그와 사용자 직접 입력 의견을 모두 온전히 보존하여 재질문 텍스트로 합성하도록 수정하고, 파싱 유틸리티를 공용화하며, 입력창 자동 스크롤 및 피드백 알림을 보강하여 사용자가 잘못된 답변을 시스템에 되돌려주고 즉시 개선된 답변을 유도할 수 있도록 개선해야 한다.
## 요구사항
- [ ] `public-front/src/components/AnswerFeedback.tsx`의 `parseReasonsFromComment`, `stripReasonTags`, `FEEDBACK_REASONS`를 외부로 `export`하여 다른 컴포넌트 및 테스트에서 일관되게 재사용할 수 있도록 한다.
- [ ] `public-front/src/components/AiService.tsx`의 `handleRefineFeedbackQuestion`에서 피드백 코멘트(`rawComment`)에 사유 태그와 사용자 직접 작성 상세 의견이 모두 존재하는 경우, 둘 다 보존하여 `"${questionText} (보완 요청: ${tags} ${customComment})"` 형식으로 질문 입력창에 바인딩한다.
  - 사유 태그와 상세 의견이 모두 있는 경우: 예) `"${questionText} (보완 요청: [근거 문서와 다름] 2024년 2분기 영업이익 기준)"`
  - 사유 태그만 있는 경우: 예) `"${questionText} (보완 요청: [환각/사실 오류])"`
  - 상세 의견만 있는 경우: 예) `"${questionText} (보완 요청: 최신 규정 반영 필요)"`
  - 피드백 내용이 전혀 없거나 공백인 경우: 예) `"${questionText}"`
- [ ] `public-front/src/components/AiService.tsx`의 `handleRefineFeedbackQuestion` 및 `handleRefineQuestion` 실행 시 질문 텍스트를 바인딩하고 입력창에 포커스를 주는 것과 함께, 하단 질문 입력창 엘리먼트(`chatInputRef.current`) 위치로 화면을 부드럽게 스크롤(`scrollIntoView({ behavior: 'smooth', block: 'nearest' })`) 이동시킨다.
- [ ] `[질문 보완하여 다시 질문]` 버튼 클릭 시 사용자가 질문 보완 반영을 즉각 체감할 수 있도록 "불만족 피드백이 질문 입력창에 반영되었습니다." 토스트 알림 메시지를 화면에 노출한다.
- [ ] `public-front/src/components/AiService.tsx`의 `btn-feedback-refine` 버튼에 `data-testid="btn-feedback-refine"` 속성과 스크린 리더용 `aria-label="불만족 사유를 반영하여 질문 다시 작성"` 속성을 추가한다.
- [ ] 사유 태그와 사용자 상세 코멘트 동시 보존, 단독 사유 태그 보존, 단독 커스텀 코멘트 보존, 빈 코멘트 폴백, 입력창 스크롤 및 토스트 피드백 동작을 검증하는 프론트엔드 단위 테스트를 작성한다.
## 비요구사항 (Out of scope)
- 백엔드 피드백 DTO 규격(`SubmitFeedbackIn`, `FeedbackOut`)이나 MongoDB 스키마 자체를 수정하지 않는다.
- `AnswerFeedback.tsx`의 내부 별점 평가 척도(1~5점)나 불만족 사유 칩 5개 항목 자체를 수정하지 않는다.
- 실시간 채팅(`ChatRoom.tsx`)이나 결제 내역(`PaymentHistory.tsx`) 컴포넌트를 수정하지 않는다.
- 지식베이스 문서 인제스트 또는 벡터 검색 알고리즘을 변경하지 않는다.
## 프론트엔드
`public-front/src/components/AnswerFeedback.tsx`
- 유틸리티 함수 및 상수 내보내기:
  - `export const FEEDBACK_REASONS = [...] as const;`
  - `export const parseReasonsFromComment = (comment: string): string[] => { ... }`
  - `export const stripReasonTags = (comment: string): string => { ... }`
`public-front/src/components/AiService.tsx`
- `handleRefineFeedbackQuestion` 로직 정정:
  - `AnswerFeedback`으로부터 `parseReasonsFromComment`, `stripReasonTags` 임포트.
  - 코멘트 파싱:
    ```typescript
    const rawComment = (feedback?.comment || '').trim();
    let extractedReason = '';
    if (rawComment) {
      const reasons = parseReasonsFromComment(rawComment);
      const customComment = stripReasonTags(rawComment);
      const tagText = reasons.map((r) => `[${r}]`).join(' ');
      if (tagText && customComment) {
        extractedReason = `${tagText} ${customComment}`;
      } else if (tagText) {
        extractedReason = tagText;
      } else if (customComment) {
        extractedReason = customComment;
      }
    }
    ```
  - 질문 바인딩 및 부드러운 스크롤 이동:
    ```typescript
    if (extractedReason) {
      setQuestion(`${questionText} (보완 요청: ${extractedReason})`);
    } else {
      setQuestion(questionText);
    }
    chatInputRef.current?.focus();
    chatInputRef.current?.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
    showBookmarkToast('불만족 피드백이 질문 입력창에 반영되었습니다.');
    ```
- `btn-feedback-refine` 엘리먼트 속성 보강:
  - `data-testid="btn-feedback-refine"`
  - `aria-label="불만족 사유를 반영하여 질문 다시 작성"`
## 수용 기준 (Acceptance Criteria)
- Given 사용자가 정확도 1점과 함께 사유 태그 `[근거 문서와 다름]`을 선택하고 상세 코멘트로 `"2024년 2분기 영업이익 기준이 아닙니다"`를 입력하여 피드백을 저장한 답변이 있을 때 When 사용자가 해당 답변 버블 하단의 `[질문 보완하여 다시 질문]`(`btn-feedback-refine`) 버튼을 클릭하면 Then 직전 사용자 질문 뒤에 `(보완 요청: [근거 문서와 다름] 2024년 2분기 영업이익 기준이 아닙니다)`가 온전히 결합되어 하단 질문 입력창에 채워지고, 입력창에 포커스되며, "불만족 피드백이 질문 입력창에 반영되었습니다." 토스트 알림이 표시된다.
- Given 사용자가 사유 태그 없이 자유 텍스트로만 `"표 형식으로 다시 정리해줘"`라는 코멘트를 남긴 불만족 답변이 있을 때 When `[질문 보완하여 다시 질문]` 버튼을 클릭하면 Then 하단 질문 입력창에 `"${questionText} (보완 요청: 표 형식으로 다시 정리해줘)"`가 채워진다.
- Given 사용자가 사유 칩 `[환각/사실 오류]` `[최신 정보 아님]` 2개를 선택하고 추가 코멘트는 작성하지 않은 불만족 답변이 있을 때 When `[질문 보완하여 다시 질문]` 버튼을 클릭하면 Then 하단 질문 입력창에 `"${questionText} (보완 요청: [환각/사실 오류] [최신 정보 아님])"`이 채워진다.
- Given (경계 케이스) 피드백에 코멘트가 전혀 없거나 공백 문자만 입력된 불만족 답변이 있을 때 When `[질문 보완하여 다시 질문]` 버튼을 클릭하면 Then 보완 요청 접미사 없이 원래의 `"${questionText}"`만 질문 입력창에 복원된다.
- Given (경계 케이스) 스트리밍이 진행 중인 상태(`isStreaming === true`)일 때 When 답변 버블들을 확인하면 Then 모든 `[질문 보완하여 다시 질문]` 버튼이 비활성화(`disabled`)되어 중복 클릭이나 입력창 덮어쓰기가 방지된다.
## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1157-1185` (`handleRefineFeedbackQuestion`)
- 고쳐야 할 자리: `public-front/src/components/AnswerFeedback.tsx:15-37` (`parseReasonsFromComment`, `stripReasonTags`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1152-1155` (`handleRefineQuestion`)
- 관련 기존 구현: `public-front/src/components/AiService.tsx:2172-2183` (`btn-feedback-refine` 렌더링부)
- 관련 기존 구현: `public-front/src/components/AiService.test.tsx:9140-9162` (`handleRefineFeedbackQuestion` 기존 테스트부)

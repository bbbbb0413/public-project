---
id: SPEC-064
title: 답변 평가(피드백) 제출 토스트 알림 및 평가 점수별 상태 요약 시각화
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
사용자가 RAG 답변의 신뢰성과 정확도를 점검하고 피드백을 시스템에 되돌려주는 루프는 이번 분기 핵심 목표 중 하나다. 그러나 현재 프론트엔드의 피드백 UI는 제출 후 상태 피드백과 평가 완료 요약 시각화 측면에서 다음과 같은 사용성 결함이 존재한다.
- `public-front/src/components/AnswerFeedback.tsx:72-102` (`AnswerFeedback.handleSubmit`)에서 평가 저장 성공 시 아무런 완료 알림 없이 폼이 닫힌다. 반면 세션/답변 보관 기능(`public-front/src/components/AiService.tsx:125-129` (`showBookmarkToast`))은 토스트 알림을 제공하고 있어 인터랙션 일관성이 결여되어 있고 사용자가 저장이 정상 완료되었는지 즉각 인지하기 어렵다.
- `public-front/src/components/AiService.tsx:309-316` (`handleSubmitFeedback`) 및 `public-front/src/components/AiService.tsx:1154-1159` (`chatLog.map`)에서 평가 제출이 완료되어도 대화창 상단이나 토스트 영역에 성공 메시지를 전달하는 피드백 처리가 누락되어 있다.
- `public-front/src/components/AnswerFeedback.tsx:108-117` (`AnswerFeedback`) 및 `public-front/src/components/AiService.css:1546-1550`에서 이미 평가가 완료된 답변은 사용자가 매긴 점수(정확도/유용성)와 상관없이 단일 텍스트 색상(`#a5b4fc`)으로만 표시된다. 이로 인해 자신이 답변에 오류가 있어 낮은 점수(1~2점)를 주었는지, 정확하여 높은 점수(4~5점)를 주었는지 대화 이력에서 한눈에 구분할 수 없다.
- `public-front/src/components/AnswerFeedback.tsx:148-158` (`AnswerFeedback`)에서 의견 입력창에 `maxLength={MAX_COMMENT_LENGTH}`(1000자)가 설정되어 있으나 실시간 글자 수 카운터가 표시되지 않아 긴 피드백을 작성할 때 잔여 입력 가능 분량을 확인하기 어렵다.

## 요구사항
- [ ] `public-front/src/components/AnswerFeedback.tsx`에서 평가 저장(`handleSubmit`) 성공 시 즉각적인 성공 피드백을 표시하거나, `onSuccess` 콜백을 통해 상위 컴포넌트로 알린다.
- [ ] `public-front/src/components/AiService.tsx`의 `handleSubmitFeedback`에서 피드백 제출 완료 시 "답변 평가가 저장되었습니다." (기존 평가 수정인 경우 "답변 평가가 수정되었습니다.") 토스트 알림 메시지를 노출한다.
- [ ] `public-front/src/components/AnswerFeedback.tsx`의 접힌 평가 요약 영역(`feedback-summary`)에 정확도 점수에 따른 수준별 CSS 클래스(`feedback-positive`: 4~5점, `feedback-neutral`: 3점, `feedback-negative`: 1~2점)를 적용한다.
- [ ] `public-front/src/components/AiService.css`에 정확도 수준별 텍스트/배지 색상 테마(positive: 녹색 계열, neutral: 황색/노랑 계열, negative: 주황/적색 계열) 스타일을 추가한다.
- [ ] `public-front/src/components/AnswerFeedback.tsx`의 의견(comment) 입력 textarea 하단에 실시간 글자 수 카운터(`{comment.length} / 1000자`)를 렌더링한다.
- [ ] `public-front/src/components/AnswerFeedback.test.tsx`에 피드백 저장 완료 알림, 정확도 점수별 배지 클래스 렌더링, 글자 수 카운터 동작을 검증하는 단위 테스트를 추가한다.

## 비요구사항 (Out of scope)
- 백엔드 피드백 API(`POST /rag/feedback`, `GET /rag/feedback/:sessionId`)의 스키마나 라우팅 로직을 변경하지 않는다.
- 피드백 삭제 기능이나 관리자 피드백 통계 대시보드 신설은 이번 범위에 포함하지 않는다.
- 피드백 평가 항목(현재: 정확도, 유용성, 의견)의 척도나 종류를 변경하지 않는다.

## 프론트엔드
- `public-front/src/components/AnswerFeedback.tsx`
  - `Props` 인터페이스에 `onSuccess?: (isUpdate: boolean) => void` 선택적 콜백을 지원하거나 자체 성공 피드백을 처리한다.
  - 정확도 점수(`existing.accuracy`)에 따른 상태 클래스 계산 함수 또는 클래스 바인딩을 적용한다.
  - 의견 텍스트 영역 하단에 `.feedback-char-count` 요소를 추가하여 현재 글자 수와 최대 글자 수를 표시한다.
- `public-front/src/components/AiService.tsx`
  - `handleSubmitFeedback`에서 저장 성공 시 `showBookmarkToast`와 유사한 방식 또는 공통 토스트 알림을 통해 성공 메시지를 화면에 노출한다.
- `public-front/src/components/AiService.css`
  - `.feedback-summary.feedback-positive`, `.feedback-summary.feedback-neutral`, `.feedback-summary.feedback-negative` 스타일을 추가한다.
  - `.feedback-char-count` 글자 수 카운터 스타일을 추가한다.

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 세션 내 AI 답변의 "이 답변을 평가하기" 버튼을 눌러 평가 폼을 연 상태에서
  When 정확도 5점, 유용성 4점을 선택하고 의견을 입력한 뒤 "평가 제출" 버튼을 클릭하면
  Then 평가가 저장되고 폼이 닫히며 "답변 평가가 저장되었습니다." 토스트 알림이 화면에 표시되고, 접힌 요약 영역에 `feedback-positive` 스타일이 적용되어 렌더링된다.
- Given 이미 정확도 1점으로 평가가 등록된 AI 답변이 있을 때
  When 사용자가 대화창에서 해당 답변의 평가 요약을 확인하면
  Then 요약 텍스트에 `feedback-negative` 클래스가 부여되어 경고/부정 피드백 색상으로 표시된다.
- Given 사용자가 피드백 수정 폼을 열었을 때
  When 의견 텍스트 영역에 120자의 텍스트를 입력하면
  Then 하단 글자 수 카운터에 "120 / 1000자"가 정확히 렌더링된다.
- Given 의견(comment)을 전혀 입력하지 않은 빈 상태(0자)일 때
  When 피드백 폼을 열면
  Then 글자 수 카운터에 "0 / 1000자"가 정상 표시되고 오류가 발생하지 않는다.

## 참고
- 고쳐야 할 자리: `public-front/src/components/AnswerFeedback.tsx:72-102` (`AnswerFeedback.handleSubmit`)
- 고쳐야 할 자리: `public-front/src/components/AnswerFeedback.tsx:108-117` (`AnswerFeedback`)
- 고쳐야 할 자리: `public-front/src/components/AnswerFeedback.tsx:148-158` (`AnswerFeedback`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:309-316` (`handleSubmitFeedback`)
- 관련 스타일 정의: `public-front/src/components/AiService.css:1546-1550`

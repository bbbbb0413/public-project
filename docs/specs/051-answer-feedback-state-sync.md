---
id: SPEC-051
title: 답변 평가(피드백) 비동기 상태 동기화 및 수정 폼 복원 개선
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
문서 기반 AI 답변의 품질을 개선하고 사용자가 틀렸거나 부족한 답변을 시스템에 알릴 수 있도록 답변 평가(`AnswerFeedback`) 기능이 도입되었으나, 대화 세션을 불러올 때 비동기 로딩 타이밍 불일치로 인해 기존 평가 수정 폼이 빈 값으로 열리고 수정이 불가능해지는 결함이 존재한다.
- `public-front/src/components/AnswerFeedback.tsx:72-81` (`AnswerFeedback`)에서 `accuracy`, `helpfulness`, `comment` 로컬 상태를 `useState(existing?.accuracy ?? null)`와 같이 마운트 시점의 초기값으로만 설정하고 있다.
- `public-front/src/components/AiService.tsx:320-330` (`handleLoadSession`)에서 세션 대화 내역(`setChatLog`)을 먼저 설정한 뒤 `loadFeedback`을 비동기로 호출하므로, `public-front/src/components/AiService.tsx:1153-1159` (`AiService`)의 `AnswerFeedback` 컴포넌트는 처음에 `existing=undefined` 상태로 마운트된다. 이후 `feedbackByTurn`이 로드되어 `existing` prop이 전달되어도 내부 `useState`가 갱신되지 않는다.
- `public-front/src/components/AnswerFeedback.tsx:104-128` (`AnswerFeedback`)에서 `existing`이 존재하여 요약 배너("내 평가 · 정확도 N/5 · 유용성 M/5")와 [수정] 버튼이 정상 노출되지만, 사용자가 [수정]을 클릭하여 폼을 열었을 때(`setIsOpen(true)`) 내부 점수 상태가 `null`로 남아있어 라디오 버튼이 모두 해제된 빈 폼으로 표시되고, `canSubmit` 조건 미충족으로 인해 "평가 수정" 버튼이 비활성화된다.
- `public-front/src/components/AnswerFeedback.tsx:167-175` (`AnswerFeedback`)에서 사용자가 평가 수정 중 값을 변경하다가 [취소]를 클릭한 뒤 다시 [수정]을 열었을 때, 기존 `existing` 값으로 롤백되지 않고 미저장된 입력값이 남아있는 상태 오염 문제가 발생한다.

## 요구사항
- [ ] `AnswerFeedback` 컴포넌트에서 `existing` prop이 비동기로 주입되거나 변경될 때, 또는 사용자가 [수정] 버튼을 클릭하여 폼을 열 때 기존 평가 데이터(`accuracy`, `helpfulness`, `comment`)가 입력 상태에 즉시 동기화되어야 한다.
- [ ] 기존 평가가 등록된 답변에서 [수정] 버튼을 클릭했을 때 기존에 선택했던 정확도 및 유용성 라디오 버튼이 선택된 상태로 렌더링되고, 기존 작성 의견(comment)이 텍스트 영역에 표시되어야 한다.
- [ ] 기존 평가가 채워진 상태로 수정 폼이 열리면 "평가 수정" 버튼이 활성화(`enabled`) 상태여야 한다.
- [ ] 수정 폼에서 값을 변경하다가 [취소] 버튼을 클릭하면 미저장된 입력값이 취소되고, 다시 폼을 열었을 때 기존 `existing` 평가 데이터로 정상 복원되어야 한다.
- [ ] 신규 답변에 대해 처음 평가를 등록할 때([이 답변을 평가하기])는 기존과 동일하게 모든 항목이 빈 상태로 열리고 유효한 점수 입력 전까지 제출 버튼이 비활성화되어야 한다.
- [ ] 프론트엔드 단위 테스트(`AnswerFeedback.test.tsx`)에 비동기 `existing` prop 주입 시 수정 폼 동기화 및 취소 시 상태 롤백을 검증하는 테스트 케이스를 추가한다.

## 비요구사항 (Out of scope)
- 백엔드 피드백 API 스키마 변경, 엔드포인트 수정 및 데이터베이스 마이그레이션.
- 피드백 삭제(DELETE) 엔드포인트 신설.
- RAG 질의응답 및 스트리밍 로직 변경.
- 관리자 화면의 피드백 통계 UI 변경.

## 프론트엔드
- `public-front/src/components/AnswerFeedback.tsx`:
  - `existing` prop 변화 및 `isOpen` 전환 시 `accuracy`, `helpfulness`, `comment` 상태 동기화 처리.
  - [수정] 버튼 클릭 시 기존 `existing` 값으로 폼 상태를 리셋 및 채움.
  - [취소] 버튼 핸들러에서 폼 상태를 `existing` 값으로 초기화하고 폼 닫기.
  - `existing` prop이 비동기로 뒤늦게 도착하는 시나리오 대응.
- `public-front/src/components/AnswerFeedback.test.tsx`:
  - 초기 마운트 이후 `existing` prop이 `rerender`를 통해 비동기로 전달되었을 때 [수정] 클릭 시 기존 평가가 정상 선택되어 열리는지 검증.
  - 수정 중 취소 후 재진입 시 기존 값으로 복원되는지 검증.

## 수용 기준 (Acceptance Criteria)
- Given 이전에 정확도 4점, 유용성 5점, 의견 "상세한 답변"으로 평가를 남긴 대화 세션을 불러왔을 때
  When 사용자가 해당 답변의 [수정] 버튼을 클릭하면
  Then 정확도 4점, 유용성 5점 라디오 버튼이 체크되어 있고 의견란에 "상세한 답변"이 입력된 채로 폼이 열리며 "평가 수정" 버튼이 활성화된다.
- Given 사용자가 기존 평가(정확도 4점, 유용성 5점)를 수정하기 위해 폼을 열고 정확도를 2점으로 변경한 뒤
  When [취소] 버튼을 누르고 다시 [수정] 버튼을 클릭하면
  Then 변경하려던 2점이 아닌 원래의 4점/5점 상태로 깨끗하게 복원되어 표시된다.
- Given 아직 평가하지 않은 신규 AI 답변인 경우 (`existing`이 없는 경우)
  When "이 답변을 평가하기" 버튼을 클릭하면
  Then 모든 라디오 버튼이 미선택 상태이고 의견란이 비어 있으며 "평가 제출" 버튼이 비활성화 상태로 표시된다.

## 참고
- 답변 평가 컴포넌트 상태 및 렌더링: `public-front/src/components/AnswerFeedback.tsx:72-186` (`AnswerFeedback`)
- 세션 및 피드백 로딩 연동: `public-front/src/components/AiService.tsx:320-330` (`handleLoadSession`), `public-front/src/components/AiService.tsx:1153-1159` (`AiService`)
- 답변 평가 단위 테스트: `public-front/src/components/AnswerFeedback.test.tsx:88-97`

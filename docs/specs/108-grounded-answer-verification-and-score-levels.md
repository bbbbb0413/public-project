---
id: SPEC-108
title: AI 답변 근거 충분 검증 배지 및 출처 청크 관련도 수준별 시각화와 근거 미발견 판정 결함 수정
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품의 핵심 목표는 올려 둔 문서를 근거로 질문에 답하되, 사용자가 답변을 그대로 믿지 않고 스스로 검증할 수 있도록 돕는 도구다. 지금까지 추론 과정 노출(SPEC-005), 근거 미리보기(SPEC-006), 신뢰도 전달(SPEC-010, SPEC-018), 근거 문서 미참조 경고(SPEC-059), 저신뢰도 경고 배너(SPEC-093), 확인하지 못한 항목 보완(SPEC-075, SPEC-104) 등이 단계적으로 배송되었다.
그러나 "근거가 부족한 답과 충분한 답이 화면에서 명확히 구분되는가"라는 이번 분기 제품 방향과 관련하여, 현재 사용자 화면에는 다음과 같은 심각한 판정 결함과 시각적 구분의 부재가 존재한다:
첫째, `public-front/src/components/AiService.tsx:1684-1687` (`AiService` 내 메시지 버블 상태 판별부)에서 근거 미발견 플래그 `isUngrounded`를 `isAi && isNoSources && (msg.confidence === 0 || !msg.confidence)`로 계산하고 있다. 이로 인해 참고 문서 목록이 비어 있거나 누락되어 출처가 전혀 없는 답변(`isNoSources === true`)임에도, 모델이 자체 산출한 `confidence`가 0보다 큰 값(예: 0.5, 0.7 등)으로 전달되는 경우 `isUngrounded`가 `false`로 오판정된다. 또한 `isLowConfidence` 역시 `msg.confidence < 0.6 && !isUngrounded` 조건으로 묶여 있어 신뢰도가 0.6 이상인 경우 경고에서 제외된다. 결국 출처 문서가 단 하나도 참조되지 않은 일반 지식 답변인데도 화면에 아무런 주의 배너(`ungrounded-warning`)가 노출되지 않고 마치 정상적인 지식 기반 답변인 것처럼 둔갑하는 결함이 발생한다.
둘째, `public-front/src/components/AiService.tsx:1702-1715` (`AiService` 내 메시지 버블 래퍼부)에서 `message-bubble` 엘리먼트에 미확인 항목(`has-missing`), 저신뢰도(`has-low-confidence`) 클래스만 부여하고 있다. 가장 치명적인 환각 위험 상태인 `isUngrounded`(근거 문서 미발견) 상태에 대해서는 버블 자체의 시각적 격리 스타일(`has-ungrounded`)이 전무하여, 상단 배너를 제외하면 버블 외형이 일반 정상 답변과 완전히 동일하게 방치된다.
셋째, 근거가 부족한 답변에 대한 경고(`ungrounded-warning`, `low-confidence-warning`)는 마련되었으나, 반대로 신뢰도가 80% 이상이고 출처가 1건 이상 확실히 확보된 "근거가 충분한 답변(High Grounding & High Confidence)"에 대해서는 긍정적 검증 상태를 보증하는 피드백이 전무하다. `public-front/src/components/AiService.tsx:1930-1947` (`AiService` 내 메타 배지 렌더링부)에서 단순 숫자 텍스트(`신뢰도: 85%`)만 표시될 뿐, 답변 상단에 이 답변이 지식베이스 문서를 바탕으로 근거 검증을 마쳤음을 알리는 `[지식베이스 근거 검증 완료]`(Verified Answer) 상태 배지가 없어 사용자가 안심하고 답변을 신뢰할 수 있는 시각적 확신을 얻지 못한다.
넷째, `public-front/src/components/AiService.tsx:2031-2036` (`AiService` 내 참고 문서 청크 렌더링부) 및 `public-front/src/components/AiService.tsx:2203-2208` (`AiService` 내 스트리밍 참고 문서 청크 렌더링부)에서 각 청크의 관련도 점수를 `source-ref-score` 단일 클래스로만 렌더링하고 있다. `public-front/src/components/AiService.css:1229-1235` (`.source-ref-score` 스타일 정의부)를 보면 모든 점수가 획일적인 연보라색(`#a5b4fc`) 단색으로만 지정되어 있어, 관련도가 80% 이상인 핵심 결정적 근거 청크와 30% 수준의 부차 청크가 동일한 외형으로 표시된다. 이로 인해 사용자가 수많은 청크 중 어떤 청크가 답변을 뒷받침하는 핵심 근거인지 직관적으로 파악하기 어렵다.
따라서 근거 미발견 판정 조건을 바로잡고, 근거가 충분한 고신뢰도 답변에 대해 `[지식베이스 근거 검증 완료]` 상태 배지를 노출하며, 출처 청크의 관련도 점수에 수준별(고·중·저) 시각 테마를 부여하여 사용자가 근거가 충분한 답과 부족한 답을 한눈에 식별하고 답변을 신뢰할 수 있도록 개선해야 한다.
## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 `isUngrounded` 판정 로직을 `isAi && isNoSources`로 정정하여, `confidence` 수치와 관계없이 출처(`msg.sources`)가 비어 있거나 누락된 모든 AI 답변에 대해 근거 문서 미발견 상태로 판정한다.
- [ ] `isUngrounded === true`인 AI 메시지 버블(`message-bubble`)에 `has-ungrounded` CSS 클래스를 적용하여, 테두리 및 배경에 시각적 주의 스타일을 적용한다.
- [ ] 신뢰도가 80% 이상이고(`msg.confidence >= 0.8`), 출처가 1건 이상 존재하며(`!isNoSources`), 확인하지 못한 항목이 없고(`displayMissing.length === 0`), 중단/에러 상태가 아닌(`!msg.isCancelled && !msg.isError`) 정상 AI 메시지 버블에 대해 `isGroundedVerified` 상태로 판정한다.
- [ ] `isGroundedVerified === true`인 AI 메시지 버블(`message-bubble`)에 `is-grounded-verified` CSS 클래스를 적용하여 안정적인 신뢰 테두리 스타일을 부여한다.
- [ ] `isGroundedVerified === true`인 AI 메시지 상단(메타 배지 영역 또는 경고 배너 위치)에 `[지식베이스 근거 검증 완료]` 상태 배지(`grounded-verified-badge`, `data-testid="grounded-verified-badge"`)를 렌더링한다.
- [ ] `public-front/src/components/AiService.tsx`의 완료 메시지 및 스트리밍 메시지 참고 문서 목록에서, 각 청크(`chunk.score`) 및 문서 그룹(`group.maxScore`)의 점수(`score`)에 따라 수준별 CSS 클래스를 부여한다.
  - 관련도 80% 이상(`score >= 0.8`): `score-high`
  - 관련도 60% 이상 80% 미만(`score >= 0.6 && score < 0.8`): `score-medium`
  - 관련도 60% 미만(`score < 0.6`): `score-low`
- [ ] 점수 데이터(`score`)가 `undefined`이거나 유효한 숫자가 아닌 경우, 기본 `source-ref-score` 스타일만 유지하고 수준별 클래스는 부여하지 않는다.
- [ ] 세션 복원(`handleLoadSession`)으로 불러온 이전 대화 세션의 메시지 버블에서도 `has-ungrounded`, `is-grounded-verified`, `grounded-verified-badge`, 점수 수준별 클래스가 일관되게 렌더링된다.
- [ ] 스트리밍 진행 중인 최신 AI 메시지의 참고 문서 목록(`currentSources`)에서도 점수 수준별 클래스(`score-high`, `score-medium`, `score-low`)가 실시간으로 반영된다.
- [ ] `public-front/src/components/AiService.css`에 `has-ungrounded`, `is-grounded-verified`, `grounded-verified-badge`, `score-high`, `score-medium`, `score-low` 스타일을 추가한다.
- [ ] 근거 미발견 판정 정상화, 검증 완료 배지 표시, 청크 관련도 수준별 스타일 클래스 바인딩을 검증하는 프론트엔드 단위 테스트를 작성한다.
## 비요구사항 (Out of scope)
- 백엔드 벡터 검색 유사도 산출 공식이나 RAG 비평 프롬프트의 신뢰도 수식 자체를 수정하지 않는다.
- 출처 청크의 정렬 순서나 청크 열람 모달 내 스크롤 이동 기능은 변경하지 않는다 (SPEC-106의 범위).
- 사용자 답변 평가 폼(`AnswerFeedback.tsx`)의 내부 입력 필드나 불만족 사유 태그 목록을 수정하지 않는다.
- 실시간 채팅(`ChatRoom.tsx`)이나 결제 화면(`Payment.tsx`)의 UI를 수정하지 않는다.
## 프론트엔드
`public-front/src/components/AiService.tsx`
- 메시지 버블 상태 판별부 정정:
  - `const isNoSources = !msg.sources || msg.sources.length === 0;`
  - `const isUngrounded = isAi && isNoSources;` (신뢰도 수치와 무관하게 출처 부재 시 즉시 ungrounded 판정)
  - `const isLowConfidence = isAi && !isNoSources && msg.confidence !== undefined && msg.confidence < 0.6;`
  - `const isGroundedVerified = isAi && !isNoSources && !isUngrounded && !hasMissing && !msg.isCancelled && !msg.isError && msg.confidence !== undefined && msg.confidence >= 0.8;`
- 메시지 버블 클래스 확장:
  - `className={... message-bubble${hasMissing ? ' has-missing' : ''}${isLowConfidence ? ' has-low-confidence' : ''}${isUngrounded ? ' has-ungrounded' : ''}${isGroundedVerified ? ' is-grounded-verified' : ''}${turnIndex === highlightedTurnIndex ? ' highlighted-turn' : ''}}`
- 검증 완료 배지 렌더링:
  - `isGroundedVerified && (<div className="grounded-verified-badge" data-testid="grounded-verified-badge">✓ 지식베이스 근거 검증 완료</div>)`
- 청크 및 문서 점수 수준별 클래스 헬퍼 적용:
  - 헬퍼 함수 `getScoreLevelClass(score?: number): string`: `score >= 0.8 ? 'score-high' : score >= 0.6 ? 'score-medium' : 'score-low'`
  - 완료 및 스트리밍 청크 점수 엘리먼트: `<span className={`source-ref-score ${getScoreLevelClass(chunk.score)}`}>관련도 {Math.round(chunk.score * 100)}%</span>`
  - 문서 그룹 최고 관련도 엘리먼트: `<span className={`source-ref-score ${getScoreLevelClass(group.maxScore)}`}>최고 관련도 {Math.round(group.maxScore * 100)}%</span>`
`public-front/src/components/AiService.css`
- `.message-bubble.has-ungrounded`: 근거 부재 답변에 대한 주황/적색 계열 테두리 강조 스타일 (`border: 1px solid rgba(248, 113, 113, 0.4);`)
- `.message-bubble.is-grounded-verified`: 충분한 근거를 가진 답변에 대한 녹색 신뢰 테두리 스타일 (`border: 1px solid rgba(74, 222, 128, 0.35);`)
- `.grounded-verified-badge`: 상단 검증 완료 배지 스타일 (녹색 배경 `rgba(74, 222, 128, 0.15)`, 녹색 텍스트 `#4ade80`, 11px, 둥근 모서리)
- `.source-ref-score.score-high`: 녹색 계열 (`color: #4ade80; background: rgba(74, 222, 128, 0.14);`)
- `.source-ref-score.score-medium`: 황색 계열 (`color: #facc15; background: rgba(250, 204, 21, 0.14);`)
- `.source-ref-score.score-low`: 주황/회색 계열 (`color: #fb923c; background: rgba(251, 146, 60, 0.14);`)
## 수용 기준 (Acceptance Criteria)
- Given 출처(`sources`)가 빈 배열(`[]`)이고 신뢰도(`confidence`)가 0.75인 AI 답변이 있을 때 When 화면에 렌더링되면 Then `isUngrounded`로 판정되어 상단에 `[근거 문서 미발견 (모델 일반 지식 기반 답변)]` 경고 배너가 표시되고 메시지 버블에 `has-ungrounded` 클래스가 적용된다.
- Given 출처(`sources`)가 1건 이상 존재하고 신뢰도가 85%(0.85)이며 확인하지 못한 항목(`missing`)이 빈 배열인 정상 AI 답변이 있을 때 When 화면에 렌더링되면 Then 메시지 버블에 `is-grounded-verified` 클래스가 적용되고 상단에 `[지식베이스 근거 검증 완료]`(`grounded-verified-badge`) 배지가 표시된다.
- Given 답변 참고 문서에 관련도가 각각 0.92, 0.71, 0.45인 3개의 청크가 존재할 때 When 참고 문서 목록을 펼치면 Then 0.92 청크 점수에는 `score-high`, 0.71 청크 점수에는 `score-medium`, 0.45 청크 점수에는 `score-low` 클래스가 각각 독립적으로 부여되어 렌더링된다.
- Given (경계 케이스) 청크 객체에 `score` 속성이 없거나 `undefined`인 레거시 출처 데이터가 주어졌을 때 When 화면에 렌더링되면 Then 자바스크립트 오류 없이 기본 `source-ref-score` 클래스만 렌더링되고 수준별 클래스는 적용되지 않는다.
- Given (경계 케이스) 신뢰도가 85% 이상이지만 생성 중단된 답변(`isCancelled: true`)이거나 오류 답변(`isError: true`)일 때 When 화면에 렌더링되면 Then `is-grounded-verified` 클래스와 `grounded-verified-badge`가 렌더링되지 않는다.
- Given (경계 케이스) 신뢰도가 85% 이상이지만 `displayMissing` 항목이 1개 이상 존재하는 불완전 답변일 때 When 화면에 렌더링되면 Then `grounded-verified-badge`는 노출되지 않고 `[추가 확인 필요]`(`missing-warning-badge`) 배지만 노출된다.
## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1684-1695` (`isUngrounded` 및 상태 판별부)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1702-1715` (`message-bubble` 클래스 및 상단 배지 바인딩부)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1993-1997, 2031-2036` (완료 메시지 참고 문서/청크 점수 렌더링부)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:2165-2169, 2203-2208` (스트리밍 참고 문서/청크 점수 렌더링부)
- 고쳐야 할 자리: `public-front/src/components/AiService.css:1229-1235` (`.source-ref-score` 스타일 정의부)
- 관련 기존 구현: `public-front/src/components/AiService.tsx:1930-1947` (신뢰도 수준별 배지 `confidence-high` 등 스타일링)

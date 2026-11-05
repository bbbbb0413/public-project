---
id: SPEC-093
title: 저신뢰도(Low Confidence) RAG 답변 주의 경고 배너 표시 및 질문 구체화 연동
status: done
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
현재 RAG 답변 시스템은 사용자가 답변을 무조건 맹신하지 않고 근거를 스스로 확인할 수 있도록 신뢰도 점수와 출처 정보를 제공하고 있다. 그러나 신뢰도 점수가 기준치 미만(60% 미만)인 저신뢰도 답변이 생성되었을 때 사용자가 그 위험성을 즉시 인지하고 대처할 수 있는 시각적 안내와 상호작용 장치가 결여되어 있다.
첫째, `public-front/src/components/AiService.tsx:1395-1411` (`AiService`)에서는 검색 출처가 아예 없는 경우(`isNoSources && (confidence === 0 || missing)`)에 한해서만 `[근거 문서 미발견 (모델 일반 지식 기반 답변)]` 경고 배너를 렌더링하고 있다. 하지만 관련도 점수가 낮은 청크만 검색되었거나 에이전틱 비평에서 신뢰도 기준을 충족하지 못한 채 반복 회차가 종료되어 신뢰도가 낮은(`confidence < 0.6`) 답변의 경우, 출처(`msg.sources`)가 존재한다는 이유로 상단 경고 배너가 일절 표시되지 않는다.
둘째, `public-front/src/components/AiService.tsx:1440-1456` (`AiService`) 및 `public-front/src/components/AiService.css:1384-1387` (`.confidence-badge.confidence-low`)를 보면 신뢰도가 60% 미만일 때 메시지 하단 메타 영역의 작은 배지(`confidence-badge confidence-low`) 색상만 빨간색으로 변경될 뿐이다. 사용자가 답변 본문을 읽는 동안 해당 답변이 사실과 다르거나 왜곡되었을 가능성(환각)을 경고하는 인라인 안내가 없어 잘못된 정보를 사실로 오인하기 쉽다.
셋째, `public-front/src/components/AiService.tsx:1280-1285` (`AiService`)에서 누락 정보가 있을 때는 `has-missing` 클래스와 함께 `[추가 확인 필요]` 배지를 부여하고 `public-front/src/components/AiService.tsx:1468-1475` (`handleAskMissingItem`)를 통해 보완 질문 연계를 지원하는 반면, 신뢰도가 60% 미만인 불완전한 답변에는 버블 테두리 경고 스타일이나 질문 수정/구체화 유도 수단이 전혀 없다. 지식베이스의 검색 한계나 질문의 모호함으로 인해 저신뢰도 답변이 나왔을 때 사용자는 무엇을 해야 할지 알 수 없다.
따라서 신뢰도 60% 미만의 저신뢰도 AI 답변에 대해 눈에 띄는 인라인 주의 경고 배너 및 버블 경고 스타일을 부여하고, 기존 질문을 입력창으로 복원하여 구체화할 수 있는 [질문 구체화] 액션을 제공해야 한다.

## 요구사항
- [ ] 신뢰도 수치가 60% 미만(`msg.confidence !== undefined && msg.confidence < 0.6`)인 AI 메시지 버블에 `has-low-confidence` CSS 클래스를 적용하여 시각적 경고 테두리를 표시한다.
- [ ] 신뢰도 60% 미만인 AI 메시지 상단에 `[답변 신뢰도 낮음]` 주의 배지(`low-confidence-badge`)와 인라인 주의 안내 배너(`low-confidence-warning`)를 렌더링한다. (단, 출처가 아예 없어 이미 `ungrounded-warning`이 표시되는 경우는 중복 방지를 위해 문구를 분기하거나 우선순위를 정돈한다.)
- [ ] 저신뢰도 안내 배너 내에 "지식베이스와의 관련도가 낮아 답변이 부정확할 수 있습니다. 질문을 더 구체적으로 작성해 보세요." 안내 문구를 명시한다.
- [ ] 저신뢰도 안내 배너 내에 [질문 구체화] 액션 버튼(`btn-refine-question`)을 제공한다.
- [ ] [질문 구체화] 버튼 클릭 시, 해당 답변의 직전 사용자 질문 내용을 하단 질문 입력창(`question`)에 복원하고 입력창 ref에 포커스를 주어 사용자가 조건을 덧붙여 즉시 재질문할 수 있도록 지원한다.
- [ ] 스트리밍 진행 중(`isStreaming === true`)일 때는 모든 [질문 구체화] 버튼을 비활성화(`disabled`)하여 입력 충돌 및 중복 요청을 차단한다.
- [ ] 과거 세션을 불러왔을 때(`handleLoadSession`)에도 영속화된 신뢰도가 60% 미만인 답변 메시지에 저신뢰도 주의 배너와 [질문 구체화] 버튼이 정상 복원되어 동작해야 한다.
- [ ] 신뢰도가 60% 이상인 정상 답변(`confidence >= 0.6`)에는 저신뢰도 경고 배너 및 `has-low-confidence` 클래스가 노출되지 않아야 한다.

## 비요구사항 (Out of scope)
- 백엔드 신뢰도 산출 공식(`AskUseCase`의 max score, `AgenticAskUseCase`의 비평 신뢰도 점수 산출 로직) 자체의 변경은 포함하지 않는다.
- Qdrant 벡터 검색이나 하이브리드 검색 알고리즘 및 임베딩 모델 변경은 포함하지 않는다.
- 관리자 화면의 LLM 모니터링이나 품질 평가(RAGAS) 집계 로직의 변경은 포함하지 않는다.

## 프론트엔드
`public-front/src/components/AiService.tsx`
- AI 메시지 버블 렌더링부에서 `msg.sender === 'ai'`이고 `msg.confidence !== undefined && msg.confidence < 0.6`인 조건을 판별하여 `has-low-confidence` 클래스 바인딩
- AI 메시지 상단에 `low-confidence-badge` 및 `low-confidence-warning` 배너 렌더링
- [질문 구체화] 버튼 클릭 시 직전 사용자 질문을 찾아 `setQuestion` 및 입력창 포커스를 수행하는 `handleRefineQuestion` 핸들러 연결
- 스트리밍 진행 중 비활성화 처리(`disabled={isStreaming}`)

`public-front/src/components/AiService.css`
- `.message-bubble.has-low-confidence`: 저신뢰도 경고 테두리 스타일 (붉은색/주황색 계열 반투명 테두리)
- `.low-confidence-badge`: `[답변 신뢰도 낮음]` 시각적 뱃지 스타일
- `.low-confidence-warning`: 인라인 경고 배너 및 설명 문구 스타일
- `.btn-refine-question`: 질문 구체화 버튼 스타일 및 hover/disabled 스타일

`public-front/src/components/AiService.test.tsx`
- 신뢰도 60% 미만일 때 저신뢰도 경고 배너 및 `has-low-confidence` 클래스 렌더링 검증 단위 테스트
- [질문 구체화] 버튼 클릭 시 직전 사용자 질문이 입력창에 복원되고 포커스되는지 검증하는 단위 테스트
- 신뢰도 60% 이상일 때 경고 배너가 노출되지 않음을 검증하는 단위 테스트
- 세션 복원 시 저신뢰도 답변의 경고 배너 및 질문 구체화 버튼 복원 검증 단위 테스트

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 질문을 전송하여 RAG 답변 스트리밍이 완료된 상황에서 When 응답 신뢰도가 45%(0.45)로 60% 미만이면 Then AI 메시지 버블에 `has-low-confidence` 클래스가 적용되고 `[답변 신뢰도 낮음]` 배지와 주의 안내 배너가 표시된다.
- Given 저신뢰도 답변 버블에 [질문 구체화] 버튼이 표시된 상태에서 When 사용자가 [질문 구체화] 버튼을 클릭하면 Then 직전 전송했던 사용자 질문 텍스트가 하단 질문 입력창에 그대로 채워지고 입력창에 포커스가 맞춰진다.
- Given 신뢰도가 85%(0.85)인 정상 답변이 완료되거나 복원된 상황에서 When 메시지 버블이 렌더링되면 Then 저신뢰도 경고 배너와 [질문 구체화] 버튼이 표시되지 않고 녹색 신뢰도 85% 배지만 정상 노출된다.
- Given 신뢰도 정보가 누락되었거나 `undefined`인 레거시 세션 메시지를 불러온 상황(경계 케이스)에서 When 세션 상세가 렌더링되면 Then 저신뢰도 경고 배너가 표시되지 않고 오류 없이 기본 텍스트만 정상 렌더링된다.
- Given 답변 스트리밍이 진행 중(`isStreaming === true`)인 상황에서 When 이전 대화 내역의 저신뢰도 답변 버블을 확인하면 Then [질문 구체화] 버튼이 `disabled` 상태로 비활성화되어 중복 클릭이 방지된다.

## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1395-1411` (`AiService`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1440-1456` (`AiService`)
- 고쳐야 할 자리: `public-front/src/components/AiService.css:1384-1387` (`.confidence-badge.confidence-low`)
- 기존 missing 보완 질문 구현: `public-front/src/components/AiService.tsx:1468-1475` (`AiService`)
- 백엔드 신뢰도 계산 및 전달부: `public-python-server/src/ai_service/rag/application/ask_use_case.py:92-113` (`AskUseCase.execute`)

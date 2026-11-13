---
id: SPEC-104
title: RAG 답변 확인하지 못한 항목(missing)의 시스템 상태 오염 분리 및 왜곡된 후속 질문 방지
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품은 올려 둔 문서를 근거로 질문에 답하며, 사용자가 답을 그대로 믿지 않고 스스로 확인하고 검증할 수 있도록 돕는 도구다. 이를 위해 답변 완료 이벤트에 신뢰도 및 누락 정보 전달(SPEC-010), 단발성 질의 신뢰도 산출 및 근거 문서 미참조 경고(SPEC-059), 확인하지 못한 항목(missing) 보완 질문 원클릭 연계(SPEC-075), 비평 파싱 실패 안전한 폴백(SPEC-037), 저신뢰도 답변 주의 경고(SPEC-093), 근거 미참조·저신뢰도 답변의 피드백 연계 개선(SPEC-100) 등이 순차적으로 배송되었다.
그러나 사용자가 RAG 답변의 불완전성을 파악하고 질문을 보완하는 주요 흐름에서, 시스템 내부 처리 상태 문자열이 '확인하지 못한 항목(`missing`)' 목록을 오염시켜 비정상적인 후속 질문이 유발되는 심각한 사용성 및 계약 결함이 존재한다.
첫째, `public-python-server/src/ai_service/rag/application/ask_use_case.py:108-113` (`AskUseCase.execute`)을 보면 지식베이스 검색 결과 청크가 0건일 때 `missing = ["관련 지식베이스 문서를 찾지 못함"]`을 반환한다. 또한 `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:186-192` (`AgenticAskUseCase.execute`)에서는 1회차 예산 소진 시 `last_missing = ["답변 검증 예산 소진으로 상세 비평 생략"]` 또는 `["관련 지식베이스 문서를 찾지 못함"]`을 반환하고, `public-python-server/src/ai_service/rag/application/critique_generator_service.py:151-158` (`CritiqueGeneratorService._fallback_critique`)에서도 비평 파싱 실패 시 `missing=["비평 파싱 실패"]`를 주입한다. `missing` 필드는 본래 사용자의 질문 내용 중 답변에서 검증되지 못한 "사실적 지식 주제(content items)"를 담아야 함에도, 백엔드의 여러 경로에서 시스템 내부 상태 및 에러 메시지를 `missing` 배열에 문자열로 담아 클라이언트로 방출하고 있다.
둘째, `public-front/src/components/AiService.tsx:1743-1763` (`AiService`)을 보면 `msg.missing`이 존재하면 화면 하단에 `확인하지 못한 항목:` 목록을 렌더링하고, 각 항목 우측에 `[질문하기]`(`btn-ask-missing`) 액션 버튼을 노출한다. 이로 인해 근거 문서를 찾지 못해 상단에 이미 `[근거 문서 미발견 (모델 일반 지식 기반 답변)]` 경고 배너(`ungrounded-warning`, SPEC-059/SPEC-100)가 표시되었음에도 불구하고, 그 하단에 다시 "확인하지 못한 항목: 관련 지식베이스 문서를 찾지 못함 [질문하기]"가 중복 표시되어 화면 정보를 어지럽힌다.
셋째, 가장 치명적인 결함은 `public-front/src/components/AiService.tsx:997-1000` (`handleAskMissingItem`)에서 발생한다. 사용자가 확인하지 못한 항목 우측의 `[질문하기]` 버튼을 누르면 `setQuestion(`"${itemText.trim()}"에 대해 자세히 설명해 줘`)`가 실행된다. 이로 인해 시스템 상태 문자열이 템플릿에 그대로 결합되어 `""관련 지식베이스 문서를 찾지 못함"에 대해 자세히 설명해 줘"`, `""비평 파싱 실패"에 대해 자세히 설명해 줘"`, `""답변 검증 예산 소진으로 상세 비평 생략"에 대해 자세히 설명해 줘"`라는 황당하고 왜곡된 질문이 사용자 입력창에 채워진다. 사용자가 이를 전송하면 백엔드 RAG 파이프라인에 내부 에러 문구가 질의어로 유입되어 의미 없는 답변이 생성되며, 이전 작업(SPEC-075)으로 배송된 보완 질문 연계 기능의 성과를 심각하게 훼손한다.
넷째, `public-front/src/components/AiService.tsx:1482-1490` (`AiService`)의 `isUngrounded` 조건(`isNoSources && (msg.confidence === 0 || (msg.missing && msg.missing.length > 0))`) 역시 `missing`에 시스템 메시지가 주입되는 것에 의존하고 있어, 근거가 없어 모델의 일반 지식으로 답변한 경우와 모델 비평이 정상 수행되어 세부 누락 항목이 발생한 경우가 의미론적으로 혼재되어 있다.
따라서 백엔드에서 시스템 상태/오류 안내성 문자열을 `missing`에서 배제하고 실제 누락된 지식 항목만 담도록 정돈하며, 프론트엔드에서도 시스템 상태성 문구를 필터링하고 `isUngrounded` 상태에서는 `missing-info` 노출을 억제하여 올바른 질문 보완 흐름을 복원해야 한다. 이 개선은 "답변을 믿을 수 있게 만든다 — 답이 틀렸거나 근거가 부족할 때 사용자가 그것을 알아채고 올바르게 후속 조치할 수 있는가"라는 이번 분기 제품 방향에 직결된다.

## 요구사항
- [ ] `public-python-server/src/ai_service/rag/application/ask_use_case.py`의 `AskUseCase.execute`에서 검색된 지식 청크가 없을 때(`not chunks`), `missing`에 시스템 상태 메시지 대신 빈 배열(`missing = []`)을 전달한다.
- [ ] `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py`의 `AgenticAskUseCase.execute`에서 예산 소진으로 비평을 생략하거나 검색 청크가 없을 때, `last_missing`에 시스템 사유 문자열 대신 빈 배열(`last_missing = []`)을 할당한다.
- [ ] `public-python-server/src/ai_service/rag/application/critique_generator_service.py`의 `CritiqueGeneratorService._fallback_critique`에서 파싱 실패 폴백 시 `missing`에 시스템 에러 문자열 대신 빈 배열(`missing=[]`)을 설정한다.
- [ ] `public-front/src/components/AiService.tsx`에서 알려진 시스템 상태성 메시지(`"관련 지식베이스 문서를 찾지 못함"`, `"비평 파싱 실패"`, `"비평 응답 분석 실패"`, `"답변 검증 예산 소진으로 상세 비평 생략"`)를 판별하여 `missing-info` 목록 렌더링 대상에서 제외하는 필터 로직을 적용한다.
- [ ] `public-front/src/components/AiService.tsx`에서 근거 문서 미발견 상태(`isUngrounded === true`)인 AI 메시지 버블에서는 `missing-info` 영역 및 `[추가 확인 필요]`(`has-missing`) 배지를 렌더링하지 않고, 상단 `ungrounded-warning` 배너와 [질문 구체화] 액션만 제공하도록 정리한다.
- [ ] `public-front/src/components/AiService.tsx`의 `handleAskMissingItem`에서 전달받은 `itemText`가 비어있거나 시스템 상태성 메시지인 경우 질문 입력을 차단하고 안전하게 무시한다.
- [ ] 세션 복원(`handleLoadSession`)으로 불러온 과거 대화 턴에 시스템 상태성 문자열이 `missing`으로 저장되어 있더라도, 프론트엔드 필터링을 통해 `missing-info`에 왜곡된 항목이 노출되지 않고 `[질문하기]` 오동작이 발생하지 않아야 한다.
- [ ] 백엔드 단순/에이전틱 RAG 및 비평기 폴백 시의 빈 missing 처리와 프론트엔드 시스템 메시지 필터링 및 질문 템플릿 생성 방어 동작에 대한 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- `CritiqueGeneratorService`의 비평 프롬프트 템플릿 내용이나 LLM 모델(gpt-4o-mini)을 변경하지 않는다.
- `JobEventPublisher`의 Redis Stream 이벤트 구조(`confidence`, `missing` 페이로드 키 규격)를 변경하지 않는다.
- MongoDB 세션 컬렉션의 스키마 구조나 `ConversationSession`의 `missing` 필드 타입을 변경하지 않는다.
- 실시간 채팅(`ChatRoom.tsx`)이나 결제 내역 컴포넌트의 동작을 수정하지 않는다.

## 백엔드
`public-python-server/src/ai_service/rag/application/ask_use_case.py`
- `AskUseCase.execute` (라인 108~113):
  - `chunks`가 없을 때 `missing = []`로 설정하여 시스템 메시지 대신 빈 배열을 `__CONFIDENCE` 페이로드로 방출한다.
  - 검색된 청크가 없으므로 신뢰도는 0.0을 유지하고(`confidence = 0.0`), 세부 누락 항목은 존재하지 않음을 명시한다.

`public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py`
- `AgenticAskUseCase.execute` (라인 186~192):
  - 예산 소진으로 비평을 수행하지 못했을 때 `last_missing = []`로 설정한다. (chunks 유무에 상관없이 시스템 사유 문자열을 missing에 주입하지 않는다)

`public-python-server/src/ai_service/rag/application/critique_generator_service.py`
- `CritiqueGeneratorService._fallback_critique` (라인 151~158):
  - 파싱 실패 폴백 시 `missing=[]`로 빈 리스트를 반환하여 에러 메시지가 도메인 missing 데이터로 오염되지 않도록 한다.

## 프론트엔드
`public-front/src/components/AiService.tsx`
- 시스템 상태성 메시지 필터링 헬퍼 정의:
  - 시스템 상태 메시지 목록(`SYSTEM_STATUS_PATTERNS = ['관련 지식베이스 문서를 찾지 못함', '비평 파싱 실패', '비평 응답 분석 실패', '답변 검증 예산 소진으로 상세 비평 생략']`)을 정의한다.
  - `getSanitizedMissing(missing?: string[])`: `missing` 배열에서 시스템 상태 패턴을 포함하는 문자열을 제외한 순수 질문 누락 항목만 반환한다.
- `isUngrounded` 판정부 정돈 (라인 1482~1490):
  - `isUngrounded`는 `isAi && isNoSources && (msg.confidence === 0 || !msg.confidence)` 형태로 정리하여 시스템 메시지 존재 여부에 의존하지 않도록 한다.
- `missing-info` 렌더링 조건 및 목록 개선 (라인 1743~1763):
  - `const displayMissing = getSanitizedMissing(msg.missing);`를 추출한다.
  - 근거 미발견 답변(`isUngrounded`)이 아니고 정제된 누락 항목이 1개 이상 존재할 때(`!isUngrounded && displayMissing.length > 0`)만 `missing-info` 영역 및 `has-missing` 스타일을 적용한다.
  - 렌더링 목록에는 `displayMissing`의 항목들을 노출한다.
- `handleAskMissingItem` 보완 (라인 997~1000):
  - `itemText`가 시스템 상태 패턴에 매칭되거나 공백인 경우 입력을 중단하고 함수를 조기 종료한다.

## 수용 기준 (Acceptance Criteria)
- Given 지식베이스에 관련 문서가 없어 검색 결과가 0건인 질문이 전송되었을 때
  When 답변 생성이 완료되면
  Then 완료 이벤트의 missing 배열은 빈 배열([])이어야 하고, 프론트엔드 화면 상단에 [근거 문서 미발견] 배너만 표시되며 하단 missing-info 영역 및 [질문하기] 버튼은 노출되지 않아야 한다.
- Given 에이전틱 RAG 1회차에서 토큰 또는 시간 예산이 소진되어 비평이 생략되었을 때
  When 스트리밍이 종료되고 세션에 턴이 저장될 때
  Then missing 필드는 빈 배열([])이어야 하며, 화면의 확인하지 못한 항목에 "답변 검증 예산 소진으로 상세 비평 생략"이 노출되지 않아야 한다.
- Given 비평 LLM 응답이 유효한 JSON 형식이 아니어서 파싱에 실패했을 때
  When 비평 결과 폴백 객체가 생성될 때
  Then confidence는 0.0, answered는 False, missing은 빈 배열([])이어야 한다.
- Given 과거 레거시 세션 데이터의 missing에 "관련 지식베이스 문서를 찾지 못함" 또는 "비평 파싱 실패" 문자열이 저장되어 있을 때
  When 사용자가 해당 대화 세션을 불러왔을 때
  Then 프론트엔드 필터링을 통해 해당 시스템 상태 메시지가 missing-info 목록에 렌더링되지 않아야 한다.
- Given 유효한 누락 항목(예: "제품 환불 규정")이 missing에 포함된 답변 버블에서
  When 사용자가 해당 항목의 [질문하기] 버튼을 클릭했을 때
  Then 하단 질문 입력창에 `"제품 환불 규정"에 대해 자세히 설명해 줘` 형식으로 질문이 복원되고 입력창에 포커스가 이동해야 한다.
- Given 확인하지 못한 항목 텍스트가 빈 문자열이거나 공백만으로 이루어진 경우
  When 사용자가 비정상적인 방법으로 [질문하기]를 호출했을 때
  Then 질문 입력창 내용이 갱신되지 않고 기존 입력 상태가 보존되어야 한다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/ask_use_case.py:108-113` (`AskUseCase.execute`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:186-192` (`AgenticAskUseCase.execute`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/critique_generator_service.py:151-158` (`CritiqueGeneratorService._fallback_critique`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:997-1000` (`handleAskMissingItem`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:1743-1763` (`AiService`)
- 관련 기존 구현: `public-front/src/components/AiService.tsx:1482-1490` (`AiService`)

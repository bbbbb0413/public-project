---
id: SPEC-075
title: RAG 답변의 확인하지 못한 항목(missing) 보완 질문 원클릭 연계 및 불완전 답변 주의 안내
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
이 제품의 이번 분기 핵심 방향은 "답변을 믿을 수 있게 만든다"이며, 사용자가 스스로 근거의 충분 여부를 확인하고 불완전하거나 틀린 답변을 시스템을 통해 바로잡을 수 있어야 한다.
현재 백엔드 에이전틱 RAG 비평 루프(`CritiqueGeneratorService`)와 완료 이벤트 발행부 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:196-203` (`AskRequestedConsumer._process`) 및 세션 영속화 도메인 `public-python-server/src/ai_service/rag/schemas.py:162-179` (`ConversationSession.append_turn`)에서는 답변 생성 중 지식베이스 문서에서 검증하지 못한 결손 항목(`missing`)을 체계적으로 수집하여 프론트엔드로 전달하고 있다.
하지만 프론트엔드 대화 화면 `public-front/src/components/AiService.tsx:1121-1132` (`AiService`)에서는 수신된 `msg.missing` 항목들을 단순한 정적 텍스트 불릿 목록(`ul/li`)으로만 렌더링하고 있어 다음과 같은 치명적인 한계가 존재한다:
- 확인하지 못한 항목(missing)이 다수 존재하는 답변은 환각이나 오답 가능성이 높은 '불완전 답변'임에도 불구하고, 메시지 버블 자체는 정상 완성 답변과 동일한 스타일로 렌더링되어 사용자가 답변을 신뢰할 수 있는 완전한 정보로 오인할 위험이 크다.
- 사용자가 확인하지 못한 핵심 항목(예: 특정 프로토콜 규격, 예외 처리 정책 등)에 대해 추가 조사를 이어가고 싶어도, 긴 전문 용어를 하단 입력창 `public-front/src/components/AiService.tsx:1284-1294` (`AiService`)에 일일이 수동 타이핑해야 하는 단절된 사용성을 겪는다.
- `public-front/src/components/AiService.css:1081-1107` (`.missing-info`)에 정의된 확인하지 못한 항목 영역에 액션 인터랙션이 전혀 없어, 에이전트가 찾아낸 소중한 비평 메타데이터가 실제 후속 질의응답 흐름으로 이어지지 못하고 방치된다.
- 따라서 누락 항목이 포함된 답변에 시각적 주의 배지를 제공하여 불완전성을 명확히 알리고, 각 항목을 원클릭으로 질문창에 채워 보완 질문을 던질 수 있는 연계 인터랙션을 구축해야 한다.
## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 `AiService` 컴포넌트에 확인하지 못한 항목 내용을 보완 질문 템플릿으로 변환하여 질문 입력창에 채우고 포커스하는 핸들러(`handleAskMissingItem`)를 추가한다.
- [ ] `handleAskMissingItem` 실행 시, 해당 항목 텍스트(`item`)를 활용해 `"${item}"에 대해 자세히 설명해 줘` 형식의 텍스트를 질문 상태(`question`)에 바인딩하고, 입력창 ref(`chatInputRef`)에 포커스를 맞춘다.
- [ ] `AiService.tsx`의 `missing-info` 영역 내 각 항목(`missing-item`) 우측에 `btn-ask-missing` 클래스를 가진 "[질문하기]" 액션 버튼을 렌더링한다.
- [ ] `msg.missing`이 존재하고 1개 이상의 항목을 가질 때(`msg.missing.length > 0`), 해당 AI 메시지 버블(`message-bubble`)에 `has-missing` CSS 클래스를 적용하고 상단에 `[추가 확인 필요]` 주의 배지(`missing-warning-badge`)를 렌더링한다.
- [ ] 답변 스트리밍 진행 중(`isStreaming === true`)일 때는 중복 요청 및 입력 충돌을 방지하기 위해 모든 "[질문하기]"(`btn-ask-missing`) 버튼을 비활성화(`disabled`)한다.
- [ ] 세션 복원(`handleLoadSession`) 시에도 저장된 대화의 `missing` 항목들에 대해 불완전 답변 주의 배지와 "[질문하기]" 버튼이 정상 렌더링되고 클릭 시 동작해야 한다.
- [ ] `missing` 항목이 없거나 빈 배열(`[]`)인 정상 답변의 경우, `has-missing` 클래스, `missing-warning-badge`, `missing-info`가 노출되지 않고 기존의 일반 메시지 버블로 렌더링된다.
## 비요구사항 (Out of scope)
- 백엔드 LLM 비평(Critique) 프롬프트 및 누락 사유 분석 수식 변경
- MongoDB 세션 영속화 모델(`ConversationTurn`, `TurnRecord`)의 스키마 변경 (이미 `missing: list[str]` 필드 지원 중)
- 버튼 클릭 시 사용자의 검토 없이 즉시 API를 호출하는 전자동 백그라운드 전송 (사용자가 입력창에 채워진 질문을 확인하고 수정할 수 있는 반자동 흐름 준수)
- 채팅 입력창 외의 음성 입력이나 별도 모달 등 신규 입력 컴포넌트 추가
## 프론트엔드
- 대상 컴포넌트: `public-front/src/components/AiService.tsx`
- 대상 스타일: `public-front/src/components/AiService.css`
- 단위 테스트: `public-front/src/components/AiService.test.tsx`
- 주요 구현 내용:
  - `handleAskMissingItem` 핸들러 정의:
    - 매개변수로 누락 항목 문자열(`itemText: string`)을 수신
    - `setQuestion(`"${itemText.trim()}"에 대해 자세히 설명해 줘`)` 호출
    - `chatInputRef.current?.focus()`를 통해 하단 입력창에 즉각적인 포커스 유도
  - 메시지 버블 상단 주의 배지 표시:
    - `msg.missing && msg.missing.length > 0` 조건 만족 시 버블 상단에 `<div className="missing-warning-badge" data-testid="missing-warning-badge">추가 확인 필요</div>` 렌더링
  - 누락 항목 행 액션 버튼 렌더링:
    - `missing-list` 내부의 각 `missing-item`을 `span`과 `button`의 flex 구조로 개선
    - `<button type="button" className="btn-ask-missing" onClick={() => handleAskMissingItem(item)} disabled={isStreaming} aria-label={`${item}에 대해 질문하기`}>질문하기</button>`
  - 스타일 추가:
    - `.message-bubble.has-missing`: 황색/앰버 계열의 미묘한 테두리(`border: 1px solid rgba(245, 158, 11, 0.4)`)
    - `.missing-warning-badge`: 작은 경고 칩 스타일 (황색 배경, 어두운 텍스트 또는 앰버 포인트)
    - `.btn-ask-missing`: 항목 텍스트와 시각적으로 조화되는 컴팩트한 인라인 버튼 스타일
## 수용 기준 (Acceptance Criteria)
- Given 확인하지 못한 항목("JWT 만료 시 갱신 흐름")이 1건 포함된 AI 답변이 화면에 표시된 상태에서
  When 사용자가 해당 항목 우측의 "[질문하기]" 버튼을 클릭하면
  Then 하단 질문 입력창에 `"\"JWT 만료 시 갱신 흐름\"에 대해 자세히 설명해 줘"`가 입력되고 입력창에 포커스가 맞춰져야 한다.
- Given `missing` 항목이 포함된 AI 답변 메시지가 렌더링될 때
  When 사용자가 화면의 메시지 버블을 확인하면
  Then 메시지 버블에 `has-missing` 클래스가 적용되어 있고 버블 상단에 `[추가 확인 필요]` 배지가 표시되어야 한다.
- Given 답변 스트리밍이 진행 중인 상태(`isStreaming === true`)에서
  When 화면에 렌더링된 이전 또는 현재 답변의 `missing` 항목 영역을 확인하면
  Then 모든 "[질문하기]"(`btn-ask-missing`) 버튼이 `disabled` 상태여야 한다.
- Given `missing` 필드가 없거나 빈 배열(`[]`)인 정상 AI 답변의 경우 (경계 조건)
  When 메시지 버블이 렌더링되면
  Then `has-missing` 클래스와 `[추가 확인 필요]` 배지 및 `missing-info` 영역이 일절 표시되지 않아야 한다.
- Given 확인하지 못한 항목이 포함된 대화 세션에서
  When 사용자가 사이드바에서 해당 세션을 다시 불러오면(`handleLoadSession`)
  Then 복원된 메시지 버블에 `has-missing` 클래스와 `[추가 확인 필요]` 배지가 유지되고, 각 항목의 "[질문하기]" 버튼 클릭 시 질문 템플릿이 입력창에 정상 채워져야 한다.
## 참고
- 고쳐야 할 자리 (프론트 메시지 렌더링): `public-front/src/components/AiService.tsx:1121-1132` (`AiService`)
- 고쳐야 할 자리 (채팅 입력창 및 ref): `public-front/src/components/AiService.tsx:1284-1294` (`AiService`)
- 고쳐야 할 자리 (누락 항목 스타일): `public-front/src/components/AiService.css:1081-1107` (`.missing-info`)
- 관련 정의 (메시지 인터페이스): `public-front/src/components/AiService.tsx:50-55` (`ChatMessage`)
- 백엔드 연관 발행부 (done 이벤트 missing 전달): `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:196-203` (`AskRequestedConsumer._process`)
- 백엔드 도메인 영속화 (turn missing 저장): `public-python-server/src/ai_service/rag/schemas.py:162-179` (`ConversationSession.append_turn`)

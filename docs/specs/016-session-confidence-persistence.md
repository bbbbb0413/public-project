---
id: SPEC-016
title: 대화 세션 영속화 시 신뢰도 메타데이터 저장 누락 수정 및 신뢰도 수준별 배지 시각화
status: draft
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
이전 작업(SPEC-011)에서 대화 세션 복원 시 출처 및 신뢰도/누락 정보 복원 인터페이스를 구현하였으나, `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:166-169` 에서 실시간 답변 스트리밍 완료 후 `session.append_turn` 호출 시 `sources` 만 전달하고 `confidence` 와 `missing` 인자를 전달하지 않아 세션 DB에 신뢰도 정보가 `None` 으로 저장되어 영속화되지 않는 결함이 발생하고 있다.
`public-python-server/src/ai_service/rag/schemas.py:162-169` 의 `append_turn` 메서드는 `confidence` 와 `missing` 인자를 받도록 준비되어 있으나, 호출부의 누락으로 인해 세션을 다시 열었을 때 신뢰도 배지와 확인하지 못한 항목이 복원되지 않고 유실된다.
또한 프론트엔드 `public-front/src/components/AiService.tsx:640-644` 및 `public-front/src/components/AiService.css:762-790` 에서는 신뢰도 배지가 수치(%)와 상관없이 고정된 녹색(`#34d399`) 단일 스타일로만 렌더링된다. 이로 인해 신뢰도가 매우 낮은 답변(예: 30%)도 신뢰도가 높은 답변과 동일하게 긍정적인 초록색으로 표시되어 사용자가 부정확하거나 근거가 부족한 답변을 판별하기 어려운 문제가 있다.

## 요구사항
- [ ] `public-python-server` 의 `ask_requested_consumer.py` 에서 RAG 답변 완료 후 세션에 턴을 추가할 때 수집된 `last_confidence` 와 `last_missing` 을 `session.append_turn` 에 함께 전달하여 영속화한다.
- [ ] 비에이전틱 단발성 질의이거나 신뢰도 정보가 없는 경우(`last_confidence` 또는 `last_missing` 이 `None`), `None` 값이 안전하게 전달되어 오류 없이 세션이 저장된다.
- [ ] 프론트엔드 `public-front/src/components/AiService.tsx` 에서 신뢰도 수치에 따라 신뢰도 배지에 수준별 CSS 클래스(`high`, `medium`, `low`)를 부여한다. (예: 80% 이상 `high`, 60% 이상 80% 미만 `medium`, 60% 미만 `low`)
- [ ] 프론트엔드 `public-front/src/components/AiService.css` 에 신뢰도 수준별 색상 테마(높음: 녹색, 보통: 황색/노랑, 낮음: 주황/적색) 스타일을 추가한다.
- [ ] 세션 복원 시 저장된 신뢰도와 누락 정보가 각 AI 메시지 버블에 올바른 수준별 배지 색상과 함께 복원되어 렌더링된다.
- [ ] 백엔드 컨슈머 및 프론트엔드 신뢰도 배지 수준별 렌더링에 대한 단위/통합 테스트를 작성한다.

## 비요구사항 (Out of scope)
- RAG 추론/비평 알고리즘의 신뢰도 계산 산식 변경이나 임계값 조정
- 세션 외 별도의 사용자 피드백(좋아요/싫어요) 수집 API 구축
- 결제 및 인증 등 RAG 외 도메인의 상태 변경

## 백엔드
`public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py` 의 질의응답 처리 완료 지점에서 `session.append_turn(message.question, full_response, sources=sources, confidence=last_confidence, missing=last_missing)` 으로 인자를 보강하여 세션 저장소에 신뢰도 및 누락 정보를 온전하게 영속화한다.

## 프론트엔드
`public-front/src/components/AiService.tsx` 및 `public-front/src/components/AiService.css` 에서 AI 메시지 버블의 신뢰도 배지(`confidence-badge`) 렌더링 로직을 수정하여, `msg.confidence` 값의 범위에 따라 적절한 클래스를 적용하고 사용자가 답변의 신뢰 수준을 즉각 시각적으로 식별할 수 있도록 한다.

## 수용 기준 (Acceptance Criteria)
- Given 에이전틱 RAG 질의가 실행되어 신뢰도가 85%(0.85)로 계산되고 세션이 존재하는 상황에서 When 답변 스트리밍이 완료되고 세션이 저장되면 Then 세션 저장소의 마지막 턴 레코드에 `confidence: 0.85` 가 정상 저장된다.
- Given 세션에 `confidence: 0.45` (45%)인 이전 대화가 저장되어 있을 때 When 사용자가 해당 세션을 불러오면 Then 해당 AI 답변 버블의 신뢰도 배지에 `low` 클래스가 적용되어 45%가 낮은 신뢰도 색상(주황/적색 계열)으로 표시된다.
- Given `confidence` 가 `undefined` 또는 `None` 인 레거시 세션 턴을 불러올 때 When 화면에 렌더링되면 Then 신뢰도 배지가 렌더링되지 않고 텍스트 본문만 정상 노출된다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:166-169` (`append_turn` 호출)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:640-644` (`confidence-badge` 렌더링)
- 고쳐야 할 자리: `public-front/src/components/AiService.css:762-790`
- 관련 세션 모델 메서드: `public-python-server/src/ai_service/rag/schemas.py:162-169` (`ConversationSession.append_turn`)

> 2026-08-27 의 기능별 모듈 구조 전환으로 세션 모델이 `rag/domain/model/conversation_session.py` 에서 `public-python-server/src/ai_service/rag/schemas.py:115` 로 옮겨졌다. 결함 자체는 그대로 남아 있다 — 호출부가 여전히 `sources` 만 넘긴다.

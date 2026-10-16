---
id: SPEC-076
title: 대화 세션 턴 영속화 시 에이전틱 RAG 반복 검증 횟수(iterations) 저장 및 복원
status: done
targets: [python-server]
stages: [backend, qa]
priority: high
---
## 배경 / 문제
SPEC-044를 통해 에이전틱 RAG 질의 완료 시 총 추론 반복 회차(`iterations`)를 스트리밍 완료 이벤트에 담아 전달하고 화면에 `[N회 반복 검증]` 배지를 표시하도록 개선했다. 프론트엔드 역시 대화 세션을 다시 불러올 때 배지가 유지되도록 `public-front/src/api/ai.ts:298-306` (`SessionTurn`)에 `iterations?: number` 타입을 정의하고, `public-front/src/components/AiService.tsx:329-342` (`handleLoadSession`)에서 `t.iterations`를 매핑하여 `public-front/src/components/AiService.tsx:1150-1157` (`AiService`)에서 배지를 렌더링하도록 구현되어 있다.

그러나 정작 Python 백엔드에서는 스트리밍 완료 직후 세션을 저장할 때 `iterations` 값을 누락하고 있다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:206-215` (`AskRequestedConsumer._process`)에서는 `last_iteration`을 `done` 이벤트 페이로드(`done_data["iterations"]`)에 담아 발행하지만, 바로 직전 줄인 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:197-205` (`AskRequestedConsumer._process`)의 `session.append_turn` 호출에는 `iterations`를 전달하지 않는다.
- 도메인 모델 및 응답 스키마인 `public-python-server/src/ai_service/rag/schemas.py:30-37` (`TurnValue`), `public-python-server/src/ai_service/rag/schemas.py:48-64` (`ConversationTurn.of_assistant`), `public-python-server/src/ai_service/rag/schemas.py:96-103` (`TurnRecord`), `public-python-server/src/ai_service/rag/schemas.py:139-187` (`ConversationSession.restore`, `ConversationSession.append_turn`), 그리고 세션 상세 조회 입구인 `public-python-server/src/ai_service/rag/router.py:29-44` (`get_session`)가 반환하는 `public-python-server/src/ai_service/rag/schemas.py:337-356` (`SessionTurnOut`) 모두에 `iterations` 필드가 빠져 있다.
- MongoDB 영속화 계층인 `public-python-server/src/ai_service/rag/repository.py:93-139` (`ConversationSessionRepository._to_domain`, `ConversationSessionRepository._to_record`)에서도 턴의 `iterations`를 저장하거나 복원하지 않는다.

이로 인해 사용자가 복합 질문을 던져 에이전틱 RAG가 2회 이상 반복 검증을 수행하더라도, 다른 대화를 열었다가 돌아오거나 화면을 새로고침하여 세션을 복원하면 `[N회 반복 검증]` 배지가 사라진다. 이는 이번 분기 핵심 방향인 "답변을 믿을 수 있게 만든다"에서 이미 배송한 SPEC-044의 추론 검증 요약이 세션 복원 시 유실되는 결함을 바로잡는 작업이다.

## 요구사항
- [ ] `public-python-server/src/ai_service/rag/schemas.py`의 `TurnValue`, `TurnRecord` 데이터클래스에 `iterations: int | None = None` 선택적 필드를 추가한다.
- [ ] `public-python-server/src/ai_service/rag/schemas.py`의 `ConversationTurn`에 `iterations` 프로퍼티(`int | None`)를 추가하고, `ConversationTurn.of_assistant` 및 `ConversationSession.restore`, `ConversationSession.append_turn`이 `iterations: int | None = None`을 받아 어시스턴트 턴에 전달하도록 수정한다.
- [ ] `public-python-server/src/ai_service/rag/schemas.py`의 `SessionTurnOut` 모델에 `iterations: int | None = None` 필드를 추가하고 `SessionTurnOut.from_domain`에서 `turn.iterations`를 매핑한다.
- [ ] `public-python-server/src/ai_service/rag/repository.py`의 `ConversationSessionRepository._to_record`에서 `t.iterations is not None`일 때 `turn_dict["iterations"] = t.iterations`를 저장하고, `_to_domain`에서 `t.get("iterations")`로 복원한다.
- [ ] `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`의 `AskRequestedConsumer._process`에서 `session.append_turn` 호출 시 `iterations=last_iteration`을 함께 전달한다.
- [ ] `iterations` 필드가 없는 기존 MongoDB 세션 문서(레거시 데이터)나 단순 질의(`last_iteration is None`) 턴을 조회·저장할 때도 오류 없이 `iterations=None`으로 안전하게 처리되어야 한다.
- [ ] 도메인 스키마, 리포지토리 매핑, 컨슈머 세션 저장 및 세션 상세 조회 API에 대한 단위·통합 테스트를 추가한다.

## 비요구사항 (Out of scope)
- 프론트엔드(`public-front`) 및 NestJS 게이트웨이(`public-server`) 코드 수정은 포함하지 않는다 (`public-front/src/api/ai.ts`와 `AiService.tsx`는 이미 `iterations` 필드를 수신해 렌더링하도록 구현되어 있으며, 게이트웨이는 세션 상세 응답 JSON을 그대로 투과한다).
- 에이전틱 RAG의 반복 검증 루프 로직(`AgenticAskUseCase`)이나 복잡도 라우팅 기준 자체는 변경하지 않는다.
- 기존 MongoDB에 저장된 과거 세션 문서에 대한 일괄 데이터 마이그레이션은 수행하지 않는다.

## 백엔드
- `public-python-server/src/ai_service/rag/schemas.py`:
  - `TurnValue`(`schemas.py:30-37`) 및 `TurnRecord`(`schemas.py:96-103`)에 `iterations: int | None = None` 필드를 추가한다.
  - `ConversationTurn.of_assistant`(`schemas.py:48-64`)에 `iterations: int | None = None` 매개변수를 추가하고 `TurnValue` 생성 시 전달하며, `ConversationTurn.iterations` 프로퍼티를 추가한다.
  - `ConversationSession.restore`(`schemas.py:139-160`)에서 `TurnValue` 복원 시 `iterations=t.iterations`를 전달하고, `ConversationSession.append_turn`(`schemas.py:162-187`)에 `iterations: int | None = None` 매개변수를 추가하여 `ConversationTurn.of_assistant`로 넘긴다.
  - `SessionTurnOut`(`schemas.py:337-356`)에 `iterations: int | None = None`을 선언하고 `from_domain`에서 `iterations=turn.iterations`를 매핑한다.
- `public-python-server/src/ai_service/rag/repository.py`:
  - `ConversationSessionRepository._to_domain`(`repository.py:93-113`)의 `TurnRecord` 생성자에 `iterations=t.get("iterations")`를 추가한다.
  - `ConversationSessionRepository._to_record`(`repository.py:116-139`)에서 `if t.iterations is not None: turn_dict["iterations"] = t.iterations`를 추가하여 불필요한 `null` 저장을 피하고 하위 호환성을 유지한다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`:
  - `AskRequestedConsumer._process`(`ask_requested_consumer.py:197-203`)의 `session.append_turn` 호출 인자에 `iterations=last_iteration`을 추가한다.

## 수용 기준 (Acceptance Criteria)
- Given 에이전틱 RAG 경로(`complexity == "complex"`)에서 2회 반복 검증(`last_iteration = 2`)을 거쳐 답변 생성이 완료되었을 때, When `AskRequestedConsumer._process`가 대화 세션에 턴을 추가하고 저장하면, Then 저장된 어시스턴트 턴의 `iterations` 값은 `2`이며 `GET /rag/sessions/{session_id}` 응답의 해당 어시스턴트 턴 JSON에 `"iterations": 2`가 포함된다.
- Given 단발성 단순 RAG 질의(`complexity != "complex"`)로 `last_iteration`이 `None`인 상태에서 답변 생성이 완료되었을 때, When 세션이 저장되고 조회되면, Then 사용자 턴과 어시스턴트 턴의 `iterations`는 `None`(`null`)으로 반환되며 MongoDB 문서에는 `iterations` 키가 강제되지 않는다.
- Given `iterations` 필드가 저장되기 전에 생성된 레거시 MongoDB 세션 문서를 불러올 때(경계 케이스), When `ConversationSessionRepository._to_domain` 및 `GET /rag/sessions/{session_id}`가 실행되면, Then `KeyError` 없이 `iterations: null`로 복원되어 기존 대화 내용과 출처·신뢰도가 정상 반환된다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:197-205` (`AskRequestedConsumer._process`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/schemas.py:30-37` (`TurnValue`), `public-python-server/src/ai_service/rag/schemas.py:162-187` (`ConversationSession.append_turn`), `public-python-server/src/ai_service/rag/schemas.py:337-356` (`SessionTurnOut`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/repository.py:93-139` (`ConversationSessionRepository._to_domain`, `ConversationSessionRepository._to_record`)
- 관련 정의나 기존 구현: `public-python-server/src/ai_service/rag/router.py:29-44` (`get_session`), `public-front/src/api/ai.ts:298-306` (`SessionTurn`), `public-front/src/components/AiService.tsx:329-342` (`handleLoadSession`), `public-front/src/components/AiService.tsx:1150-1157` (`AiService`)

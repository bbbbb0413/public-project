---
id: SPEC-088
title: 에이전틱 RAG 다회차 정제 쿼리(next_query) 추적 및 진행 이벤트·세션 영속화 연동
status: done
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
현재 에이전틱 RAG 질의응답 시스템은 초기 검색 결과의 답변 충실도가 부족할 경우 비평(`CritiqueGeneratorService`)을 거쳐 보완 검색을 위한 정제 쿼리(`next_query`)를 생성하고 2회차 이상의 재검색(`query_refiner.refine`)을 수행한다.
그러나 `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:193-210` (`AgenticAskUseCase.execute`)에서는 `_query_refiner.refine`을 통해 산출된 정제 쿼리를 오직 내부 `current_query` 변수에만 할당하여 검색 루프에 소비할 뿐, 진행 콜백(`_emit_progress`)을 호출할 때 정제 쿼리 정보를 전달하지 않는다. 또한 `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:213-233` (`AgenticAskUseCase._emit_progress`) 메서드의 시그니처 자체에도 쿼리 정보를 담는 매개변수가 부재하여 `refining` 단계나 후속 `searching` 단계에서 어떤 검색어로 지식을 탐색하는지 알 수 없다.
이로 인해 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:117-128` (`AskRequestedConsumer._process`)의 진행 이벤트 발행(`publish_progress`)과 209-218줄의 최종 완료 메타데이터(`publish_done`)에 정제 쿼리가 누락된다. 더불어 195-207줄의 세션 영속화(`session.append_turn`) 및 `public-python-server/src/ai_service/rag/schemas.py:50-65` (`ConversationTurn.of_assistant`), `public-python-server/src/ai_service/rag/schemas.py:95-108` (`TurnRecord`), `public-python-server/src/ai_service/rag/schemas.py:368-378` (`SessionTurnOut`)에도 정제 쿼리 필드가 없어 대화 세션에 영속화되지 않는다.
결과적으로 프론트엔드 `public-front/src/api/ai.ts:267-272` (`AgentProgress`), `public-front/src/components/AiService.tsx:1557-1566` (`AiService` 진행 인디케이터)에서는 다회차 반복 실행 중 사용자가 어떤 보완 키워드로 검색이 진행되는지 실시간으로 알 수 없고, 스트리밍 완료 후 `public-front/src/components/AiService.tsx:1392-1401` (`AiService` 메시지 메타 배지 영역) 및 세션 복원 함수 `public-front/src/components/AiService.tsx:341-354` (`handleLoadSession`)에서도 AI가 결론을 도출하기 위해 어떤 보완 질문을 던져 근거를 확보했는지 검증할 수 없다. 이는 "답을 그대로 믿지 않고 사용자가 스스로 확인할 수 있어야 한다"는 이번 분기 제품 방향에 미흡한 공백이다.

## 요구사항
- [ ] 에이전틱 RAG 유스케이스(`AgenticAskUseCase`)에서 비평 후 정제 쿼리(`next_query`) 생성 시 `refining` 단계 및 후속 회차 `searching` 단계의 `_emit_progress` 호출에 정제 쿼리를 포함하여 발행한다.
- [ ] 에이전틱 RAG 루프 동안 생성된 모든 정제 쿼리 목록(`refined_queries: list[str]`)을 누적 관리하고, 완료 메타데이터 및 진행 콜백을 통해 전달한다.
- [ ] Kafka 메시지 컨슈머(`AskRequestedConsumer`)에서 에이전틱 질의 처리 시 `publish_progress` 이벤트 페이로드에 `nextQuery`를 전달하고, `publish_done` 완료 이벤트 데이터에 `refinedQueries` 배열을 포함한다.
- [ ] 백엔드 도메인 모델(`TurnValue`, `ConversationTurn`, `TurnRecord`), 세션 리포지토리(`ConversationSessionRepository`), DTO 스키마(`SessionTurnOut`)에 `refined_queries` 필드를 추가하여 MongoDB에 영속화하고 세션 상세 조회 시 반환한다.
- [ ] 프론트엔드 API 규격(`AgentProgress`, `AskDoneMeta`, `SessionTurn`) 및 컴포넌트 인터페이스(`ChatMessage`)에 `nextQuery` 및 `refinedQueries` 선택적 속성을 추가한다.
- [ ] 프론트엔드 `AiService.tsx`의 실시간 스트리밍 인디케이터(`agent-progress-indicator`)에서 `nextQuery`가 존재할 때 정제 검색어(예: `[보완 검색: "…"]`)를 시각적으로 노출한다.
- [ ] 프론트엔드 `AiService.tsx`의 AI 메시지 버블에서 `refinedQueries`가 존재하는 경우(1개 이상), 반복 검증 배지 부근에 보완 검색 쿼리 목록 태그/배지를 렌더링한다.
- [ ] 세션 복원(`handleLoadSession`) 시에도 저장된 세션 턴의 `refinedQueries`를 복원하여 새로고침 또는 세션 재진입 시 보완 검색어 목록이 정상 렌더링되도록 보장한다.

## 비요구사항 (Out of scope)
- 단발성 비에이전틱 질의(`complexity != "complex"`)에 대한 인위적인 쿼리 재생성이나 정제 쿼리 필드 강제 주입은 수행하지 않는다. (기존 동작 유지)
- 비평 프롬프트나 `QueryRefinerService`의 쿼리 정제 휴리스틱 로직 자체는 변경하지 않는다.
- 사용자가 보완 검색 쿼리를 수동으로 편집하거나 재검색을 트리거하는 인터랙션 기능은 이번 범위에 포함하지 않는다.

## 백엔드
- `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py`
  - `_emit_progress` 메서드에 `next_query: str | None = None` 선택적 매개변수를 추가하고, `payload`에 `nextQuery: next_query`를 포함하도록 수정한다.
  - `AgenticAskUseCase.execute`에서 루프 내 정제 쿼리 목록(`refined_queries: list[str] = []`)을 수집하고, `_query_refiner.refine` 후 `refining` 단계 및 다음 `iteration`의 `searching` 단계 `_emit_progress`에 해당 쿼리를 전달한다.
  - 루프 종료 시 마지막 상태의 `refined_queries`를 완료 처리로 연계할 수 있도록 콜백 또는 반환 메타데이터로 방출한다.
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`
  - `on_progress` 핸들러에서 `data`에 포함된 `nextQuery` 및 누적된 `refined_queries`를 추적한다.
  - `publish_done` 호출 시 `done_data`에 `refinedQueries: last_refined_queries`를 포함한다.
  - `session.append_turn` 호출 시 `refined_queries=last_refined_queries`를 전달한다.
- `public-python-server/src/ai_service/rag/schemas.py`
  - `TurnValue`, `TurnRecord` 데이터클래스에 `refined_queries: list[str] | None = None` 필드를 추가한다.
  - `ConversationTurn.of_assistant` 및 `restore`에 `refined_queries: list[str] | None = None` 매개변수를 추가하고 프로퍼티로 노출한다.
  - `ConversationSession.restore` 및 `append_turn` 메서드에 `refined_queries: list[str] | None = None` 매개변수를 지원한다.
  - `SessionTurnOut` Pydantic 모델에 `refined_queries: list[str] | None = Field(default=None, alias="refinedQueries")`를 추가하고 `from_domain`에서 매핑한다.
- `public-python-server/src/ai_service/rag/repository.py`
  - `ConversationSessionRepository._to_record`에서 `t.refined_queries is not None`일 때 `turn_dict["refinedQueries"] = t.refined_queries`를 직렬화한다.
  - `ConversationSessionRepository._to_domain`에서 `t.get("refinedQueries")`를 역직렬화하여 레거시 데이터와의 하위 호환성을 보장한다.

## 프론트엔드
- `public-front/src/api/ai.ts`
  - `AgentProgress` 인터페이스에 `nextQuery?: string;` 필드를 추가한다.
  - `AskDoneMeta` 인터페이스에 `refinedQueries?: string[];` 필드를 추가한다.
  - `SessionTurn` 인터페이스에 `refinedQueries?: string[];` 필드를 추가한다.
- `public-front/src/components/AiService.tsx`
  - `ChatMessage` 인터페이스에 `refinedQueries?: string[];` 필드를 추가한다.
  - `startQuestionStreaming`의 `onDone` 콜백에서 `finalMeta?.refinedQueries`를 AI 메시지 객체에 영속화한다.
  - `handleLoadSession`에서 세션 복원 시 `detail.turns`의 `t.refinedQueries`를 `ChatMessage`의 `refinedQueries`로 매핑한다.
  - 스트리밍 진행 인디케이터(`agent-progress-indicator`) 내에 `currentProgress.nextQuery`가 존재할 경우 보완 검색어 문구를 표시한다.
  - AI 메시지 버블 내 메타 배지 영역(`message-meta-badges`)에 `msg.refinedQueries`가 1개 이상 존재할 때 보완 검색 쿼리 목록(`refined-queries-badge`)을 렌더링한다.
- `public-front/src/components/AiService.css`
  - 실시간 보완 검색어 인디케이터 및 완료 메시지 보완 검색 쿼리 태그 스타일을 추가한다.

## 수용 기준 (Acceptance Criteria)
- Given 에이전틱 RAG 질의 처리 중 첫 번째 답변의 신뢰도가 기준치 미달하여 비평이 수행될 때, When 비평 결과 다음 검색 쿼리가 도출되면, Then `phase: "refining"` 진행 이벤트에 `nextQuery` 필드가 포함되어 발행되고, 프론트엔드 스트리밍 인디케이터에 해당 정제 검색어가 표시된다.
- Given 2회 이상 반복 검증을 거친 에이전틱 RAG 답변 생성이 완료되었을 때, When `done` 이벤트가 클라이언트에 전달되면, Then `done` 이벤트 데이터에 `refinedQueries` 문자열 배열이 포함되고, AI 메시지 버블에 반복 검증 배지와 함께 보완 검색 쿼리 목록이 정상 렌더링된다.
- Given 보완 검색 쿼리가 영속화된 대화 세션을 다시 조회할 때, When 사이드바에서 해당 세션을 클릭하여 `handleLoadSession`을 호출하면, Then 새로고침 후에도 AI 메시지 버블에 저장되었던 보완 검색 쿼리 목록이 그대로 복원되어 표시된다.
- Given 단발성 일반 RAG 질의이거나 1회차에서 비평을 만족하여 추가 정제 쿼리가 없는 경우(경계 케이스), When 답변 스트리밍이 완료되거나 세션을 복원하면, Then `refinedQueries`가 빈 배열이거나 `None`/`undefined`로 안전하게 처리되어 보완 검색어 배지가 화면에 노출되지 않고 기존 UI가 깨짐 없이 정상 렌더링된다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:193-233` (`AgenticAskUseCase.execute`, `_emit_progress`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:117-128, 195-218` (`AskRequestedConsumer._process`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/schemas.py:50-108, 368-378` (`ConversationTurn`, `TurnRecord`, `SessionTurnOut`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/repository.py:105-145` (`ConversationSessionRepository._to_domain`, `_to_record`)
- 고쳐야 할 자리: `public-front/src/api/ai.ts:267-272, 298-306, 448-452` (`AgentProgress`, `SessionTurn`, `AskDoneMeta`)
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:48-58, 341-354, 671-686, 1392-1401, 1557-1566` (`ChatMessage`, `handleLoadSession`, `startQuestionStreaming`, `AiService`)

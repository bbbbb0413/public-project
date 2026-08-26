---
id: SPEC-010
title: RAG 완료(done) 이벤트의 최종 신뢰도 및 누락 정보 전달
status: done
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---
## 배경 / 문제
에이전틱 RAG 질의응답 과정에서 `AgenticAskUseCase` 는 비평(Critique)을 통해 답변의 신뢰도(`confidence`)와 누락된 정보(`missing`)를 매 반복마다 도출한다.
하지만 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:170-176` 에서는 스트리밍 완료 후 `publish_done(message.job_id)` 만 호출하여 최종 평가 메타데이터를 `done` 이벤트 페이로드에 담지 않고 있다.
이로 인해 `public-front/src/components/AiService.tsx:263-289` 에서는 완료 시점에 `lastProgress` 변수에 캐시된 마지막 중간 `progress` 이벤트 값에 의존하여 채팅 로그(`chatLog`)에 신뢰도 및 누락 정보를 기록하고 있다.
만약 중간 `progress` 이벤트가 네트워크 지연이나 유실로 인해 늦게 도착하거나 유실되는 경우, `public-front/src/api/ai.ts:194-204` 에서 수신하는 `done` 이벤트만으로는 최종 신뢰도와 누락 정보가 누락되어 화면에 빈 값으로 남게 되는 문제가 있다.
`public-python-server/src/ai_service/shared_kernel/messaging/job_event_publisher.py:33-40` 에는 이미 `publish_done(job_id, data)` 형태로 완료 페이로드를 전달할 수 있는 기능이 구현되어 있으므로, Python 컨슈머 및 프론트엔드 API/컴포넌트에서 `done` 이벤트에 최종 메타데이터(`confidence`, `missing`)를 담아 전달받도록 개선해야 한다.

## 요구사항
- [ ] `public-python-server` 의 RAG 응답 스트리밍 종료 시 `done` 이벤트 페이로드에 최종 `confidence` 와 `missing` 정보를 포함하여 발행한다
- [ ] 비에이전틱 단발성 질의(`ask_use_case.py`)이거나 신뢰도 평가 결과가 없는 경우 `done` 이벤트의 데이터는 빈 객체 또는 `None` 으로 발행되어도 정상 동작한다
- [ ] 프론트엔드 `public-front/src/api/ai.ts` 의 `askQuestionStream` 에서 `onDone` 콜백 호출 시 `done` 이벤트의 페이로드 객체(`{ confidence?: number; missing?: string[] }`)를 인자로 전달한다
- [ ] 프론트엔드 `public-front/src/components/AiService.tsx` 에서 `onDone` 콜백 인자로 전달받은 최종 `confidence` 및 `missing` 을 우선적으로 사용하여 AI 메시지 버블에 반영한다
- [ ] `done` 이벤트에 메타데이터가 없는 레거시 응답이 오더라도 기존의 `lastProgress` 또는 기본값 처리를 통해 화면이 깨지지 않고 정상 렌더링된다
- [ ] 변경 사항에 대한 백엔드 및 프론트엔드 단위 테스트를 작성한다

## 비요구사항 (Out of scope)
- 게이트웨이(`public-server`) 수정: `redis-streams-relay.service.ts` 가 이미 `data` 필드를 JSON 파싱하여 범용 중계하므로 게이트웨이 코드는 수정하지 않는다
- 대화 세션 영속화 스키마 변경: DB/Redis 세션 저장소에 신뢰도와 누락 정보를 추가 저장하는 것은 이번 범위에 포함하지 않는다
- 비에이전틱 질의(`ask_use_case.py`)에 새로운 신뢰도 산출 알고리즘을 도입하지 않는다

## 백엔드
`public-python-server` 에서 다음 파일들을 수정한다.

- `src/ai_service/rag/application/agentic_ask_use_case.py`: 스트리밍 도중 또는 완료 시 최종 계산된 `confidence` 와 `missing` 정보를 컨슈머가 획득할 수 있도록 반환 또는 콜백 메커니즘을 제공한다 (`public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:133-147`).
- `src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`: 스트림 완료 후 `self._publisher.publish_done(message.job_id, {"confidence": last_confidence, "missing": last_missing})` 형태로 최종 메타데이터를 담아 발행한다 (`public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:170-176`).

## 프론트엔드
`public-front` 에서 다음 파일들을 수정한다.

- `src/api/ai.ts`: `askQuestionStream` 의 `onDone` 콜백 시그니처를 `(finalMeta?: { confidence?: number; missing?: string[] }) => void` 로 확장하고, SSE 블록 처리부에서 `done` 이벤트의 `event.data` 객체를 파싱하여 `onDone` 에 전달한다 (`public-front/src/api/ai.ts:194-204`).
- `src/components/AiService.tsx`: `handleSendQuestion` 의 `onDone` 핸들러에서 전달받은 `finalMeta` 의 `confidence` 및 `missing` 을 우선 채택하여 `setChatLog` 에 반영한다 (`public-front/src/components/AiService.tsx:263-289`).

## 수용 기준 (Acceptance Criteria)
- Given 에이전틱 RAG 질의가 성공적으로 완료되고 비평 신뢰도가 0.85, 누락 정보가 `["추가 설명"]` 인 상황 When 백엔드가 `done` 이벤트를 발행하면 Then Redis 스트림의 `done` 이벤트 페이로드에 `confidence: 0.85`, `missing: ["추가 설명"]` 이 포함된다
- Given 프론트엔드가 `done` 이벤트로 `{ confidence: 0.9, missing: [] }` 페이로드를 수신한 상황 When 스트리밍이 종료되고 답변 메시지가 렌더링될 때 Then AI 메시지 버블에 신뢰도 90% 뱃지가 올바르게 표시된다
- Given `done` 이벤트 페이로드가 비어있거나 `null` 인 응답(경계 케이스) When 스트리밍이 완료되면 Then 에러 없이 답변 텍스트가 정상 출력되고 기존 `lastProgress` 또는 기본값이 적용된다
- Given 단발성 비에이전틱 질의를 수행하여 신뢰도 평가 메타데이터가 없는 상황 When `done` 이벤트가 수신되면 Then 신뢰도 뱃지 없이 답변 내용만 정상 표시된다

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:170-176`
- 고쳐야 할 자리: `public-front/src/api/ai.ts:194-204`
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:263-289`
- 관련 정의나 기존 구현: `public-python-server/src/ai_service/shared_kernel/messaging/job_event_publisher.py:33-40`
- 관련 정의나 기존 구현: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:133-147`

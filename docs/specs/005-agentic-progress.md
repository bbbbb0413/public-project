---
id: SPEC-005
title: 에이전트 추론 과정 노출
status: done
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: high
---

## 배경 / 문제

`agentic_ask_use_case.py` 는 답변 하나를 만들기 위해 반복 루프를 돈다. 검색하고,
생성하고, 그 결과를 스스로 비평하고, 부족하면 질의를 다시 써서 처음으로 돌아간다.
`IterationBudget` 은 최대 10회까지  허용한다.

이 과정에서 `CritiqueGeneratorService` 가 매 반복마다 세 가지를 계산한다.

| 값 | 의미 |
|---|---|
| `confidence` | 지금 답변을 얼마나 신뢰하는지 (0~1) |
| `missing` | 아직 찾지 못한 정보 목록 |
| `next_query` | 다음 반복에서 검색할 질의 |

**이 값들은 전부 버려진다.** `agentic_ask_use_case.py:86-99` 에서 계산해 루프
제어에만 쓰고 밖으로 내보내지 않는다.

사용자가 보는 것은 토큰 스트림뿐이다. 반복이 3회 돌면 그동안 화면에 아무것도
나오지 않고, 사용자는 시스템이 멈춘 것인지 생각 중인지 구분할 수 없다. 답변이
나온 뒤에도 그것이 한 번에 나온 것인지 세 번 고쳐 쓴 것인지 알 수 없다.

`JobEventPublisher` 는 이미 `session`, `token`, `sources`, `error`, `done`
다섯 가지를 Redis 스트림으로 발행한다. 게이트웨이의 중계기는
`redis-streams-relay.service.ts:43` 에서 `type: record.type ?? 'token'` 으로
**어떤 타입이든 그대로 통과시킨다.** 새 이벤트를 추가해도 게이트웨이는 고칠
필요가 없다.

## 요구사항

- [ ] `JobEventPublisher` 에 `publish_progress` 를 추가해 `type: "progress"` 이벤트를 발행한다
- [ ] 이벤트 본문은 `{iteration, phase, confidence, missing}` 을 담는다
- [ ] `phase` 는 `searching` | `generating` | `critiquing` | `refining` 중 하나다
- [ ] 에이전틱 루프가 각 단계에 들어갈 때 해당 이벤트를 발행한다
- [ ] 마지막 반복이 끝나면 최종 `confidence` 와 `missing` 을 담은 이벤트를 한 번 더 발행한다
- [ ] 프론트엔드가 `progress` 이벤트를 받아 진행 상태를 화면에 표시한다
- [ ] 답변이 완료되면 진행 표시는 사라지고, 최종 신뢰도는 답변 옆에 남는다
- [ ] `progress` 이벤트를 모르는 옛 클라이언트가 받아도 동작이 깨지지 않는다
- [ ] 위 규칙을 검증하는 테스트를 작성한다

## 비요구사항 (Out of scope)

- 비평 전문(critique 원문)이나 중간 답변 초안은 보여주지 않는다. 이번에는 단계와 신뢰도까지만 노출한다
- 사용자가 반복 횟수나 신뢰도 임계값을 조절하는 설정 화면
- 게이트웨이(`public-server`) 수정. 중계기가 이미 범용이라 손댈 곳이 없다
- 비에이전틱 경로(`ask_use_case.py`)에 같은 이벤트를 넣는 작업. 그쪽은 반복이 없다
- 진행 상태를 세션 이력에 저장하는 것
- 관측 모듈(RAGAS)과의 연동

## 백엔드

`public-python-server` 에서 두 파일을 고친다.

`src/ai_service/shared_kernel/messaging/job_event_publisher.py`
기존 `publish_token`·`publish_sources` 와 같은 모양으로 `publish_progress` 를
추가한다. 본문은 JSON 문자열로 직렬화해 `data` 필드에 넣는다 —
`publish_sources` 가 이미 그 방식이다.

`src/ai_service/rag/application/agentic_ask_use_case.py`
루프 안에서 단계가 바뀔 때마다 발행한다. 이 파일은 `JobEventPublisher` 를 직접
알지 못하므로 생성자로 콜백을 주입받는 형태를 쓴다. 호출부는
`src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py` 다.

**발행이 실패해도 답변 생성을 멈추지 않는다.** 진행 표시는 부가 기능이고,
Redis 가 잠시 불안정하다고 답변이 사라지면 안 된다.

테스트는 `tests/unit/rag/` 아래 기존 구조를 따른다.

## 프론트엔드

`public-front/src/api/ai.ts`
`streamAsk` 의 이벤트 분기(134~142행)에 `progress` 를 추가하고
`onProgress?: (p: AgentProgress) => void` 콜백을 받는다. 기존 콜백들과 같은
선택 인자 형태를 지킨다.

`public-front/src/components/AiService.tsx`
답변이 생성되는 동안 현재 단계와 반복 회차를 보여준다. 완료 후에는 최종
신뢰도만 남긴다.

표시 문구는 사용자가 이해할 수 있는 말로 쓴다. `searching` 을 그대로 쓰지 말고
"관련 문서를 찾는 중" 처럼 옮긴다.

## 수용 기준 (Acceptance Criteria)

- Given 에이전틱 루프가 2회 반복하는 질문 When 사용자가 질문하면 Then 화면에 반복 회차와 현재 단계가 순서대로 표시된다
- Given 루프가 1회로 끝나는 단순한 질문 When 답변이 완료되면 Then 진행 표시가 사라지고 최종 신뢰도가 답변 옆에 남는다
- Given 비평이 `missing: ["결제 취소 정책"]` 을 돌려준 경우 When 답변이 완료되면 Then 확인하지 못한 항목으로 "결제 취소 정책" 이 표시된다
- Given `progress` 이벤트를 처리하지 않는 옛 클라이언트 When 같은 스트림을 받으면 Then 그 이벤트를 무시하고 토큰 표시는 정상 동작한다
- Given Redis 발행이 실패한 경우 When 답변 생성이 진행 중이면 Then 진행 표시만 누락되고 답변은 끝까지 스트리밍된다

## 참고

- 계산 지점: `src/ai_service/rag/application/agentic_ask_use_case.py:86-99`
- 값 객체: `src/ai_service/rag/domain/vo/critique.py`
- 발행기: `src/ai_service/shared_kernel/messaging/job_event_publisher.py`
- 범용 중계: `public-server/apps/gateway/src/ai/stream/redis-streams-relay.service.ts:43`

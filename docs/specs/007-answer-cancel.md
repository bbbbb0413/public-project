---
id: SPEC-007
title: 답변 생성 중단
status: done
targets: [python-server, server, front]
stages: [backend, frontend, qa]
priority: high
---

## 배경 / 문제

질문을 보내면 되돌릴 방법이 없다. 저장소 전체에 `AbortController` 도
취소 엔드포인트도 없다. `job.controller.ts` 에는 작업을 만드는 `@Post()`
하나뿐이다.

에이전틱 경로에서는 이것이 더 무겁다. `IterationBudget` 은 최대 10회 반복을
허용하고, 매 반복이 검색과 생성을 새로 돈다. 사용자가 질문을 잘못 던졌다는
것을 첫 문장에서 알아차려도 끝날 때까지 기다려야 하고, 그동안 토큰 비용은
계속 나간다.

지금 사용자가 할 수 있는 것은 브라우저 탭을 닫는 것뿐이다. 그렇게 해도
백엔드는 아무것도 모른 채 남은 반복을 끝까지 돈다.

## 요구사항

- [ ] 답변이 생성되는 동안 화면에 중단 버튼이 보인다
- [ ] 중단을 누르면 즉시 스트림 구독을 끊고 입력을 다시 받을 수 있는 상태가 된다
- [ ] 중단 요청이 백엔드에 전달되어 토큰 생성이 멈춘다
- [ ] 자기 작업만 중단할 수 있다. 남의 `jobId` 로는 중단되지 않는다
- [ ] 이미 끝난 작업을 중단해도 오류가 나지 않는다
- [ ] 중단된 지점까지 받은 답변은 화면에 남는다. 지우지 않는다
- [ ] 중단된 답변임을 화면에 표시한다
- [ ] 위 규칙을 검증하는 테스트를 작성한다

## 비요구사항 (Out of scope)

- 중단한 답변을 이어서 다시 생성하는 기능
- 이미 호출된 LLM 요청을 공급자 쪽에서 취소하는 처리. 진행 중인 요청 하나는 끝나며, 그다음 반복으로 넘어가지 않는 것까지만 보장한다
- 중단 시점까지의 토큰 비용 표시
- 중단 이력을 세션에 저장하는 처리
- 관리자가 다른 사용자의 작업을 중단하는 기능

## 백엔드

세 저장소가 모두 관여한다. 취소 신호는 Redis 를 통해 전달한다 — 게이트웨이와
AI 서비스가 이미 같은 Redis 를 공유하고 있어 새 연결이 필요 없다.

**`public-server` (게이트웨이)**

`apps/gateway/src/ai/job/job.controller.ts` 에 `DELETE /ai/jobs/:jobId` 를
추가한다. 요청자의 `userId` 와 작업의 `userId` 가 다르면 거부한다 —
`job-store.service.ts:27` 의 `createJob(userId, type)` 이 이미 소유자를
저장하고 있다.

`apps/gateway/src/ai/job/job-store.service.ts` 에 취소 표시 메서드를 더한다.
기존 `createJob`·`getJob` 과 같은 방식으로 Redis 를 쓴다. 취소 표시에는
작업 메타와 같은 만료 시간을 준다.

**`public-python-server` (AI 서비스)**

`src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py` 의
토큰 발행 루프(145행 부근)에서 취소 여부를 확인한다. 취소됐으면 루프를 벗어나
`publish_done` 으로 정상 종료한다 — 오류가 아니라 사용자의 결정이다.

**매 토큰마다 Redis 를 조회하지 않는다.** 토큰은 초당 수십 개가 나오므로 그
빈도로 왕복하면 스트리밍이 느려진다. 일정 개수마다 또는 반복 경계에서 확인한다.
확인 주기는 상수로 둔다.

## 프론트엔드

`public-front/src/api/ai.ts`
`streamAsk` 가 중단 수단을 돌려주도록 한다. `AbortController` 를 쓰거나 정리
함수를 반환하는 형태 중 기존 코드에 맞는 쪽을 고른다. 중단 시 `DELETE`
요청을 보내고 스트림 구독을 닫는다.

`public-front/src/components/AiService.tsx`
생성 중에만 중단 버튼을 보여준다. 중단 후에는 받은 데까지의 답변을 남기고
중단됐다는 표시를 붙인다.

`DELETE` 요청이 실패해도 화면은 중단된 상태가 되어야 한다. 사용자 입장에서
중단은 이미 끝난 일이고, 서버 정리가 늦는 것은 그 뒤의 문제다.

## 수용 기준 (Acceptance Criteria)

- Given 답변이 스트리밍되는 중 When 사용자가 중단을 누르면 Then 토큰 표시가 즉시 멈추고 입력창이 다시 활성화된다
- Given 중단된 답변 When 화면을 보면 Then 중단 시점까지 받은 내용이 남아 있고 중단됐다는 표시가 붙는다
- Given 에이전틱 루프가 2회차를 도는 중 When 중단 요청이 도착하면 Then 3회차로 넘어가지 않고 종료된다
- Given 다른 사용자의 `jobId` When 중단을 요청하면 Then 거부되고 그 작업은 계속 진행된다
- Given 이미 완료된 `jobId` When 중단을 요청하면 Then 오류 없이 처리되고 화면 상태가 바뀌지 않는다
- Given `DELETE` 요청이 실패한 경우 When 사용자가 중단을 눌렀다면 Then 화면은 중단 상태가 되고 오류 메시지를 띄우지 않는다

## 참고

- 작업 생성: `public-server/apps/gateway/src/ai/job/job.controller.ts:25`
- 소유자 저장: `public-server/apps/gateway/src/ai/job/job-store.service.ts:27`
- 토큰 발행 루프: `src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:145`
- 반복 예산: `src/ai_service/rag/domain/vo/iteration_budget.py`

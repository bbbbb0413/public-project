---
id: SPEC-022
title: 스트리밍 RAG 질문 — 진행 단계 정리, SSE 재연결 클라이언트 연동, 멱등 재제출
status: done
targets: [server, front]
stages: [backend, frontend]
priority: high
---

## 출처
`UI _ UX 개선 및 AI 관련한 다양한 기능 추가하고싶어_기능명세서_2026-08-28.md`의 "1. AI 질의응답 경험" 중 **1.1 스트리밍 RAG 질문**(1.1.1 질문 작업 스트림 연결, 1.1.2 질문 작업 생성). 답변 근거 확인/평가 피드백은 [[023-answer-citation]], [[024-answer-feedback]]에서 별도로 다룬다.

## 배경
스트리밍 RAG 질의응답 자체는 이미 대부분 구현되어 있다. 이 문서의 목적은 "무엇을 새로 만들 것인가"가 아니라 "이미 있는 것 중 명세서와 다르게 동작하는 부분을 어떻게 정리할 것인가"에 가깝다.

## 현재 구현 상태 (조사 결과)

| 항목 | 상태 | 근거 |
|---|---|---|
| 질문 제출 → jobId 즉시 반환 | ✅ 구현됨 | `apps/gateway/src/ai/job/job.controller.ts:35-58` `POST /ai/rag/jobs` → 202 + `{jobId}`, Kafka `ask.requested` 발행 |
| SSE로 진행 상태 + 토큰 스트리밍 | 🟡 부분구현 | `apps/gateway/src/ai/stream/job-stream.controller.ts:35-84` |
| Last-Event-ID 기반 재생(서버) | ✅ 구현됨 | `job-stream.controller.ts:43,63,67` — `Last-Event-ID` 헤더 읽어 `RedisStreamsRelayService.readNext(jobId, lastId)` |
| Last-Event-ID 기반 재생(클라이언트) | ❌ 미구현 | `public-front/src/api/ai.ts:41-48` — fetch에 `Last-Event-ID` 헤더 전송 안 함, 끊긴 뒤 자동 재접속 로직 없음(1회성 구독) |
| 작업 취소 | ✅ 구현됨 | `job.controller.ts:60-83` `DELETE /ai/jobs/:jobId`(`/ai/rag/jobs/:jobId`), 소유자 검증 포함. 컨슈머는 매 청크마다 `job:{jobId}:cancelled` Redis 키 확인(`ask_requested_consumer.py:153-156`) |
| 동일 요청 반복 시 기존 작업 반환(멱등성) | ❌ 미구현 | `apps/gateway/src/ai/job/job-store.service.ts:27-38` `createJob`이 매번 `randomUUID()`로 새 잡 생성. 중복 감지 로직 없음 |
| 프론트 UI(질문 제출/스트림 구독/취소/진행 표시) | ✅ 구현됨 | `public-front/src/components/AiService.tsx`, `src/api/ai.ts` |

## 명세서와 현재 구현 간 상충 / 차이

1. **진행 단계 라벨 불일치**: 명세서는 "검색·생성·검토·보완" 4단계 고정 표시를 요구하지만, 실제 SSE 이벤트는 `session/token/sources/progress/done/error` 6종(`public-python-server/src/ai_service/.../core/events.py:14-46`)이다. `progress` 이벤트는 질문이 "complex"로 분류되어 agentic 경로(`ask_requested_consumer.py:117-136`)를 탈 때만 발행되고, 내용도 `{confidence, missing}` 형태의 자기평가 루프 정보라 명세서가 말하는 범용 4단계와 의미가 다르다. **단순 질문 경로는 progress 이벤트 자체가 없어 "검색 중" 표시가 전혀 뜨지 않는다.**
   - 결정 필요: (a) 명세서의 4단계 라벨을 폐기하고 실제 이벤트 체계(session/token/sources/progress/done/error) 기준으로 화면 문구를 다시 쓸지, (b) 단순 질문 경로에도 최소한의 "검색 중/생성 중" 단계 이벤트를 추가해 명세서 요구를 그대로 맞출지 — 작업 착수 전에 정해야 한다.
2. **SSE 재연결이 서버만 준비돼 있고 실제로 발동하지 않음**: 서버 재생 기능은 이미 있는데 프론트가 안 쓴다. 즉 "기능이 없다"가 아니라 "배선이 안 됐다"에 가깝다 — 프론트에서 연결이 끊기면 `EventSource`/fetch 재시도 시 마지막으로 받은 이벤트 ID를 기억해 `Last-Event-ID` 헤더로 보내도록 `src/api/ai.ts`만 고치면 될 가능성이 높다.
3. **멱등 재제출 규칙이 명세서에만 있고 코드엔 없음**: "동일한 제출 요청이 반복되면 기존 작업을 반환한다"는 문구를 그대로 구현하려면 (질문 내용, userId, 시간창) 기준의 중복 판정 키가 필요한데 지금 데이터 모델엔 그런 키가 없다. payment 서비스의 idempotencyKey 패턴([[SPEC-019]])을 참고할 수 있다.

## 요구사항 (이번 작업 범위)
1. 프론트에서 SSE 연결이 끊기면(네트워크 오류, 탭 백그라운드 복귀 등) 마지막으로 수신한 이벤트 ID를 `Last-Event-ID` 헤더에 실어 재구독하도록 `src/api/ai.ts`를 수정한다.
2. 진행 단계 표시 방식을 위 "결정 필요" 항목 중 하나로 확정하고 반영한다(기본 추천: 단순 질문 경로에도 최소 `progress`성 이벤트를 추가해 명세서의 4단계 기대를 맞추는 쪽 — 사용자에게 "아무 표시 없이 멈춰있는" 구간을 없앨 수 있음).
3. 질문 제출에 멱등키(또는 최소한 "같은 사용자가 짧은 시간 안에 같은 질문을 또 보내면 새 잡 대신 기존 잡을 반환") 로직을 추가할지 결정하고, 하기로 하면 `job-store.service.ts`에 반영한다.

## 비요구사항 (Out of scope)
- 답변 근거(citation) 표시 방식 변경 — [[023-answer-citation]].
- 답변 평가/피드백 — [[024-answer-feedback]].
- RAG 검색/생성 알고리즘 자체 개선(Python 파이프라인의 검색 품질) — 이 문서는 전달/표시 계층만 다룬다.

## 구현 기록

**완료 (2026-08-28)**

- **SSE 재연결 클라이언트 연동** (`public-front/src/api/ai.ts`)
  - `askQuestionStream`/`subscribeIngestJob`가 각자 중복 구현하던 fetch+reader+버퍼 파싱 루프를 `connectJobStream` 공용 헬퍼로 통합했다.
  - `parseSseEvent`가 `id:` 라인도 파싱하도록 확장하고, `connectJobStream`이 마지막으로 받은 이벤트 id를 추적하다가 fetch/스트림 자체가 예외로 끊기면(네트워크 오류 등) `Last-Event-ID` 헤더에 실어 최대 3회, 지수 백오프(1s/2s/3s)로 재접속한다. done/error 정상 종료나 터미널 이벤트 없는 자연 종료는 기존 동작을 그대로 유지해 회귀를 막았다.
  - 재접속 로직을 검증하는 신규 테스트(`ai.test.ts`)를 추가했다 — 스트림이 첫 청크 이후 끊기면 두 번째 fetch 호출에 `Last-Event-ID: '1'`이 실리는지 확인.
- **단순 질문 경로 진행 단계 보강** (`public-python-server/.../ask_requested_consumer.py`)
  - 기존엔 "complex"(에이전틱) 경로만 `searching/generating/critiquing/refining` progress 이벤트를 냈고, 대부분인 단순 질문 경로는 아무 신호 없이 곧장 답변이 나왔다(사용자 입장에선 아무 표시 없이 멈춰 보임).
  - 단순 경로 진입 시 `searching` 이벤트를 먼저 발행하고, 첫 콘텐츠 청크(근거 문서 or 토큰) 직전에 `generating` 이벤트를 한 번 발행하도록 최소한으로 보강했다. `confidence`/`missing` 필드는 실제 값이 없으므로 아예 포함하지 않는다.
  - 프론트 `AgentProgress` 타입의 `confidence`/`missing`을 optional로 바꿔, 값이 없을 때 "신뢰도 0%" 같은 오해 소지가 있는 배지가 뜨지 않도록 했다(`AiService.tsx`의 기존 `!== undefined` 폴백 로직은 손대지 않고 타입만 맞췄다).
  - 기존 "단순 경로는 progress를 전혀 안 보낸다"를 검증하던 테스트를 새 기대값(searching → generating 2건)으로 갱신했다.
- **RAG 질문 생성 멱등성** (`apps/gateway/src/ai/job`)
  - payment의 idempotencyKey 패턴([[019-payment-scale-architecture-roadmap]])을 그대로 따랐다. 프론트가 질문 제출마다 `createIdempotencyKey()`(payment/ai가 공유하도록 `src/utils/idempotency-key.ts`로 추출)로 키를 만들어 `POST /ai/rag/jobs` 바디에 실어 보낸다.
  - `JobStoreService.createJob`이 `SET NX`로 먼저 잡ID를 선점하고, 선점 실패 시(동시 중복 요청) 기존 잡을 그대로 반환한다(`{job, isNew}`). `isNew`가 false면 `JobController`가 Kafka `ask.requested`를 재발행하지 않는다 — 상태값(`queued` 등)으로 판별하면 아직 처리 전인 기존 잡에 대해 이중 발행될 수 있어 별도 플래그로 명시했다.
  - `knowledge-job.controller.ts`의 기존 `createJob` 호출부(멱등키 미사용)도 반환 타입 변경에 맞춰 갱신했다 — 동작은 그대로([[025-knowledge-document-ingest]] 범위에서 별도로 다룸).
- 검증: 프론트 전체 154개 테스트, 게이트웨이 유닛 40개, 파이썬 RAG 유닛 17개(ruff 통과) 모두 통과. `tsc -b`(프론트/게이트웨이) 클린. `docker compose build gateway ai-service-py frontend` 후 재기동해 라우트 매핑과 무오류 기동을 로그로 확인했다.
- **다루지 않은 것**: 진행 단계 라벨을 "검색·생성·검토·보완" 4단계 고정 문구로 강제 통일하지는 않았다 — 기존 `PHASE_LABELS`(searching/generating/critiquing/refining)를 그대로 재사용했고, 이게 명세서 문구와 자연스럽게 대응된다고 판단했다. 실제 로그인 세션으로 브라우저에서 질문을 던져 SSE가 눈에 보이게 재연결되는지까지는 확인하지 못했다(테스트로만 검증) — 로그인 계정이 없다.

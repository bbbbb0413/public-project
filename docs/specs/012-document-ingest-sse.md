---
id: SPEC-012
title: 문서 업로드 후 SSE 기반 실시간 인제스트 상태 수신 및 폴링 제거
status: done
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
현재 `public-server/apps/gateway/src/ai/knowledge/knowledge-job.controller.ts:80-90` 에서는 문서 업로드 요청에 대해 비동기 인제스트 작업을 생성하고 `{ jobId }` 를 반환하며, `public-python-server/src/ai_service/knowledge/infrastructure/messaging/ingest_requested_consumer.py:78-101` 에서 인제스트 작업 완료 시 `done`, 실패 시 `error` 이벤트를 Redis Streams에 발행한다.
또한 백엔드 게이트웨이 `public-server/apps/gateway/src/ai/stream/job-stream.controller.ts:35-56` 에는 작업의 진행/종료 이벤트를 실시간으로 전달하는 SSE 엔드포인트(`GET /ai/jobs/:jobId/stream`)가 이미 갖추어져 있다.
그러나 프론트엔드 `public-front/src/components/AiService.tsx:184-196` 에서는 업로드 응답으로 받은 `jobId` 의 SSE 스트림을 구독하지 않고, `setInterval` 을 통해 3초마다 `getDocuments()` 전체 목록 조회를 최대 30초 동안 반복 호출하는 단순 폴링 방식을 사용하고 있다.
이로 인해 다음과 같은 문제점이 발생한다.
- 인제스트가 수백 밀리초 만에 끝나도 다음 3초 폴링 주기까지 화면 상태 갱신이 지연된다.
- 인제스트 처리 중 에러(`error` 이벤트)가 발생해도 프론트엔드가 이를 즉시 인지하지 못하고 타임아웃까지 불필요한 조회를 반복한다.
- 백엔드에 불필요한 다건의 HTTP 요청 부하가 발생한다.

## 요구사항
- [ ] `public-front/src/api/ai.ts` 에 `jobId` 를 받아 SSE 스트림(`GET /ai/jobs/:jobId/stream`)을 구독하고 `done`, `error` 이벤트를 수신하는 헬퍼 함수(예: `subscribeJobStream` 또는 `subscribeIngestJob`)를 추가한다.
- [ ] `public-front/src/components/AiService.tsx` 의 `handleFileUpload` 에서 문서 업로드 성공 후 반환된 `jobId` 로 SSE 스트림을 구독하여 인제스트 완료/실패를 실시간으로 감지한다.
- [ ] SSE 스트림으로부터 `done` 이벤트를 수신하면 즉시 `fetchDocuments()` 를 호출하여 문서 목록을 갱신하고 스트림 연결을 닫는다.
- [ ] SSE 스트림으로부터 `error` 이벤트를 수신하면 에러 메시지(이벤트 데이터의 `error` 필드 또는 기본 메시지)를 `setErrorMsg` 로 화면에 표시하고 스트림 연결을 닫는다.
- [ ] 기존의 3초 주기 `setInterval` 폴링 로직을 제거한다.
- [ ] 브라우저 네트워크 단절 등으로 SSE 연결 오류 발생 시 최대 15초 후 타임아웃 처리되어 최종 `fetchDocuments()` 를 1회 호출하고 연결을 정리한다.
- [ ] 컴포넌트 언마운트 시 활성화된 SSE 연결(`EventSource`)을 정상 해제(close)하여 메모리 누수를 방지한다.
- [ ] 단위 테스트 및 컴포넌트 테스트를 작성하여 SSE 이벤트 수신에 따른 문서 갱신 및 에러 처리를 검증한다.

## 비요구사항 (Out of scope)
- 백엔드 게이트웨이 및 Python 서버의 잡 생성 및 이벤트 발행 로직 수정
- 문서 업로드 외의 RAG 질의응답 스트리밍 로직 변경
- SSE 재연결 시 `Last-Event-ID` 기반의 백오프 재시도 정책 추가 구현
- 문서 삭제(`handleDeleteDocument`)의 비동기 잡 전환

## 프론트엔드
`public-front/src/api/ai.ts`
- `uploadDocument` 의 반환 타입을 `{ jobId: string }` 형태로 명시
- `subscribeIngestJob(jobId: string, callbacks: { onDone?: () => void; onError?: (error: string) => void })` 함수 추가

`public-front/src/components/AiService.tsx`
- `handleFileUpload` 함수 내 `setInterval` 기반 폴링 제거 및 `subscribeIngestJob` 연동
- 언마운트 시 클린업 로직 처리

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 유효한 문서를 업로드했을 때
  When 백엔드가 `jobId` 를 응답하고 비동기 인제스트 완료 후 `done` 이벤트를 발행하면
  Then 프론트엔드는 폴링 없이 즉시 `done` 이벤트를 받아 `fetchDocuments()` 를 1회 호출하고 스트림을 종료한다.
- Given 사용자가 문서를 업로드했으나 백엔드 처리 중 오류가 발생했을 때
  When 백엔드에서 `error` 이벤트(`{ error: "파싱 실패" }`)를 발행하면
  Then 프론트엔드는 즉시 스트림을 닫고 에러 배너에 해당 오류 메시지를 표시한다.
- Given 업로드 후 백엔드 응답에 `jobId` 가 없거나 빈 값인 경우 (경계 케이스)
  When `handleFileUpload` 가 실행되면
  Then SSE 구독을 시도하지 않고 기존대로 `fetchDocuments()` 를 1회 호출한 후 안전하게 종료한다.
- Given SSE 연결이 응답 없이 장시간 유지되는 경우 (타임아웃 케이스)
  When 15초 동안 종료 이벤트가 오지 않으면
  Then SSE 연결을 닫고 최종 1회 `fetchDocuments()` 를 호출하여 상태를 확인한다.

## 참고
- 폴링 로직 위치: `public-front/src/components/AiService.tsx:184-196`
- 업로드 API 함수: `public-front/src/api/ai.ts:10-19`
- 백엔드 잡 발행 및 jobId 반환: `public-server/apps/gateway/src/ai/knowledge/knowledge-job.controller.ts:80-90`
- 백엔드 SSE 엔드포인트: `public-server/apps/gateway/src/ai/stream/job-stream.controller.ts:35-56`
- Python 인제스트 완료/에러 이벤트 발행: `public-python-server/src/ai_service/knowledge/infrastructure/messaging/ingest_requested_consumer.py:78-101`

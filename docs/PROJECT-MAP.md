# 코드 지도

<!--
이 문서는 조사(survey) 스테이지가 덮어쓴다. 손으로 고치지 않는다.
낡았다고 느끼면 고치는 대신 파이프라인을 돌린다 — 타깃 전체에
map_stale_commits 만큼 커밋이 쌓이면 기획 앞에서 자동으로 다시 만든다.

구조가 아니라 좌표를 담는 문서다. 서비스 경계·계약·역할은 README.md 가,
제품 방향은 PRODUCT.md 가 담당한다.
-->

## 저장소 구조 규약

경로 하나하나보다 이 규약이 오래 간다. 아래 흐름의 좌표가 어긋나 보이면
규약을 기준으로 같은 자리를 다시 찾는다.

- `public-front` — `src/api/*.ts` 가 gateway 호출을 전담하고, `src/components/*.tsx`
  는 그 함수만 부른다. 컴포넌트에서 직접 `fetch` 하지 않는다.
- `public-server` — NestJS 모노레포. `apps/<서비스>` 가 실행 단위, `libs/<이름>`
  이 공유 코드다. gateway 만 HTTP 를 열고 나머지는 gRPC 컨트롤러(`*.grpc-controller.ts`)
  로만 열린다.
- `public-python-server` — 기능별 모듈 구조. 기능 디렉토리마다 `router.py`(입구),
  `service.py`/`application/*_use_case.py`(로직), `repository.py`(저장),
  `schemas.py`(계약), `dependencies.py`(조립)를 둔다. 공통 설정·DB·이벤트·보안은
  `core/` 에 모인다. Kafka 소비자는 `<기능>/infrastructure/messaging/*_consumer.py` 다.
  2026-08-27 에 계층형(도메인/애플리케이션/인프라/프레젠테이션)에서 옮겨왔다.

## 흐름

- **RAG 질의응답 (Kafka 잡 + SSE)**: `public-front/src/components/AiService.tsx:377`
  (`handleSendQuestion`) -> `public-front/src/api/ai.ts:226` (`askQuestionStream` — POST 후
  SSE 구독) -> `public-server/apps/gateway/src/ai/job/job.controller.ts:35` (`POST /ai/rag/jobs`,
  Kafka 발행) -> Kafka `ai.rag.ask.requested` ->
  `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:91`
  (`_process` — 프롬프트 해석·검색·생성) ->
  `public-python-server/src/ai_service/core/events.py:20` (`publish_token`, Redis Streams
  `ai:job:{jobId}:events`) -> `public-server/apps/gateway/src/ai/stream/job-stream.controller.ts:35`
  (`GET /ai/jobs/:jobId/stream` SSE 릴레이) -> 화면 렌더링
- **답변 생성 중단**: `public-front/src/components/AiService.tsx:355` (`handleCancelStreaming`)
  -> `public-server/apps/gateway/src/ai/job/job.controller.ts:60` (`DELETE /ai/jobs/:jobId`,
  Redis 취소 플래그) ->
  `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:153`
  (토큰 루프에서 `job:{jobId}:cancelled` 확인 후 조기 종료)
- **문서 업로드 · 인제스트**: `public-front/src/components/AiService.tsx:214`
  (`handleFileUpload`) -> `public-front/src/api/ai.ts:14`, `:25` (`uploadDocument`,
  `subscribeIngestJob`) -> `public-server/apps/gateway/src/ai/knowledge/knowledge-job.controller.ts:48`
  (멀티파트 수신, Redis 스테이징, Kafka 발행) -> Kafka `ai.knowledge.ingest.requested` ->
  `public-python-server/src/ai_service/knowledge/infrastructure/messaging/ingest_requested_consumer.py:82`
  -> `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py:86`
  (파싱·청킹·임베딩·Qdrant 저장) -> 같은 SSE 채널로 완료 통지
- **개인화 시스템 프롬프트**: `public-front/src/components/AiService.tsx:317` (`handleSavePrompt`)
  -> `public-front/src/api/ai.ts:136`, `:141` -> `public-server/apps/gateway/src/ai/proxy/my-prompt-proxy.controller.ts:23`
  (userId 를 인증 세션에서 채워 프록시) -> `public-python-server/src/ai_service/prompt/router.py:25`
  -> `public-python-server/src/ai_service/prompt/service.py:38` (`get_active_prompt` — 개인 -> 전역 ->
  기본값) -> `public-python-server/src/ai_service/prompt/repository.py:55` (`find_active_for_user`)
- **실시간 채팅 (WebSocket)**: `public-front/src/hooks/useChatSocket.ts:29` (Socket.IO 연결,
  `auth: { token }`) -> `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:76`
  (`join_room` — gRPC 로 히스토리 조회) / `:191` (`send_message`) ->
  `public-server/apps/chat-service/src/message/rpc/chat.grpc-controller.ts:13` (`SaveMessage`)
  -> `public-server/apps/chat-service/src/message/message.service.ts:16` (`persistAndNotify` —
  Redis ZSET 저장 + 샤드 Pub/Sub 발행) -> `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:132`
  (`onShardMessage` — ZSET pull 후 방에 emit)
- **관리자 인증 · 유저 관리**: `public-front/src/api/admin.ts` -> `public-server/apps/gateway/src/admin/admin-gateway.controller.ts:55`
  (`POST /admin/auth/signup`) -> admin-server gRPC `:50054` -> Bull `mail` 큐 -> identity gRPC `SendMail`
- **결제 (gRPC)**: `public-front/src/components/Payment.tsx` -> `public-front/src/api/payment.ts`
  -> `public-server/apps/gateway/src/payment/payment-gateway.controller.ts:26` ->
  `public-server/apps/payment/src/payment/rpc/payment.grpc-controller.ts:17` ->
  `public-server/apps/payment/src/payment/application/create-payment.use-case.ts:14`

## 확장 지점

- `public-python-server/src/ai_service/rag/rag_composition.py:20` — 하이브리드 검색·RRF 융합·HyDE·
  리랭커·쿼리 분해기를 DI 로 조립한다. 검색 단계를 갈아 끼우는 자리.
- `public-python-server/src/ai_service/knowledge/infrastructure/providers/embedding_factory.py:1` —
  임베딩 프로바이더를 환경변수로 고른다. Ollama·Google 이 이미 붙어 있어 추가가 쉽다.
- `public-python-server/src/ai_service/llm_gateway/infrastructure/providers/factory.py:1` —
  LLM 공급자(Groq·Anthropic·OpenAI·Ollama) 플러그인 지점.
- `public-server/apps/gateway/src/ai/proxy/` — ai-service-py 로 가는 HTTP 프록시 컨트롤러가 한 곳에
  모여 있다. 파이썬 쪽 라우터를 하나 열면 여기에 컨트롤러 하나만 붙이면 된다.
- `public-server/apps/gateway/src/auth/gateway-auth.guard.ts:64` — JWT·Basic·API Key 세 갈래를 모두
  `Session` 하나로 정규화한다. 새 인증 수단은 여기에만 붙인다.
- `public-front/src/api/client.ts:1` — Axios 인터셉터가 JWT 주입과 에러 처리를 중앙화한다.

## 아쉬운 곳

- **지식베이스 문서 삭제에 확인 절차가 없다** — `public-front/src/components/AiService.tsx:291`
  (`handleDeleteDocument`) 가 확인 대화상자 없이 곧바로 `deleteDocument(id)` 를 호출한다. 실수로 누르면
  즉시 영구 삭제되고, 삭제 중 로딩 상태도 보이지 않는다. (SPEC-013)
- **결제 응답의 status 가 하드코딩돼 있다** — `public-server/apps/payment/src/payment/rpc/payment.grpc-mapper.ts:22`
  이 도메인 모델에 상태 필드가 없어 `status: 'COMPLETED'` 를 무조건 채운다. 실패·대기·취소를 표현할 수 없다.
- **결제 조회 응답에 상품 식별자가 없다** — `public-server/libs/rpc/proto/payment.proto:20` 의
  `PaymentReply` 에 `product_id` 가 없는데 `public-front/src/components/Payment.tsx` 는 `productId` 를
  읽어 빈 값을 그린다. 생성 요청(`payment.proto:13`)에는 있는 필드다. (SPEC-009)
- **채팅 소켓에 인증 컨텍스트가 주입되지 않는다** — `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:61`
  (`handleConnection`) 이 토큰의 존재만 확인하고 검증·주입을 하지 않는다. 그래서 `:214` 에서 `client.data.user`
  가 항상 `undefined` 이고 메시지 전송이 `Authentication failed` 로 거부된다. HTTP 쪽은
  `public-server/apps/gateway/src/auth/gateway-auth.guard.ts:67` 이 같은 일을 이미 하고 있다. (SPEC-017)
- **죽은 NestJS ai-service 가 남아 있다** — `public-server/apps/ai-service` 는 128개 파일이 남아 있으나
  루트 `docker-compose.yml` 에 없고 2026-08-09 이후 커밋이 없다. 역할은 `public-python-server` 가 가져갔다.
  그런데도 `public-server/package.json:52` 의 `test:all` 이 `test:ai` 를 계속 돌린다.
- **admin-server 에 좁힌 테스트 게이트가 없다** — 오늘 `apps/admin-server` 가 생겼는데
  `.specflow/config.yaml` 의 `scoped_test` 에 항목이 없고 `package.json` 에도 `test:admin` 이 없다.
  admin-server 를 건드리는 spec 은 기본 게이트(`libs/common/src/utils`)만 통과하고 검사받지 않는다.

## 건드리면 안 되는 곳

- `public-server/libs/auth/src/auth/strategy/`, `public-server/libs/auth/src/auth/auth.service.ts` —
  토큰 발급·검증과 비밀키를 다루는 지점.
- `public-server/libs/common/src/databases/typeorm/migrations` — 운영 스키마 이력.
- `public-server/apps/payment/src/payment/application/create-payment.use-case.ts` 의 금액 계산 경로 —
  표시·전달은 대상이지만 금액을 만드는 곳은 아니다.
- `public-python-server/src/ai_service/rag/application/filter/secret_pii_scanner.py`,
  `public-python-server/src/ai_service/rag/application/filter/rag_content_validator.py` —
  프롬프트 인젝션 방어와 개인정보·비밀값 마스킹 규칙.

<!-- specflow-survey generated=2026-08-27T21:54:49 front=679a73eb7736 python-server=0b9f40ce0790 server=46fc6d70c876 -->

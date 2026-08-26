# 코드 지도

## 흐름
- **AI RAG 질의응답 (Stream)**: `public-front/src/components/AiService.tsx:279-336` (질문 전송) -> `public-front/src/api/ai.ts:198-309` (POST `/ai/rag/jobs` & Fetch SSE 구독) -> `public-server/apps/gateway/src/ai/job/job.controller.ts:35-58` (잡 생성 및 Kafka 발행) -> `public-server/apps/gateway/src/ai/kafka/ai-kafka-producer.service.ts` -> Kafka(`ai.rag.ask.requested`) -> `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:80-173` (메시지 수신 및 분기) -> `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py` (Agentic 검색/생성/비평 루프) -> `public-python-server/src/ai_service/shared_kernel/messaging/job_event_publisher.py` (Redis Streams `job:{jobId}:events` 발행) -> `public-server/apps/gateway/src/ai/stream/job-stream.controller.ts:35-84` (SSE 릴레이) -> `public-front/src/components/AiService.tsx:298-320` (스트리밍 화면 렌더링)
- **문서 업로드 및 인제스트**: `public-front/src/components/AiService.tsx:192-267` -> `public-front/src/api/ai.ts:10-19` -> `public-server/apps/gateway/src/ai/knowledge/knowledge-job.controller.ts:48-90` (멀티파트 수신, Redis 임시 스테이징, Kafka 발행) -> Kafka(`ai.knowledge.ingest.requested`) -> `public-python-server/src/ai_service/knowledge/infrastructure/messaging/ingest_requested_consumer.py:68-102` -> `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py` (파싱, 청킹, 임베딩, Qdrant 벡터 저장) -> Redis Streams `done` 이벤트 -> `public-front/src/api/ai.ts:25-45` 및 `public-front/src/components/AiService.tsx:245-258` (SSE 기반 실시간 완료 수신 후 목록 갱신)
- **실시간 채팅 (WebSocket)**: `public-front/src/components/ChatRoom.tsx:16-21` -> `public-front/src/hooks/useChatSocket.ts:83-100` -> `public-front/src/flatbuffers/chatUtils.ts:22-42` (FlatBuffers 직렬화) -> `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:69-124` (gRPC 호출) -> `public-server/apps/chat-service/src/message/rpc/chat.grpc-controller.ts:13-40` -> `public-server/apps/chat-service/src/message/message.service.ts:16-28` (Redis ZSET 저장 및 Pub/Sub 발행) -> `public-server/apps/chat-service/src/socket/socket-connection.service.ts:79-122` (클러스터 브로드캐스트) -> `public-front/src/hooks/useChatSocket.ts:54-69` (수신 및 역직렬화)
- **결제 요청 (gRPC)**: `public-front/src/components/Payment.tsx:66-78` -> `public-front/src/api/payment.ts:12-19` -> `public-server/apps/gateway/src/payment/payment-gateway.controller.ts:26-48` -> `public-server/apps/payment/src/payment/rpc/payment.grpc-controller.ts:17-25` -> `public-server/apps/payment/src/payment/application/create-payment.use-case.ts:14-25` -> `public-server/apps/payment/src/payment/infrastructure/persistence/payment.repository-impl.ts` (TypeORM DB 저장)

## 확장 지점
- `public-python-server/src/ai_service/rag/rag_composition.py:20-60` — RAG 검색 파이프라인의 구성요소(하이브리드 검색, RRF 융합, HyDE, 리랭커, 쿼리 분해기)가 DI 방식으로 조립되어 있어 새로운 RAG 기법 추가 용이
- `public-python-server/src/ai_service/llm_gateway/infrastructure` — Provider 인터페이스를 구현하여 새로운 LLM 공급자(Claude, Groq, Ollama, OpenAI 등)를 유연하게 플러그인 가능
- `public-server/libs/llm/src/domain/port/llm-provider.port.ts:1-18` — NestJS 측 LLM Provider 포트 추상화가 완료되어 있어 백엔드 LLM 엔진 교체 용이
- `public-front/src/api/client.ts:1-41` — Axios 인터셉터 기반의 JWT 자동 주입 및 에러 처리가 중앙화되어 있어 신규 API 클라이언트 메서드 추가 용이

## 아쉬운 곳
- **지식베이스 문서 삭제 시 확인 절차 부재 및 비동기 상태 피드백 누락** — `public-front/src/components/AiService.tsx:269-277` 및 `391-394` 에서 삭제 버튼 클릭 시 확인 대화상자 없이 즉시 `deleteDocument(id)` API를 호출한다. 실수로 클릭 시 즉시 영구 삭제되며, 삭제 중 로딩 상태 표시 및 백엔드 에러 메시지(`public-front/src/components/AiService.tsx:274-276`) 노출이 누락되어 있다.
- **RAG 답변 생성 중단 UI 미연동 및 백엔드 취소 미감지** — `public-server/apps/gateway/src/ai/job/job.controller.ts:60-84` 에 잡 취소 엔드포인트(`DELETE /ai/jobs/:jobId`)가 구현되어 있으나, `public-front/src/components/AiService.tsx:644-651` 에서는 스트리밍 중 중단 버튼이 없고 `public-front/src/api/ai.ts:198-309` 에도 취소 요청 메커니즘이 없다. 또한 `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:151-159` 의 토큰 스트리밍 루프에서 Redis 취소 플래그를 확인하지 않아 사용자가 화면을 벗어나도 끝까지 토큰을 생성한다.
- **결제 응답의 status 필드 하드코딩** — `public-server/apps/payment/src/payment/rpc/payment.grpc-mapper.ts:17-24` 에서 Payment 도메인 모델에 상태 필드가 없어 gRPC 응답 시 `status: 'COMPLETED'` 로 무조건 하드코딩되어 반환된다. 결제 실패/대기/취소 등의 상태 전환을 표현할 수 없다.
- **결제 조회 응답 시 상품 식별자(productId) 누락** — `public-server/libs/rpc/proto/payment.proto:20-25` 의 `PaymentReply` 메시지에는 `product_id` 필드가 없으나, `public-front/src/components/Payment.tsx:118` 및 `170` 에서는 `receipt.productId`, `queryResult.productId`에 접근하여 화면에 빈 값으로 렌더링된다.
- **채팅 게이트웨이 인증 컨텍스트 불일치** — `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:93-96` 에서 `client.data.user`의 존재 여부를 검사하지만, `handleConnection`(`public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:34-42`)에서는 토큰 유무만 확인하고 `client.data.user` 객체를 주입하는 인증 디코딩 로직이 누락되어 메시지 전송 시 인증 실패가 발생할 수 있다.

## 건드리면 안 되는 곳
- `public-server/libs/auth` 및 `public-server/apps/identity` — 사용자 인증 토큰 발급, 비밀번호 해싱, 보안 세션 저장소 및 인증 가드가 결합된 핵심 보안 영역.
- `public-server/libs/common/src/databases/typeorm/migrations` 및 ORM 스키마 — 운영 데이터베이스 스키마 및 마이그레이션 이력이 관리되는 파일로 DB 정합성에 직접적인 영향을 줌.
- `public-python-server/src/ai_service/rag/domain/policy/injection_patterns.py:5-32` 및 `secret_pii_patterns.py:12-24` — 프롬프트 인젝션 방어 및 개인정보/비밀값 마스킹 보안 정책 정의 파일.

<!-- specflow-survey generated=2026-08-26T21:30:19 front=0e12d52d4704 python-server=cd7b0e30f463 server=f4fa1e939693 -->

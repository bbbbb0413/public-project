# 코드 지도
<!--
이 문서는 조사(survey) 스테이지가 덮어쓴다. 손으로 고치지 않는다.
낡았다고 느끼면 고치는 대신 파이프라인을 돌린다 — 타깃 전체에
map_stale_commits 만큼 커밋이 쌓이면 기획 앞에서 자동으로 다시 만든다.

구조가 아니라 좌표를 담는 문서다. 서비스 경계·계약·역할은 README.md 가,
제품 방향은 PRODUCT.md 가 담당한다.
-->
## 저장소 구조 규약
- `public-front` — `src/api/*.ts` 가 gateway 호출을 전담하고, `src/components/*.tsx` 는 그 함수만 부른다. 컴포넌트에서 직접 `fetch` 하지 않는다. UI는 React SPA로 구성되며, 탭/모달 기반 내비게이션을 사용한다.
- `public-server` — NestJS 모노레포. `apps/<서비스>` 가 실행 단위, `libs/<이름>` 이 공유 코드다. gateway 만 HTTP 를 열고 나머지는 gRPC 컨트롤러(`*.grpc-controller.ts`) 로만 열린다.
- `public-python-server` — 기능별 모듈 구조. 기능 디렉토리마다 `router.py`(입구), `service.py`/`application/*_use_case.py`(로직), `repository.py`(저장), `schemas.py`(계약), `dependencies.py`(조립)를 둔다. 공통 설정·DB·이벤트·보안은 `core/` 에 모인다. Kafka 소비자는 `<기능>/infrastructure/messaging/*_consumer.py` 다.

## 흐름
- **RAG 질의응답 (Kafka 잡 + SSE)**: `public-front/src/components/AiService.tsx:504` (`handleSendQuestion`) -> `public-front/src/api/ai.ts:327` (`askQuestionStream` — POST 후 SSE 구독) -> `public-server/apps/gateway/src/ai/job/job.controller.ts:40` (`createAskJob`, POST `/ai/rag/jobs`, Kafka 발행) -> Kafka `ai.rag.ask.requested` -> `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:91` (`_process` — 프롬프트 해석·검색·생성) -> `public-python-server/src/ai_service/core/events.py:20` (`publish_token`, Redis Streams `ai:job:{jobId}:events`) -> `public-server/apps/gateway/src/ai/stream/job-stream.controller.ts:39` (`stream`, GET `/ai/jobs/:jobId/stream` SSE 릴레이) -> 화면 렌더링
- **답변 생성 중단**: `public-front/src/components/AiService.tsx:482` (`handleCancelStreaming`) -> `public-front/src/api/ai.ts:343` (`cancel`) -> `public-server/apps/gateway/src/ai/job/job.controller.ts:74` (`cancelJob`, DELETE `/ai/jobs/:jobId`, Redis 취소 플래그) -> `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:162` (`_process` 루프에서 `job:{jobId}:cancelled` 확인 후 조기 종료)
- **문서 업로드 · 인제스트**: `public-front/src/components/AiService.tsx:325` (`handleFileUpload`) -> `public-front/src/api/ai.ts:124`, `:135` (`uploadDocument`, `subscribeIngestJob`) -> `public-server/apps/gateway/src/ai/knowledge/knowledge-job.controller.ts:70` (`createIngestJob`, POST `/ai/knowledge/jobs`, 멀티파트 수신, Redis 스테이징, Kafka 발행) -> Kafka `ai.knowledge.ingest.requested` -> `public-python-server/src/ai_service/knowledge/infrastructure/messaging/ingest_requested_consumer.py:82` (`_process`) -> `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py:98` (`execute` — 파싱·청킹·임베딩·Qdrant 저장) -> 같은 SSE 채널로 완료 통지
- **개인화 시스템 프롬프트**: `public-front/src/components/AiService.tsx:444` (`handleSavePrompt`) -> `public-front/src/api/ai.ts:204` (`saveMyPrompt`) -> `public-server/apps/gateway/src/ai/proxy/my-prompt-proxy.controller.ts:35` (`saveMine`, userId 를 인증 세션에서 채워 프록시) -> `public-python-server/src/ai_service/prompt/router.py:13` (`create`), `:35` (`activate`) -> `public-python-server/src/ai_service/prompt/service.py:38` (`get_active_prompt` — 개인 -> 전역 -> 기본값) -> `public-python-server/src/ai_service/prompt/repository.py:55` (`find_active_for_user`)
- **실시간 채팅 (WebSocket)**: `public-front/src/hooks/useChatSocket.ts:29` (`connectSocket`, Socket.IO 연결, `auth: { token }`) -> `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:63` (`handleConnection` — JWT 검증 및 user 컨텍스트 주입), `:89` (`handleJoinRoom` — gRPC 로 히스토리 조회), `:176` (`handleSendMessage`) -> `public-server/apps/chat-service/src/message/rpc/chat.grpc-controller.ts:13` (`SaveMessage`) -> `public-server/apps/chat-service/src/message/message.service.ts:16` (`persistAndNotify` — Redis ZSET 저장 + 샤드 Pub/Sub 발행) -> `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:121` (`onShardMessage` — ZSET pull 후 방에 emit)
- **결제 (gRPC)**: `public-front/src/components/Payment.tsx:113` (`handlePurchase`) -> `public-front/src/api/payment.ts:50` (`createPayment`) -> `public-server/apps/gateway/src/payment/payment-gateway.controller.ts:27` (`createPayment`) -> `public-server/apps/payment/src/payment/rpc/payment.grpc-controller.ts:17` (`createPayment`) -> `public-server/apps/payment/src/payment/application/create-payment.use-case.ts:14` (`execute`)
- **관리자 인증 · 유저 관리**: `public-front/src/api/admin.ts:3` (`adminSignup`) -> `public-server/apps/gateway/src/admin/admin-gateway.controller.ts:40` (`signup`, POST `/admin/auth/signup`) -> admin-server gRPC `:50054` -> `public-server/apps/admin-server/src/auth/admin-auth.grpc-controller.ts:33` (`signup`) -> `public-server/apps/admin-server/src/user/user.service.ts:62` (`signup`, Bull `mail` 큐 작업 등록)

## 확장 지점
- `public-python-server/src/ai_service/rag/rag_composition.py:20` (`RagComposition.create`) — 하이브리드 검색·RRF 융합·HyDE·리랭커·쿼리 분해기를 DI 로 조립한다. 검색 단계를 갈아 끼우는 자리.
- `public-python-server/src/ai_service/knowledge/infrastructure/providers/embedding_factory.py:11` (`create_embedding_provider`) — 임베딩 프로바이더를 환경변수로 고른다. Ollama·Google 이 이미 붙어 있어 추가가 쉽다.
- `public-python-server/src/ai_service/llm_gateway/infrastructure/providers/factory.py:10` (`create_chat_model`) — LLM 공급자(Groq·Anthropic·OpenAI·Ollama) 플러그인 지점.
- `public-server/apps/gateway/src/ai/proxy/` — ai-service-py 로 가는 HTTP 프록시 컨트롤러가 한 곳에 모여 있다. 파이썬 쪽 라우터를 하나 열면 여기에 컨트롤러 하나만 붙이면 된다.
- `public-server/apps/gateway/src/auth/gateway-auth.guard.ts:64` (`canActivate`) — JWT·Basic·API Key 세 갈래를 모두 `Session` 하나로 정규화한다. 새 인증 수단은 여기에만 붙인다.
- `public-front/src/api/client.ts:6` — Axios 인터셉터가 JWT 주입과 에러 처리를 중앙화한다.

## 아쉬운 곳
- **프롬프트 목록 조회에 사용자 격리가 없어 타인 프롬프트가 노출된다** — `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:33-35` (`list`)가 `GET /ai/prompts/:name` 호출 시 `public-python-server/src/ai_service/prompt/repository.py:44-47` (`find_all_by_name`)로 연결되는데, 이 쿼리에 `userId` 조건이 전혀 없어 다른 사용자의 개인 프롬프트가 전체 노출된다.
- **문서 인제스트 파이프라인의 중간 단계 상태가 전달되지 않는다** — `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py:98-124` (`execute`)가 파싱/청킹/임베딩/색인 과정을 단일 동기 루틴으로 처리하며 중간 이벤트를 발행하지 않고, `public-python-server/src/ai_service/knowledge/infrastructure/messaging/ingest_requested_consumer.py:82-96` (`_process`)도 완료(`publish_done`)와 오류(`publish_error`)만 발행한다. 업로드 중 진행률과 세부 단계를 화면에서 알 수 없다.
- **대화 기록(세션) 목록 조회 시 검색 및 질문 미리보기가 없다** — `public-python-server/src/ai_service/rag/router.py:12-21` (`list_sessions`) 및 `public-python-server/src/ai_service/rag/schemas.py:13-17` (`SessionOut`)은 `sessionId`, `title`, `updatedAt`만 반환하며 키워드 검색 쿼리 파라미터나 최근 질의 스니펫 필드가 없다. `public-front/src/components/AiService.tsx:684-740`에서도 제목 외에 검색 및 북마크 진입점이 없다.
- **AI 서비스 화면이 단일 컴포넌트에 과도하게 집중되어 있다** — `public-front/src/components/AiService.tsx:60-1126` (1126줄)에 문서 목록/업로드 모달, Q&A 스트리밍 채팅, 세션 히스토리 목록, 시스템 프롬프트 설정 모달이 모두 하나의 컴포넌트와 20여 개 상태(`useState`)로 뭉쳐 있어 허브형 내비게이션 및 대시보드 역할이 분리되지 않았다.
- **죽은 NestJS ai-service 가 남아 있다** — `public-server/apps/ai-service` 는 128개 파일이 남아 있으나 루트 `docker-compose.yml` 에 없고 역할은 `public-python-server` 가 가져갔다. 그런데도 `public-server/package.json:52` 의 `test:all` 이 `test:ai` 를 계속 돌린다.
- **admin-server 에 좁힌 테스트 게이트가 없다** — `public-server/apps/admin-server` 가 추가되었으나 `.specflow/config.yaml` 의 `scoped_test` 에 항목이 없고 `package.json:52`의 `test:all`에도 `test:admin` 이 포함되어 있지 않다. admin-server 변경 시 테스트가 검사받지 않는다.

## 건드리면 안 되는 곳
- `public-server/libs/auth/src/auth/strategy/`, `public-server/libs/auth/src/auth/auth.service.ts` — 토큰 발급·검증과 비밀키를 다루는 지점.
- `public-server/libs/common/src/databases/typeorm/migrations` — 운영 스키마 이력.
- `public-server/apps/payment/src/payment/application/create-payment.use-case.ts:14` (`execute`) 의 금액 계산 및 무결성 검증 경로 — 표시·전달은 대상이지만 금액을 만드는 곳은 아니다.
- `public-python-server/src/ai_service/rag/application/filter/secret_pii_scanner.py`, `public-python-server/src/ai_service/rag/application/filter/rag_content_validator.py` — 프롬프트 인젝션 방어와 개인정보·비밀값 마스킹 규칙.

<!-- specflow-survey generated=2026-09-02T01:02:34 front=0d1c2b041401 python-server=a6f057fe9731 server=afc9502932b2 -->

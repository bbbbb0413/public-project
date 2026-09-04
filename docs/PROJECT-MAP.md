# 코드 지도
<!--
이 문서는 조사(survey) 스테이지가 덮어쓴다. 손으로 고치지 않는다.
낡았다고 느끼면 고치는 대신 파이프라인을 돌린다 — 타깃 전체에
map_stale_commits 만큼 커밋이 쌓이면 기획 앞에서 자동으로 다시 만든다.

구조가 아니라 좌표를 담는 문서다. 서비스 경계·계약·역할은 README.md 가,
제품 방향은 PRODUCT.md 가 담당한다.
-->
## 저장소 구조 규약
- `public-front` — React 기반 SPA. `src/api/*.ts` 가 gateway REST/SSE API 호출을 전담하고, `src/components/*.tsx` 는 이 API 함수 및 훅을 호출한다. Socket.IO 실시간 통신은 `src/hooks/useChatSocket.ts` 및 `src/flatbuffers/` 가 담당한다.
- `public-server` — NestJS 모노레포 구조 (`apps/*`, `libs/*`). `apps/gateway` 만 외부 HTTP/WebSocket(Socket.IO) 진입점을 열고, 내부 도메인 서비스(`identity`, `payment`, `chat-service`, `admin-server`)와는 gRPC(`*.grpc-controller.ts`)로 통신한다. 공통 커널·인증·DB 유틸은 `libs/*` 에 위치한다.
- `public-python-server` — FastAPI 기반 비동기 마이크로서비스. 기능 모듈 디렉토리마다 `router.py`(HTTP 진입점), `service.py`/`application/*_use_case.py`(비즈니스 로직), `repository.py`/`infrastructure/persistence/*`(데이터 영속화), `schemas.py`(DTO/엔티티 계약), `dependencies.py`(DI 조립)를 둔다. 공통 설정·DB·보안은 `core/`에 모이고, Kafka 이벤트 소비자는 `<도메인>/infrastructure/messaging/*_consumer.py`에 위치한다.

## 흐름
- **RAG 질의응답 (Kafka 잡 + SSE)**: `public-front/src/components/AiService.tsx:561` (`handleSendQuestion`) -> `public-front/src/api/ai.ts:371` (`askQuestionStream`) -> `public-server/apps/gateway/src/ai/job/job.controller.ts:40` (`createAskJob`, POST `/ai/rag/jobs`, Kafka 발행) -> Kafka `ai.rag.ask.requested` -> `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:91` (`_process` — 프롬프트 해석·검색·생성) -> `public-python-server/src/ai_service/core/events.py:20` (`publish_token`, Redis Streams `ai:job:{jobId}:events`) -> `public-server/apps/gateway/src/ai/stream/job-stream.controller.ts:39` (`stream`, GET `/ai/jobs/:jobId/stream` SSE 릴레이) -> 프론트 화면 렌더링
- **답변 생성 중단**: `public-front/src/components/AiService.tsx:540` (`handleCancelStreaming`) -> `public-front/src/api/ai.ts:387` (`cancel`) -> `public-server/apps/gateway/src/ai/job/job.controller.ts:74` (`cancelJob`, DELETE `/ai/jobs/:jobId`) -> `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:162` (`_process` 루프에서 `job:{jobId}:cancelled` 키 확인 후 조기 종료)
- **문서 업로드 · 인제스트**: `public-front/src/components/AiService.tsx:314` (`handleFileUpload`) -> `public-front/src/api/ai.ts:80`, `:169` (`uploadDocument`, `subscribeIngestJob`) -> `public-server/apps/gateway/src/ai/knowledge/knowledge-job.controller.ts:70` (`createIngestJob`, POST `/ai/knowledge/jobs`, 멀티파트 수신, Redis 파일 스테이징, Kafka 발행) -> Kafka `ai.knowledge.ingest.requested` -> `public-python-server/src/ai_service/knowledge/infrastructure/messaging/ingest_requested_consumer.py:82` (`_process`) -> `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py:86` (`execute` — 파싱·청킹·임베딩·Qdrant 저장) -> Redis Streams SSE 완료 통지
- **개인화 시스템 프롬프트 관리**: `public-front/src/components/AiService.tsx:437` (`refreshPromptData`, `handleSavePromptSlot`) -> `public-front/src/api/ai.ts:251`, `:259` (`saveMyPrompt`, `activateMyPrompt`) -> `public-server/apps/gateway/src/ai/proxy/my-prompt-proxy.controller.ts:57` (`saveMine`), `:82` (`activateMine`) -> `public-python-server/src/ai_service/prompt/router.py:13` (`create`), `:35` (`activate`) -> `public-python-server/src/ai_service/prompt/service.py:38` (`get_active_prompt` — 개인 -> 전역 -> 기본값) -> `public-python-server/src/ai_service/prompt/repository.py:55` (`find_active_for_user`)
- **실시간 채팅 (WebSocket + FlatBuffers)**: `public-front/src/hooks/useChatSocket.ts:29` (`connectSocket`, Socket.IO 연결, `auth: { token }`) -> `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:63` (`handleConnection` — JWT 검증 및 user 컨텍스트 주입), `:89` (`handleJoinRoom` — gRPC로 히스토리 조회), `:205` (`handleSendMessage`) -> `public-server/apps/chat-service/src/message/rpc/chat.grpc-controller.ts:13` (`saveMessage`) -> `public-server/apps/chat-service/src/message/message.service.ts:16` (`persistAndNotify` — Redis ZSET 저장 + 샤드 Pub/Sub 발행) -> `public-server/apps/gateway/src/chat/chat-gateway.gateway.ts:146` (`onShardMessage` — ZSET pull 후 방에 FlatBuffers 바이너리 emit)
- **결제 생성 및 승인 (gRPC)**: `public-front/src/components/Payment.tsx:113` (`handlePurchase`) -> `public-front/src/api/payment.ts:43` (`createPayment`) -> `public-server/apps/gateway/src/payment/payment-gateway.controller.ts:41` (`createPayment`) -> `public-server/apps/payment/src/payment/rpc/payment.grpc-controller.ts:24` (`createPayment`) -> `public-server/apps/payment/src/payment/application/create-payment.use-case.ts:27` (`execute` — 멱등성 락, PG 승인 및 백오프 재시도)
- **관리자 인증 · 회원가입 (gRPC + Bull Mail Queue)**: `public-front/src/api/admin.ts:44` (`adminSignup`) -> `public-server/apps/gateway/src/admin/admin-gateway.controller.ts:57` (`signup`, POST `/admin/auth/signup`) -> admin-server gRPC `:50054` -> `public-server/apps/admin-server/src/auth/admin-auth.grpc-controller.ts:33` (`signup`) -> `public-server/apps/admin-server/src/user/user.service.ts:62` (`signup`, Bull `mail` 큐 작업 등록)

## 확장 지점
- `public-python-server/src/ai_service/rag/rag_composition.py:72` (`build_rag_composition`) — 하이브리드 검색, RRF 융합, HyDE, 리랭커, 쿼리 분해기를 DI 로 조립한다. 검색 및 생성 파이프라인 단계를 갈아 끼우는 자리.
- `public-python-server/src/ai_service/knowledge/infrastructure/providers/embedding_factory.py:21` (`build_embedding_provider`) — 임베딩 모델 제공자를 환경변수 설정에 따라 선택(OpenAI, Google Gemini, Ollama 등).
- `public-python-server/src/ai_service/llm_gateway/infrastructure/providers/factory.py:15` (`build_llm_provider`) — LLM 제공자(Groq, Anthropic, OpenAI, Ollama, Gemini) 플러그인 팩토리 지점.
- `public-server/apps/gateway/src/ai/proxy/` — Python AI 서비스(`ai-service-py`)로 전달되는 HTTP 프록시 컨트롤러들이 집중된 곳. 새 AI 엔드포인트 추가 시 여기에 게이트웨이 프록시 컨트롤러만 추가하면 된다.
- `public-server/apps/gateway/src/auth/gateway-auth.guard.ts:64` (`canActivate`) — JWT, Basic, API Key 인증 요청을 단일 `Session` 객체로 정규화하는 보안 게이트 지점.
- `public-front/src/api/client.ts:10-36` — Axios 인터셉터가 JWT 자동 주입 및 401 토큰 만료 처리/에러 핸들링을 중앙 집중식으로 수행.

## 아쉬운 곳
- **프롬프트 버전 목록 조회에 사용자 격리 누락 (타인 프롬프트 노출)** — `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:48-60` (`list`)에서 `GET /ai/prompts/:name` 요청 시 세션 `userId`를 파라미터로 넘기더라도, `public-python-server/src/ai_service/prompt/router.py:19-23` (`list_versions`) 및 `public-python-server/src/ai_service/prompt/service.py:73-74` (`list_versions`), `public-python-server/src/ai_service/prompt/repository.py:44-47` (`find_all_by_name`)가 `userId` 필터 없이 전체 사용자의 프롬프트를 조회하여 타인의 커스텀 프롬프트가 노출된다.
- **문서 인제스트 파이프라인의 진행률/세부 단계 이벤트 미발행** — `public-python-server/src/ai_service/knowledge/application/ingest_document_use_case.py:86-122` (`execute`)가 파싱/검증/청킹/임베딩/색인 과정을 단일 동기 루틴으로 수행하며 중간 진행 이벤트를 발행하지 않고, `public-python-server/src/ai_service/knowledge/infrastructure/messaging/ingest_requested_consumer.py:82-99` (`_process`)도 최종 완료(`publish_done`)와 오류(`publish_error`)만 발행한다. 프론트엔드(`public-front/src/components/AiService.tsx:368-374`)는 단계별 진행률 수신 준비가 되어 있으나 백엔드에서 전달되지 않는다.
- **대화 세션 목록 조회 시 키워드 검색 및 북마크/필터 기능 부재** — `public-python-server/src/ai_service/rag/router.py:16-25` (`get_sessions`) 및 `public-python-server/src/ai_service/rag/schemas.py:359-373` (`SessionOut`)은 `sessionId`, `title`, `updatedAt`만 페이징으로 반환하며 질의 키워드 검색 파라미터나 중요 대화 북마크 필드가 없다. `public-front/src/components/AiService.tsx:760-797`의 세션 사이드바 UI에서도 검색/필터 기능이 없다.
- **AI 서비스 화면이 단일 거대 컴포넌트에 과도하게 집중** — `public-front/src/components/AiService.tsx:69-1319` (1300여 줄)에 문서 업로드/목록/삭제, Q&A 스트리밍 채팅, 세션 히스토리 사이드바, 다중 슬롯 시스템 프롬프트 설정 모달이 25개 이상의 로컬 상태(`useState`)와 함께 하나의 컴포넌트에 몰려 있어 허브형 내비게이션/대시보드 구조로의 분리가 필요하다.
- **폐기된 레거시 NestJS ai-service 설정 잔존 및 테스트 부하** — `public-server/apps/ai-service`는 Python 서버로 역할이 완전 이관되어 `docker-compose.yml`에서 제외되었으나, `public-server/package.json:52`의 `test:all` 스크립트에 `test:ai`가 포함되어 불필요한 테스트를 실행하며, `public-server/nest-cli.json:101-109` 및 `.specflow/config.yaml:76`에 레거시 프로젝트 설정이 잔존해 있다.
- **admin-server 단위 테스트 스위트 게이트 누락** — `public-server/apps/admin-server`에 `src/user/user.service.spec.ts:1-271` 단위 테스트가 작성되어 있으나, `public-server/package.json:52`의 `test:all` 및 `.specflow/config.yaml:75-81`의 `scoped_test`에 `admin-server`가 등록되어 있지 않아 CI/CD 파이프라인에서 테스트 검증이 누락된다.

## 건드리면 안 되는 곳
- `public-server/libs/auth/src/auth/strategy/`, `public-server/libs/auth/src/auth/auth.service.ts:20-43` — JWT 토큰 발급·검증, API 키 검증 및 세션 검증 보안 로직.
- `public-server/libs/common/src/databases/typeorm/migrations/` — 운영 데이터베이스 스키마 마이그레이션 이력.
- `public-server/apps/payment/src/payment/application/create-payment.use-case.ts:27-70` (`execute`) — 결제 멱등성 락, 결제 금액 무결성 검증 및 PG 승인 연동 로직.
- `public-python-server/src/ai_service/rag/application/filter/secret_pii_scanner.py:11-32`, `public-python-server/src/ai_service/rag/application/filter/rag_content_validator.py:9-40` — 프롬프트 인젝션 방어 패턴 및 개인정보·비밀키(AWS/OpenAI/RRN 등) 마스킹 검증 규칙.

<!-- specflow-survey generated=2026-09-05T01:03:09 front=80003d66b585 python-server=a6f057fe9731 server=af53e78ba899 -->

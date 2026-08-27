---
id: SPEC-019
title: 결제 서비스 대규모 트래픽 처리 아키텍처 로드맵 (멱등성 / 이벤트 / PG 연동 / 성능)
status: done
targets: [server]
stages: [backend]
priority: high
---

## 배경 / 문제
현재 결제 서비스(`public-server/apps/payment`)는 게이트웨이 → gRPC → `CreatePaymentUseCase` → MySQL 단건 INSERT로 이어지는 완전 동기 구조이며, 대규모 트래픽 환경에서 다음과 같은 구조적 한계를 가진다.

- **멱등성 부재**: 동일 요청이 재시도되면 중복 결제 레코드가 그대로 생성된다 (`public-server/apps/payment/src/payment/application/create-payment.use-case.ts:14-24`).
- **상태 개념 부재**: `Payment` 도메인 모델(`public-server/apps/payment/src/payment/domain/model/payment.ts:4-14`)과 `PaymentOrmEntity`(`public-server/apps/payment/src/payment/infrastructure/orm/payment.orm-entity.ts:9-27`)에 `status` 필드가 없어, gRPC 응답은 실제 처리 결과와 무관하게 문자열 `'COMPLETED'`로 하드코딩되어 있다 (`public-server/apps/payment/src/payment/rpc/payment.grpc-mapper.ts:22`).
- **후속 처리 결합 경로 없음**: 이벤트 발행 계층이 없어 정산/알림/재고 차감 같은 기능을 추가할 때마다 `CreatePaymentUseCase`가 계속 비대해지는 구조다.
- **PG(결제대행사) 연동 계층 부재**: 외부 결제사 승인·웹훅·재시도를 수용할 자리가 없다.
- **조회가 항상 DB 직접 조회**: 트래픽이 몰리면 결제 생성과 조회가 같은 커넥션 풀을 두고 경쟁한다.

토스페이먼츠·카카오페이·쿠팡/아마존류 대형 커머스의 공개 사례를 조사한 결과, 공통적으로 (1) 멱등키, (2) Outbox + 이벤트 기반 후속 처리 분리, (3) append-only 원장과 상태 머신, (4) PG 웹훅 + 재시도 + 주기적 대사(reconciliation), (5) 조회 경로 캐싱과 트랜잭션 범위 최소화 패턴을 사용한다. 이 문서는 그 결론을 현재 레포 구조(Kafka는 `gateway`/`ai-service`에 이미 연동되어 있고, Redis는 세션/캐시로 이미 연동되어 있으며, `payment` MySQL DB는 이미 독립되어 있음)에 맞춰 적용할 로드맵을 정리한다.

이 문서는 로드맵이자 진행 기록이다. SpecFlow 파이프라인이 자동 실행하는 대상이 아니라, Claude가 직접 Phase 단위로 구현하고 완료 시 해당 Phase를 `done`으로 표시해 기록으로 남긴다. Phase는 앞 단계가 완료되어야 다음 단계의 전제가 성립하므로 순서대로 진행한다.

## 요구사항

### Phase 1 — 멱등성 키 + 결제 상태 머신 (선행, 최우선) — **done (2026-08-28)**
- [x] `Payment` 도메인/ORM에 `status`(`PENDING` | `COMPLETED` | `FAILED` 등) 필드를 추가하고, 상태 전이를 도메인 메서드로 캡슐화한다.
- [x] `CreatePaymentRequest`(gRPC)와 `CreatePaymentInDto`(HTTP)에 `idempotencyKey`를 추가하고, Redis에 TTL과 함께 저장해 동일 키로 재요청이 오면 새 레코드를 만들지 않고 기존 처리 결과를 그대로 반환한다.
- [x] `payment.proto`의 `PaymentReply.status` 하드코딩을 제거하고, 매퍼가 실제 상태 값을 반환하도록 수정한다.
- [x] 상태 필드 추가에 따른 리포지토리/매퍼/단위 테스트를 갱신한다.

### Phase 2 — Outbox 패턴 + Kafka 이벤트 발행 — **done (2026-08-28)**
- [x] `payment` DB에 outbox 테이블을 추가하고, 결제 상태 변경을 같은 트랜잭션 안에서 outbox 레코드로 함께 저장한다.
- [x] `gateway`의 Kafka producer 패턴(`ai-kafka-producer.service.ts`, `ai-kafka-client.module.ts`)을 참고해 `payment` 서비스 전용 Kafka producer를 구성하고, `payment.events`(가칭) 토픽에 상태 변경 이벤트를 발행한다.
- [x] outbox → Kafka 발행을 담당하는 릴레이(폴러 또는 워커)를 추가하고, 발행 실패 시 재시도되도록 한다.

### Phase 3 — PG 연동 대비 (웹훅 수신 + 재시도/대사) — **done (2026-08-28)**
- [x] 외부 PG 승인 결과를 비동기로 수신하는 웹훅 엔드포인트를 추가하고, 서명 검증과 멱등 처리를 적용한다.
- [x] PG 승인 요청에 지수 백오프 기반 재시도를 적용한다.
- [x] 매일 1회 내부 결제 상태와 PG 거래 내역을 대사(reconciliation)하는 배치 잡을 추가하고, 불일치 건을 로깅/알림한다.
- 참고: 실제 PG사가 아직 확정되지 않은 경우, 이 Phase는 어댑터 인터페이스 설계까지만 먼저 진행하고 구현은 PG 확정 이후 착수한다. → 사용자와 상의해 **어댑터 인터페이스 + Mock PG 구현체**까지 진행하기로 범위를 확정함(실제 PG는 여전히 미확정).

### Phase 4 — 조회 캐싱 및 트랜잭션 최적화 — **done, 범위는 부하테스트 결과로 축소함 (2026-08-28)**
- [ ] 결제 단건 조회(`GetPayment`)를 Redis 캐시 우선 조회로 전환하고, 상태 변경 시 캐시를 무효화한다. → **보류**. 부하테스트가 쓰기 경로만 다뤘고 읽기 경합이 실제로 문제라는 증거가 아직 없어서, 근거 없이 구현하지 않음(아래 "남겨둔 것" 참고).
- [x] 외부 호출(PG 등)이 포함된 구간을 DB 트랜잭션 범위 밖으로 분리한다. → Phase 3에서 이미 구조적으로 만족됨: `CreatePaymentUseCase`에서 PG 승인 호출은 `persist(pending)`과 `persistWithEvents(outcome, events)` 두 트랜잭션 사이, 트랜잭션 밖에서 일어남. 코드 확인으로 완료 처리.
- [x] 실제 부하 테스트로 병목 구간을 확인한 뒤에만 착수한다 (조기 최적화 지양). → k6로 실측함. 아래 기록 참고.

## 비요구사항 (Out of scope)
- 실제 PG사 선정 및 계약/정산 수수료 정책 결정
- 결제 취소/환불 도메인 로직 신규 설계 (별도 SPEC에서 다룬다)
- `payment` 외 도메인(정산, 재고, 알림) 서비스의 내부 구현
- 성능 목표 TPS 수치 확정 (실측 후 별도 문서화)

## 수용 기준 (로드맵 완료 기준)
- Given 동일한 `idempotencyKey`로 결제 요청이 2회 연속 들어왔을 때 When 두 번째 요청이 처리되면 Then 결제 레코드가 1건만 생성되고 두 응답이 동일한 결제 결과를 반환한다.
- Given 결제 상태가 `COMPLETED`로 변경됐을 때 When 트랜잭션이 커밋되면 Then `payment.events` 토픽에 해당 상태 변경 이벤트가 정확히 1회 발행된다.
- Given PG 웹훅이 네트워크 문제로 중복 전달됐을 때 When 웹훅이 재처리되면 Then 결제 상태가 중복 갱신되지 않는다.
- Given 결제 단건 조회가 상태 변경 없이 반복 호출될 때 When 두 번째 이후 요청이 들어오면 Then 응답이 DB가 아닌 캐시에서 반환된다.

## 참고
- 고쳐야/추가해야 할 자리: `public-server/apps/payment/src/payment/domain/model/payment.ts:4-14` (status 필드 없음)
- 고쳐야/추가해야 할 자리: `public-server/apps/payment/src/payment/infrastructure/orm/payment.orm-entity.ts:9-27`
- 고쳐야/추가해야 할 자리: `public-server/apps/payment/src/payment/rpc/payment.grpc-mapper.ts:22` (status 하드코딩)
- 고쳐야/추가해야 할 자리: `public-server/apps/payment/src/payment/application/create-payment.use-case.ts:14-24`
- Kafka producer 참고 패턴: `public-server/apps/gateway/src/ai/kafka/ai-kafka-producer.service.ts`, `public-server/apps/gateway/src/ai/kafka/ai-kafka-client.module.ts`
- Redis 연동 참고: `public-server/libs/common/src/databases/redis/abstract-redis.repository.ts`
- 리서치 근거: Toss Tech "레거시 결제 원장을 확장 가능한 시스템으로" (https://toss.tech/article/payments-legacy-5), 카카오페이 기술 블로그 "온라인 결제 서비스 2.5배 성능 개선기" (https://tech.kakaopay.com/post/improve-service-performance/), Team JSON Delivery "쿠팡, 아마존, 알리와 같은 기업들은 결제 시스템을 어떻게 만드는걸까?" (https://team-json-delivery.github.io/posts/pay-system/)

## 구현 기록

### Phase 1 (2026-08-28, done)
- `payment.proto`에 `idempotency_key` 필드를 추가하고 `npm run proto:gen`으로 `libs/rpc/src/generated/payment.ts`를 재생성함.
- 도메인: `Payment`에 `status`(`PaymentStatus` enum: `PENDING`/`COMPLETED`/`FAILED`)를 추가하고, `complete()`/`fail()` 상태 전이 메서드를 도메인 메서드로 캡슐화(PENDING에서만 전이 허용, 위반 시 예외). `PaymentOrmEntity`에 `status` 컬럼 추가, `PaymentMapper` 양방향 매핑 갱신.
- 멱등성: `IPaymentIdempotencyRepository` 포트 + Redis 구현체(`PaymentIdempotencyRepositoryImpl`, db 4, TTL 24시간)를 추가. 단순 조회-후-생성이 아니라 `SET NX`로 처리를 선점(`tryClaim`)하는 방식을 써서, 동시에 같은 키로 들어오는 요청이 각각 다른 레코드를 만드는 경쟁 상태까지 막음. 이미 완료된 키는 기존 결과를 그대로 반환하고, 다른 요청이 처리 중인 키는 `ConflictException`(409)을 던짐.
- `CreatePaymentUseCase`는 이제 결제를 PENDING으로 저장한 뒤 COMPLETED로 갱신하는 2단계로 처리함 — 이 사이에 Phase 3에서 실제 PG 승인 호출이 들어갈 자리.
- gRPC(`payment.grpc-mapper.ts`)와 HTTP(내부 `CreatePaymentInDto`/게이트웨이 `CreatePaymentDto`) 양쪽에 `idempotencyKey`를 관통시키고, 게이트웨이 응답의 `status` 하드코딩을 제거함.
- 사이드이펙트: 게이트웨이가 `idempotencyKey`를 필수로 받게 되어, 프론트엔드 `public-front/src/api/payment.ts`의 `createPayment`가 호출마다 `crypto.randomUUID()`로 키를 생성해 함께 전송하도록 수정함(그렇지 않으면 기존 프론트가 400을 받음). 관련 프론트 테스트(`payment.test.ts`, `Payment.test.tsx`) 갱신.
- 검증: `apps/payment` 유닛 테스트(`test:payment`, 3 suites / 16 tests), `apps/gateway` 유닛 테스트(`test:gateway`, 5 suites / 37 tests), 프론트 `payment.test.ts` + `Payment.test.tsx` (2 files / 3 tests), `tsc --noEmit`(payment, gateway) 모두 통과.
- 남겨둔 것: outbox/커밋 실패 시 idempotency 키가 `PROCESSING` 상태로 TTL(24h) 동안 묶이는 문제는 Phase 2(outbox+이벤트)에서 트랜잭션 보장과 함께 다룸. PG 실제 승인 호출은 아직 없음(Phase 3).

### Phase 2 (2026-08-28, done)
- `Payment` 도메인에 `complete()`/`fail()` 전이 시 각각 `PaymentCompletedEvent`/`PaymentFailedEvent`를 발생시키도록 `AggregateRoot.addDomainEvent`를 사용함(`domain/event/payment-completed.event.ts`, `payment-failed.event.ts`).
- `payment_outbox` 테이블(`PaymentOutboxOrmEntity`: aggregateId, eventType, payload, status, attempts, publishedAt)을 추가하고, `IPaymentRepository.persistWithEvents(payment, events)`가 결제 상태 저장과 outbox insert를 `this.manager.transaction(...)` 하나의 DB 트랜잭션으로 묶음. 기존 레포에 있던 CLS 기반 `@Transactional`/`TypeOrmHelper` 데코레이터는 payment 앱에 전혀 연결돼 있지 않고(다른 서비스에서도 실사용처가 없음) 전역 인터셉터 배선이 추가로 필요해서, 이번 phase 범위에서는 그 대신 TypeORM `EntityManager.transaction()`을 직접 써서 payment 모듈 안에서 완결되게 구현함.
- `CreatePaymentUseCase`가 이제 `pending.complete()` → `pullDomainEvents()` → `persistWithEvents(completed, events)` 순서로 동작함. 즉 상태 변경과 outbox 적재가 원자적으로 커밋됨.
- `gateway`의 `ai-kafka-producer.service.ts` 패턴을 그대로 따라 `payment` 전용 Kafka client/producer(`PaymentKafkaClientModule`, `PaymentKafkaProducerService`)를 추가하고 `payment.events` 토픽에 `{ eventType, payload }`를 발행함. 다만 결제 서비스가 Kafka 하나 죽었다고 통째로 못 뜨면 안 되므로, `onModuleInit`의 `client.connect()` 실패는 warning 로그만 남기고 넘어가도록 함(이게 바로 outbox가 존재하는 이유이기도 함 — 발행 실패는 릴레이가 재시도로 흡수).
- `PaymentOutboxRelayService`를 추가함: `setInterval`(기본 2초, `PAYMENT_OUTBOX_POLL_INTERVAL_MS`로 조정 가능)로 PENDING outbox 행을 최대 50건씩 배치 조회해 Kafka로 발행하고, 성공하면 PUBLISHED로, 실패하면 attempts를 늘려 PENDING 유지(다음 폴링에서 재시도)하다가 5회(`MAX_ATTEMPTS`) 넘으면 FAILED로 표시함. `@nestjs/schedule`처럼 이 레포에서 안 쓰이는 새 의존성을 추가하는 대신 가벼운 `setInterval` 폴러로 구현함.
- 검증: `apps/payment` 유닛 테스트 전체(`test:payment`, 5 suites / 24 tests — 신규 `payment.spec.ts`(도메인 이벤트), `payment-outbox-relay.service.spec.ts`(발행 성공/실패/재시도/중첩폴링 방지) 포함) 통과, `tsc --noEmit` 통과, `npm run proto:gen`으로 `idempotency_key` 필드 재생성 확인.
- 확인 못한 것: `npm run test:e2e:payment`를 로컬 MySQL/Redis/Kafka(모두 기동 중인 docker 컨테이너) 대상으로 돌려봤으나, e2e 스펙에 하드코딩된 DB 자격증명(`admin`/`personal!23`)이 실제 컨테이너 계정과 달라 `Access denied` → TypeORM 재시도 루프로 이어지는 기존 환경 문제로 실패함. Phase 1 때도 동일하게 e2e는 실행되지 않았던 것으로 보이며, 이번 Phase 2 변경과는 무관한 사전 존재 이슈로 판단됨(`.env` 파일은 정책상 직접 열어보지 않음). 실제 DB에 대고 outbox → Kafka 발행 전체 경로를 도는 것은 아직 미검증 상태.

### e2e 계정 문제 정리 (2026-08-28)
- `payment.e2e-spec.ts`가 `PERSONAL_DB_USER_NAME`/`PERSONAL_DB_USER_PW`(및 GAME_DB, PAYMENT_DB 동일 항목)를 `'admin'`/`'personal!23'`로 무조건 하드코딩하고 있었음. `docker-compose.yml`의 `db` 컨테이너는 `${DB_USER}`/`${DB_USER_PW}`(`.env`)로 유저를 생성하므로, `.env`가 바뀌면 이 하드코딩 값과 실제 계정이 어긋날 수 있는 구조였음.
- gateway의 `gateway.e2e-spec.ts`가 이미 쓰고 있던 `process.env.X || 'default'` 패턴을 그대로 가져와, `PERSONAL_DB_USER_NAME = process.env.DB_USER || 'admin'` 식으로 고침. 실제 환경에 `DB_USER`/`DB_USER_PW`가 export돼 있으면 그 값을 쓰고, 없으면 기존 하드코딩 값으로 폴백해 하위 호환을 유지함.
- 사용자가 알려준 실제 계정(`admin` / 비밀번호는 채팅으로 직접 전달받음, 여기 기록하지 않음)을 이 세션에 `export`해서 다시 테스트한 결과: **자격증명 자체는 문제가 아니었음**을 확인함. `mysql2`로 직접 `createConnection`/`createPool`(TypeORM이 쓰는 방식과 동일) 양쪽 다 정상 연결됨.
- 그런데도 `npm run test:e2e:payment`는 여전히 `TypeError: authSwitch.authSwitchRequestMoreData is not a function`로 실패함. 원인을 좁혀본 결과: mysql2 버전은 하나뿐(`mysql2@3.22.4`, 중복 설치 아님)이고, `jest-setup.js`의 `Buffer.SlowBuffer` 패치를 재현해도 재현되지 않음 — 즉 자격증명도, mysql2 자체도 원인이 아니고, `typeorm`(package.json에 `"typeorm": "^1.0.0"`로 고정돼 있음 — 실제 TypeORM 배포 이력에 없는 버전대라 사설/포크 패키지일 가능성이 있음)과 Jest/ts-jest 실행 환경이 맞물릴 때만 발생하는 문제로 보임. 이건 이번 Phase 2 outbox 작업이나 결제 도메인과 무관하게, **이 레포의 TypeORM e2e 테스트 인프라 자체에 있는 사전 존재 결함**으로 판단하고 이번 로드맵 범위 밖으로 남겨둠.
- **정정**: `authSwitch.authSwitchRequestMoreData is not a function`는 TypeORM/mysql2/Jest 드라이버 버그가 **아니었음**. 파고들어보니 진짜 원인은 `payment-bc.module.ts`의 실수였음 — `PaymentOutboxRepositoryImpl`이 `TypeOrmExModule.forFeatures([...])`(DB `manager`/`target`/`queryRunner`를 제대로 주입하는 provider)와, `providers` 배열에 클래스를 그냥 나열한 것(Nest가 `useClass`로 해석해 `new PaymentOutboxRepositoryImpl(undefined, undefined, undefined)`를 생성) 두 곳에 중복 등록돼 있었고, 같은 모듈의 로컬 `providers`가 `imports`로 들어온 provider를 덮어써서 고장난 쪽이 이겼음. `this.manager`가 `undefined`가 된 채로 `super.metadata` getter(`this.manager.dataSource...`)가 호출되면서 그 에러가 난 것이었고, `authSwitch...` 로그는 이 실패 이후 백그라운드에 남아있던 이전 연결 재시도의 노이즈였을 뿐, DB 접속과는 무관했음.
- `payment-bc.module.ts`의 `providers`에서 중복된 `PaymentOutboxRepositoryImpl,` 한 줄을 제거해서 고침. 이 과정에서 raw `mysql2`(plain connection/pool), TypeORM `DataSource.initialize()` 단독, 2개 DataSource 동시 초기화, DB+Kafka client 동시 등록까지 단계별로 재현 스크립트를 만들어 하나씩 제거법으로 좁혔음(모두 `apps/payment/test/_debug-*.e2e-spec.ts`에 임시로 만들었다가 확인 후 삭제함, 커밋에는 없음).
- 고친 뒤 `npm run test:e2e:payment` 전체 통과 확인(`app.close()`가 Kafka producer/Redis 클라이언트/outbox 릴레이 정리 때문에 5s 기본 hook 타임아웃을 넘겨서 `afterAll` 타임아웃을 15s로 늘림).
- **추가로 outbox → Kafka 발행 전체 경로를 진짜로 검증함**: 실행 중인 `PaymentModule`에 `POST /payment`로 실제 결제를 생성하고, kafkajs로 `payment.events` 토픽을 직접 구독해서 릴레이가 발행한 실제 메시지를 수신 확인함 — `{"eventType":"payment.completed","payload":{"paymentId":5,"userId":100,"amount":5000,"currency":"KRW","productId":"debug-product","occurredAt":"..."}}`가 결제 생성 후 약 3초 안에 수신됨(릴레이 폴링 주기 2초와 일치). 결론: outbox 트랜잭션 적재 → 릴레이 폴링 → Kafka 발행까지 실제 인프라로 확인 완료.
- 결론: e2e "계정" 문제도, 그 뒤에 숨어있던 진짜 버그(Phase 2에서 내가 만든 provider 중복 등록 실수)도 모두 해결했고, 실제 outbox→Kafka 전체 경로까지 실환경에서 검증 완료. 더 이상 "미검증" 항목 없음.

### Phase 3 (2026-08-28, done)
- 범위 확정: 실제 PG사는 여전히 미정이라, 사용자와 상의해 **어댑터 인터페이스 + Mock PG 구현체**까지 진행하기로 함(순수 인터페이스만 설계하는 옵션 대신). 실제 PG가 정해지면 `IPgAdapter`를 구현하는 어댑터 하나만 추가로 만들면 되는 구조.
- `domain/port/pg-adapter.port.ts`: `IPgAdapter` 포트 정의 — `requestApproval`(승인 요청), `listTransactions`(대사용 PG 거래 조회), `verifyWebhookSignature`(웹훅 서명 검증). PG마다 웹훅 서명 방식이 다르므로 서명 검증도 어댑터 책임으로 둠.
- `infrastructure/pg/mock-pg.adapter.ts`: `MockPgAdapter` — Redis(db 5)에 자기 자신의 "PG 쪽 원장"을 남겨서, 대사 배치가 비교할 진짜 대상이 있게 함. 승인 여부는 랜덤이 아니라 **결정적**(`amount === 999999`면 항상 거절)이라 실패 경로를 재현 가능하게 테스트할 수 있음. 웹훅 서명은 HMAC-SHA256(`${pgTransactionId}.${paymentId}.${status}`)으로 계산 — 진짜 raw-body 서명이 아니라 필드 조합 기반의 단순화된 버전임(실제 PG 붙일 때는 그 PG의 서명 스펙을 그대로 따라야 함, 예를 들어 raw body 전체에 대한 HMAC일 수 있음).
- `infrastructure/pg/backoff-retry.util.ts`: 지수 백오프 재시도 유틸(`retryWithBackoff`). `CreatePaymentUseCase`가 PG 승인 요청을 이걸로 감싸서 최대 3회, 200ms→400ms 간격으로 재시도하고 그래도 실패하면 `pending.fail()`로 처리함.
- `CreatePaymentUseCase` 흐름 변경: PENDING 저장 → **PG 승인 요청(재시도 포함)** → 승인이면 `complete()`, 거절/영구실패면 `fail()` → outbox와 함께 저장. Phase 1에서 남겨뒀던 "이 사이에 실제 PG 승인 호출이 들어갈 자리"가 이제 채워짐.
- `POST /payment/webhook/pg` 웹훅 엔드포인트(`PgWebhookController` + `HandlePgWebhookUseCase`) 추가: `x-pg-signature` 헤더로 서명 검증(실패 시 401) → 결제 조회 → **이미 PENDING이 아니면(=이미 처리됨) 아무 것도 안 하고 조용히 리턴**해서 멱등성을 보장함(별도 웹훅 전용 dedupe 저장소를 새로 만들지 않고, Phase 1의 상태 머신 불변조건을 그대로 재사용).
- `infrastructure/reconciliation/payment-reconciliation.service.ts`: `setInterval` 기반(기본 24시간, `PAYMENT_RECONCILE_INTERVAL_MS`로 조정 가능)으로 최근 24시간 PG 거래 내역과 내부 결제 상태를 비교해 불일치를 ERROR 레벨로 로깅함. 이 레포에 Slack/이메일 같은 알림 채널이 아직 없어서 "알림"은 로깅까지만 함. PG→내부 방향 불일치(PG엔 있는데 내부에 없음/상태 다름)만 검사하고, 반대 방향(내부엔 있는데 PG엔 없음 — 예: 영원히 PENDING에 갇힌 결제)은 `findRecentPayments` 같은 조회 메서드가 아직 없어 이번 범위에서 뺌.
- `@nestjs/schedule` 같은 새 의존성 없이, Phase 2 outbox 릴레이와 같은 스타일(`setInterval`)로 재사용함. 다만 이건 "앱이 계속 떠 있는 동안 N시간마다"이지, 정확한 벽시계 기준(매일 03:00) 크론은 아님 — 필요해지면 별도 스케줄러를 붙여야 함.
- 검증: `apps/payment` 유닛 테스트 전체(`test:payment`, 9 suites / 39 tests — 신규 `backoff-retry.util.spec.ts`, `handle-pg-webhook.use-case.spec.ts`, `pg-webhook.controller.spec.ts`, `payment-reconciliation.service.spec.ts` 포함) 통과, `tsc --noEmit` 통과.
- **실제 인프라로 end-to-end 검증**(임시 `_debug-phase3.e2e-spec.ts`로 확인 후 삭제, 커밋에는 없음): `POST /payment`로 결제 생성 시 PG가 승인하면 응답이 `COMPLETED`, `amount=999999`(거절 sentinel)면 `FAILED`로 실제로 나옴. 웹훅에 잘못된 서명을 보내면 401. 이미 COMPLETED인 결제에 같은 승인 웹훅을 연달아 두 번 보내도 둘 다 정상 응답(에러 없이 멱등하게 무시)함을 확인함.
- 남겨둔 것: 실제 PG사 선정 및 그 PG의 실제 서명 스펙 적용(현재는 필드 조합 HMAC으로 단순화), 반대 방향 대사(내부에만 있고 PG엔 없는 결제 탐지), 실제 알림 채널(Slack 등) 연동, 정확한 벽시계 기준 스케줄러.

### Phase 4 (2026-08-28, done — 범위는 부하테스트 결과로 축소함)
- 부하테스트 도구로 k6를 새로 설치함(`brew install k6`, npm 의존성 아님). 스크립트는 `public-server/loadtest/create-payment.js`에 커밋해둠 — VU/반복마다 `idempotencyKey`를 새로 만들어서 Redis 멱등성 캐시 히트로 인한 가짜 처리량을 방지함.
- 관측 공백을 먼저 메움: `payment.module.ts`에 `http_request_duration_seconds` 히스토그램(`HttpMetricsInterceptor`, method/route/status 라벨)을 추가함. 기존엔 `/metrics`에 프로세스 지표(메모리/GC)만 있어서 라우트별 지연시간을 볼 방법이 없었음.
- 테스트 대상: 로컬에서 직접 빌드해서 돌린 payment 인스턴스(포트 8091, 이번 로드맵의 Phase 1~3 코드가 다 들어간 버전). **주의**: docker-compose로 떠 있는 `payment` 컨테이너(8081)는 이 세션에서 `--build` 없이 재시작만 했기 때문에, 여전히 Phase 1 이전의 예전 이미지로 떠 있음 — 이 로드맵 코드로 실제 배포본을 부하테스트하려면 이미지 재빌드가 필요함.
- 시나리오: `POST /payment`에 0→200 VU로 2분간 ramping-vus (20s 단위로 20→50→100→200, 200에서 30초 유지 후 램프다운). 결과 비교:

  | | `DB_CONNECTION_LIMIT=10`(기존 기본값) | `DB_CONNECTION_LIMIT=50` |
  |---|---|---|
  | 처리량 | 732 req/s | 1,049 req/s (+43%) |
  | 평균 지연 | 140.72ms | 98.2ms |
  | p95 | 370.46ms | 218.93ms (−41%) |
  | 최대 | 997.91ms | 749.16ms |
  | 에러율 | 0% | 0% |

- **결론**: 병목은 캐싱이나 트랜잭션 구조가 아니라 `base-database.config.ts`의 `connectionLimit` 기본값(`10`)이었음. 코드 변경 없이 커넥션 풀만 늘려도 처리량 +43%, p95 −41%를 확인함. 이 발견을 근거로 `docker-compose.yml`의 `payment` 서비스에 `DB_CONNECTION_LIMIT: "50"`을 추가해 실제로 반영함(같은 세션에서 컨테이너 리소스 제한 작업과 함께 `docker compose up -d`로 적용, 아래 "컨테이너 리소스 제한" 기록 참고).
- `GetPayment` Redis 캐싱은 **보류**함: 이번 부하테스트는 쓰기 경로(`POST /payment`)만 다뤘고, `GetPayment`는 HTTP로 노출돼 있지 않아(gRPC 전용, 인증 필요) 이번 k6 스크립트로는 측정하지 못함. 읽기·쓰기가 커넥션 풀을 두고 실제로 경합한다는 증거 없이 캐싱부터 넣는 건 로드맵이 스스로 경계한 "조기 최적화"라서, 커넥션 풀을 늘린 뒤 재측정해서 여전히 문제면 그때 착수하는 쪽으로 미뤄둠.
- "외부 호출을 DB 트랜잭션 밖으로 분리"는 코드 확인으로 완료 처리함: `CreatePaymentUseCase`에서 PG 승인 호출(`retryWithBackoff` 감싼 `pgAdapter.requestApproval`)은 `persist(pending)`과 `persistWithEvents(outcome, events)` 두 개의 별도 저장 사이, 즉 트랜잭션 밖에서 실행됨(`persistWithEvents` 내부에서만 `manager.transaction()`을 씀). Phase 3를 그 순서로 설계했을 때부터 이미 만족된 조건이었음.
- 남겨둔 것: 실제 트래픽에서 읽기 부하 실측(gRPC 부하테스트 도구 필요, 이번엔 없음), docker 배포 이미지 재빌드 후 배포본 대상 재측정, `DB_CONNECTION_LIMIT` 기본값 자체(현재 `base-database.config.ts`의 하드코딩 `10`)를 다른 서비스(identity 등)에도 올릴지 여부.

## 로드맵 종료 (2026-08-28)
Phase 1~4 모두 완료. 처음 세운 4단계 로드맵을 순서대로 구현했고, Phase 4는 원래 체크리스트(캐싱 구현)를 그대로 따르는 대신 부하테스트 데이터를 근거로 범위를 스스로 좁혔다 — 이게 애초에 Phase 4를 "실제 부하 테스트로 병목 구간을 확인한 뒤에만 착수" 조건으로 걸어둔 이유였다. 결제 서비스는 이제 멱등성, 상태 머신, outbox+Kafka 이벤트, PG 어댑터(+Mock)+웹훅+재시도+대사, 그리고 실측 기반 커넥션 풀 튜닝까지 갖춘 상태다. 다음으로 자연스러운 후속 작업은 실제 PG사 선정과 `GetPayment` 읽기 부하 실측이다.

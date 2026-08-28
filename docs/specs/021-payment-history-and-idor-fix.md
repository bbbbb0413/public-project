---
id: SPEC-021
title: 결제 상세 정보 노출 버그 수정 + 소유권 검증(IDOR) + 결제 내역 조회 API/탭 분리
status: done
targets: [server, front]
stages: [backend, frontend]
priority: high
---

## 배경 / 문제
[[SPEC-020]] 작업 중에는 프론트 상태 분기만 다뤘는데, 이번에 실제 gRPC 응답 구조를 다시 확인하니 그보다 근본적인 문제 두 가지가 있었다.

1. **`PaymentReply`에 `paymentId`/`accountId`/`productId`가 없음**: `libs/rpc/proto/payment.proto`의 `PaymentReply`는 `id`/`amount`/`currency`/`status` 네 필드뿐이다(`payment.grpc-mapper.ts:18-25`). 게이트웨이는 이 응답을 그대로 클라이언트에 전달하는데(`payment-gateway.controller.ts:48,64`), 프론트 `PaymentReply` 타입은 `paymentId`/`accountId`/`productId`를 기대하고 화면에 표시한다. 즉 지금 실제로 배포된 상태에서는 결제 ID·상품 코드·계정 ID가 전부 `undefined`로 표시된다 — 사용자가 보고한 "id값이 노출 안 됨"의 원인이다.
2. **`GetPayment`에 소유권 검증이 없음(IDOR)**: `payment.grpc-controller.ts:27-40`의 `getPayment`는 `paymentId`만으로 조회하고 요청자와 결제 소유자가 같은지 전혀 확인하지 않는다. 게이트웨이(`payment-gateway.controller.ts:51-65`)도 인증된 세션의 `accountId`를 아예 전달하지 않는다. 로그인한 사용자라면 누구나 `/payments/:id`의 `id`만 바꿔가며 다른 사람의 결제 금액·상품·상태를 조회할 수 있다.

여기에 더해 사용자가 지적한 두 가지 기능 격차가 있다.

3. **단건 조회(수동 ID 입력)만 있고 목록 조회가 없음**: `Payment.tsx`의 "결제 조회" 섹션은 사용자가 결제 ID를 직접 숫자로 입력해야 한다. 애초에 클라이언트가 내부 PK를 알 필요도, 노출할 필요도 없다 — 본인 결제 내역을 목록으로 보여주는 것이 맞다.
4. **구매 탭과 조회 영역이 분리되어 있지 않음**: 지금은 상품 구매 그리드와 결제 조회 섹션이 같은 화면(`Payment.tsx`) 안에 세로로 나열되어 있다. 성격이 다른 두 기능(구매 vs 내역 확인)이므로 별도 탭으로 나눈다.

## 요구사항

### 서버 (`public-server`)
1. `libs/rpc/proto/payment.proto`
   - `PaymentReply`에 `payment_id`, `account_id`, `product_id`를 추가한다(필드명을 도메인/DTO와 맞춘다).
   - `GetPaymentRequest`에 `account_id`를 추가해 게이트웨이가 요청자 accountId를 함께 넘기도록 한다.
   - `ListPayments` rpc를 추가한다: `ListPaymentsRequest{ account_id, page, take }` → `ListPaymentsResponse{ repeated PaymentReply payments, page, take, item_count, page_count, has_previous_page, has_next_page }` (기존 `GetAdminUsers` 페이지네이션 컨벤션을 따른다).
   - `ts-proto` 코드 생성을 다시 돌려 `libs/rpc/src/generated/payment.ts`를 갱신한다.
2. `PaymentGrpcMapper.toReply`가 `paymentId`/`accountId`/`productId`를 포함하도록 수정한다.
3. `PaymentGrpcController.getPayment`에 소유권 검증을 추가한다: 결제가 없거나 `payment.userId !== request.accountId`이면 (다른 사용자의 결제 존재 여부를 노출하지 않도록) 동일하게 `NOT_FOUND`를 던진다.
4. `PaymentGrpcController`에 `listPayments`를 추가하고, `IPaymentRepository`에 사용자 소유 결제만 페이지네이션 조회하는 `findPaymentsByUserId(userId, take, skip)`를 추가해 구현한다.
5. `apps/gateway/src/payment/payment-gateway.controller.ts`
   - `getPayment`이 세션의 `accountId`를 함께 전달하도록 수정한다.
   - `GET /payments`(목록, 쿼리 파라미터 `page`/`take`)를 추가해 인증된 사용자 본인의 결제 내역만 반환한다.

### 프론트 (`public-front`)
6. `Payment.tsx`를 "구매"/"결제 내역" 두 탭으로 분리한다. 상품 그리드+영수증 모달은 구매 탭에 남기고, 결제 내역 탭은 마운트 시 자동으로 `listPayments()`를 호출해 본인 결제 목록을 보여준다. 수동 ID 입력 필드는 제거한다.
7. 목록의 각 행은 이미 응답에 포함된 정보(결제 ID, 상품 코드, 금액, 상태)를 그대로 표시한다 — 상세를 보기 위해 별도로 ID를 입력하거나 노출할 필요가 없게 한다. 상태 배지는 [[SPEC-020]]에서 만든 `status-success`/`status-failed`/`status-pending` 스타일을 재사용한다.

## 비요구사항 (Out of scope)
- 목록에 대한 무한 스크롤/커서 페이지네이션 UI (첫 페이지 표시 + 필요 시 단순 페이지 이동 정도로 충분).
- 관리자용 전체 결제 목록 조회(이미 있는 `findAllAndCount`는 관리자 화면 몫으로 남겨두고 건드리지 않는다).
- 결제 내역 실시간 갱신(웹소켓 등) — 탭 진입/새로고침 시 조회로 충분.

## 구현 기록

**완료 (2026-08-28)**

- `libs/rpc/proto/payment.proto` — `PaymentReply`에 `payment_id`/`account_id`/`product_id` 추가, `GetPaymentRequest`에 `account_id` 추가, `ListPayments` rpc + `ListPaymentsRequest`/`ListPaymentsResponse`(기존 `GetAdminUsers` 페이지네이션 컨벤션 준수) 신설. `npm run proto:gen`으로 `libs/rpc/src/generated/payment.ts` 재생성.
- `payment.grpc-mapper.ts` — `toReply`가 `paymentId`/`accountId`/`productId`를 포함하도록 수정. 이전에는 이 세 필드가 응답에 아예 없어 실제 배포본에서 프론트의 결제 ID/상품 코드/계정 ID 표시가 전부 `undefined`였다.
- `payment.grpc-controller.ts` — `getPayment`에 소유권 검증 추가(`payment.userId !== request.accountId`이면 다른 사용자 결제 존재 여부를 노출하지 않기 위해 미존재와 동일하게 `NOT_FOUND`). `listPayments` 신설, `page`/`take` 0 이하 시 기본값(1, 20)으로 보정.
- `domain/repository/payment.repository.ts` / `infrastructure/persistence/payment.repository-impl.ts` — 사용자 소유 결제만 최신순 페이지네이션하는 `findPaymentsByUserId(userId, take, skip)` 추가. 기존 `findAllAndCount`(전체 조회, 소유권 필터 없음)는 그대로 두고 건드리지 않았다 — 사용자 화면에 재사용하면 다른 사용자 결제까지 노출되는 IDOR을 새로 만들게 되기 때문.
- `apps/gateway/src/payment/payment-gateway.controller.ts` — `getPayment`이 세션의 `accountId`를 함께 전달하도록 수정, `GET /payments`(목록, `page`/`take` 쿼리) 신설.
- 테스트
  - `payment.grpc-controller.spec.ts`: 본인 결제 조회 성공, 미존재 시 NOT_FOUND, **다른 사용자 결제 조회 시 NOT_FOUND(IDOR 방지)**, `listPayments` 정상 동작 및 `page`/`take` 기본값 보정 케이스 추가.
  - `create-payment.use-case.spec.ts`/`handle-pg-webhook.use-case.spec.ts`/`payment-reconciliation.service.spec.ts`: 저장소 목(mock)에 `findPaymentsByUserId` 추가(인터페이스 변경 반영).
  - `apps/gateway/test/gateway.e2e-spec.ts`: `GET /payments/:id`가 세션 `accountId`를 함께 넘기는지, `GET /payments`가 accountId 기반으로 목록을 반환하는지 검증하는 테스트 추가. **주의**: 이 e2e 스위트는 내 변경과 무관하게 이전부터 `IdentityGatewayController`의 `AuthService` DI 문제로 전부 실패 중이었다(변경 전 `git stash` 후 재실행해 동일하게 12개 실패 확인). 이번 세션에서 원인 규명/수정은 하지 않았다.
  - 결제 서비스 유닛 테스트 20개, 위 gateway e2e 이외 항목은 모두 통과. `npx tsc -p apps/payment/tsconfig.app.json --noEmit`, `npx tsc -p apps/gateway/tsconfig.app.json --noEmit` 통과.
- 프론트 (`public-front`)
  - `src/utils/payment-status.ts` 신설 — `Payment.tsx`(영수증 모달)와 `PaymentHistory.tsx`(목록)가 공유하는 `TERMINAL_STATUSES`/`statusClassName`.
  - `src/api/payment.ts` — `listPayments(page = 1, take = 20)` 추가, `PaymentListReply` 타입 추가.
  - `src/components/PaymentHistory.tsx` 신설 — 마운트 시 `listPayments()`를 호출해 본인 결제 목록을 렌더링. 수동 ID 입력 필드 없음.
  - `src/components/Payment.tsx` — "구매"/"결제 내역" 두 탭으로 분리(`payment-tabs`/`payment-tab-button`). 기존 상품 그리드+영수증 모달 로직은 내부 `Shop` 컴포넌트로 옮기고, 수동 ID 조회 섹션(`handleQuery`/`queryResult` 등)은 완전히 제거해 `PaymentHistory`로 대체했다.
  - `Payment.css` — 탭 스타일 추가, 더 이상 쓰이지 않는 `.payment-query-*` 규칙을 `.payment-history-*`로 교체.
  - 테스트: `PaymentHistory.test.tsx` 신설(목록 렌더링/빈 상태/에러 상태, ID 입력 필드 부재 확인), `Payment.test.tsx`에 탭 전환 테스트 추가, `payment.test.ts`에 `listPayments` 테스트 추가. 프론트 전체 153개 테스트 통과, `tsc -b`/`eslint` 통과.
- 배포: `docker compose build payment gateway frontend` 후 `docker compose up -d payment gateway frontend`로 재기동. 로그에서 `GET /payments`, `GET /payments/:id`, `POST /payments` 라우트가 게이트웨이에 정상 매핑된 것과 payment 서비스가 오류 없이 기동된 것을 확인했다. 로그인 계정이 없어 실제 브라우저로 로그인 후 목록/조회를 직접 눌러보는 라이브 검증은 하지 못했다 — 코드 경로는 위 유닛/e2e 테스트로 검증했다.

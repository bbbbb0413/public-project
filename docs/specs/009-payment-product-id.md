---
id: SPEC-009
title: 결제 응답에 상품 식별자(productId) 및 계정 식별자(accountId) 포함
status: ready
targets: [server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
`public-front/src/components/Payment.tsx:106-136` 및 `public-front/src/components/Payment.tsx:158-180` 에서는 결제 생성 완료 팝업(영수증)과 결제 단건 조회 결과 화면에서 상품 코드(`receipt.productId`, `queryResult.productId`)와 계정 ID(`queryResult.accountId`)를 렌더링하도록 구현되어 있다.
또한 프론트엔드 API 인터페이스 `public-front/src/api/payment.ts:3-10` 에도 `PaymentReply` 에 `paymentId`, `accountId`, `productId` 필드가 선언되어 있다.

그러나 백엔드 gRPC 프로토콜 정의인 `public-server/libs/rpc/proto/payment.proto:20-25` 의 `PaymentReply` 메시지에는 `product_id` 와 `account_id` 필드가 누락되어 있으며, `public-server/apps/payment/src/payment/rpc/payment.grpc-mapper.ts:17-24` 의 `toReply` 매퍼에서도 `id`, `amount`, `currency`, `status` 만 반환하고 있다.
이로 인해 게이트웨이 `public-server/apps/gateway/src/payment/payment-gateway.controller.ts:26-64` 를 거쳐 프론트엔드로 전달되는 응답 데이터에 `productId` 와 `accountId` 가 누락되어, 결제 완료 영수증 및 결제 조회 화면에서 상품 코드와 계정 ID 영역이 빈 값으로 표시되는 결함이 발생한다.

## 요구사항
- [ ] `public-server/libs/rpc/proto/payment.proto` 의 `PaymentReply` 메시지에 `product_id` (string, tag 5) 및 `account_id` (int64, tag 6) 필드를 추가한다
- [ ] `public-server/libs/rpc/src/generated/payment.ts` 의 TypeScript 인터페이스 `PaymentReply` 에 `productId: string`, `accountId: number` 필드를 추가한다
- [ ] `public-server/apps/payment/src/payment/rpc/payment.grpc-mapper.ts` 의 `toReply` 에서 도메인 객체(`payment`)의 `productId` 와 `userId`(`accountId`)를 응답 객체에 매핑한다
- [ ] 게이트웨이 `public-server/apps/gateway/src/payment/payment-gateway.controller.ts` 의 응답 필드가 누락 없이 프론트엔드로 전달되도록 한다
- [ ] 프론트엔드 `public-front/src/components/Payment.tsx` 에서 결제 생성 완료 모달 및 결제 조회 결과 화면에 `productId` 와 `accountId` 가 올바르게 표시되는지 확인한다
- [ ] 백엔드 gRPC 매퍼 및 컨트롤러의 응답 변환에 대한 단위 테스트를 작성한다

## 비요구사항 (Out of scope)
- 결제 상태(`status`)의 동적 상태 머신 및 DB 스키마 마이그레이션 (기존 `'COMPLETED'` 고정 유지)
- 결제 취소/환불 API 및 신규 비즈니스 엔드포인트 추가
- 결제 목록 다건 조회 페이징 API 구현

## 백엔드
`public-server/libs/rpc/proto/payment.proto`
- `PaymentReply` 메시지에 `string product_id = 5;`, `int64 account_id = 6;` 필드를 추가한다.

`public-server/libs/rpc/src/generated/payment.ts`
- `PaymentReply` 인터페이스에 `productId: string;`, `accountId: number;` 필드를 추가한다.

`public-server/apps/payment/src/payment/rpc/payment.grpc-mapper.ts`
- `toReply(payment: Payment)` 메서드 반환 객체에 `productId: payment.productId`, `accountId: payment.userId` 매핑을 추가한다.

## 프론트엔드
`public-front/src/components/Payment.tsx`
- 기존에 `receipt.productId`, `queryResult.productId`, `queryResult.accountId` 를 참조하여 렌더링하던 UI가 백엔드 응답을 정상 수신하여 렌더링되도록 확인 및 검증한다.

## 수용 기준 (Acceptance Criteria)
- Given 유효한 상품 ID(`prod_sword_01`)와 금액으로 결제 생성을 요청했을 때 When 결제 생성이 성공하면 Then 백엔드 응답 객체에 해당 `productId` 와 요청자의 `accountId` 가 포함되어 반환되고, 프론트엔드 결제 완료 모달의 "상품 코드"에 `prod_sword_01` 이 표시된다
- Given 기존에 생성된 결제 건(ID: 1, `productId`: `prod_shield_01`, `accountId`: 42)이 존재할 때 When 해당 결제 ID로 단건 조회를 요청하면 Then 응답 객체에 `productId: 'prod_shield_01'`, `accountId: 42` 가 포함되어 반환되고, 프론트엔드 결제 조회 결과 영역에 계정 ID `42` 와 상품 코드 `prod_shield_01` 이 표시된다
- Given `productId` 또는 `userId` 가 빈 문자열이거나 0인 경계 케이스의 결제 데이터가 주어졌을 때 When 결제 정보를 조회하면 Then 런타임 오류 없이 각각 `productId: ""` 또는 `accountId: 0` 으로 정상 직렬화되어 반환된다

## 참고
- 고쳐야 할 자리: `public-server/libs/rpc/proto/payment.proto:20-25`
- 고쳐야 할 자리: `public-server/apps/payment/src/payment/rpc/payment.grpc-mapper.ts:17-24`
- 관련 정의 및 기존 구현: `public-front/src/components/Payment.tsx:116-119`
- 관련 정의 및 기존 구현: `public-front/src/components/Payment.tsx:164-171`
- 관련 정의 및 기존 구현: `public-server/apps/payment/src/payment/domain/model/payment.ts:4-14`

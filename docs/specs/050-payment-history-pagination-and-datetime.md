---
id: SPEC-050
title: 결제 내역 페이징 메타데이터 연동 및 결제 일시 표시
status: done
targets: [server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
현재 결제 모듈에서 결제 내역을 조회하는 백엔드 gRPC 프로토콜 및 게이트웨이는 페이징 메타데이터를 제공하고 있으나, 프론트엔드 컴포넌트와의 데이터 계약 불일치 및 필드 누락으로 인해 20건을 초과하는 결제 내역 확인과 결제 일시 조회가 불가능한 상태이다.

- `public-server/libs/rpc/proto/payment.proto:29-37` (`ListPaymentsResponse`) 및 게이트웨이 `public-server/apps/gateway/src/payment/payment-gateway.controller.ts:64-82` (`PaymentGatewayController.listPayments`)는 `pageCount`, `hasPreviousPage`, `hasNextPage` 등 페이징 메타데이터를 계산하여 클라이언트에 제공하고 있다.
- 그러나 프론트엔드 컴포넌트 `public-front/src/components/PaymentHistory.tsx:11-26` (`PaymentHistory`)에서는 `listPayments()` 호출 시 반환되는 `payments` 배열만 상태로 저장하고 페이징 메타데이터를 무시하여, 20건을 초과하는 이전/다음 결제 내역을 사용자가 조회할 수 있는 페이지네이션 UI가 부재하다.
- 또한 백엔드 `public-server/libs/rpc/proto/payment.proto:39-46` (`PaymentReply`) 및 `public-server/apps/payment/src/payment/domain/model/payment.ts:7-18` (`Payment`)에 결제 생성 일시(`createdAt`) 필드가 누락되어 있어, `public-server/apps/payment/src/payment/rpc/payment.grpc-mapper.ts:18-27` (`PaymentGrpcMapper.toReply`)에서 일시 정보를 내려주지 못하고 있으며, `public-front/src/components/PaymentHistory.tsx:40-65` (`PaymentHistory`) 결제 내역 목록 카드에서도 결제가 언제 발생했는지 일시 정보를 확인할 수 없다.

이를 해결하여 결제 도메인의 계약 불일치를 바로잡고 사용자가 본인의 전체 결제 내역 및 결제 발생 일시를 투명하게 조회할 수 있도록 개선해야 한다.

## 요구사항
- [ ] `public-server/libs/rpc/proto/payment.proto`의 `PaymentReply` 메시지에 `string created_at = 7;` 필드를 추가하고 proto 컴파일을 통해 generated 타입을 갱신한다.
- [ ] `public-server/apps/payment`의 `Payment` 도메인 모델에 `createdAt?: Date` 필드를 추가하고 `restore` 메서드에서 이를 복원할 수 있도록 지원한다.
- [ ] `public-server/apps/payment`의 `PaymentMapper.toDomain`에서 ORM 엔티티의 `createdAt`을 도메인 모델로 전달한다.
- [ ] `public-server/apps/payment`의 `PaymentGrpcMapper.toReply`에서 도메인 모델의 `createdAt`을 ISO 8601 문자열(`payment.createdAt ? payment.createdAt.toISOString() : ''`)로 변환하여 응답에 포함한다.
- [ ] `public-front/src/api/payment.ts`의 `PaymentReply` 인터페이스에 `createdAt: string;` 필드를 추가한다.
- [ ] `public-front/src/components/PaymentHistory.tsx`에서 `PaymentListReply`의 `page`, `pageCount`, `hasPreviousPage`, `hasNextPage` 상태를 관리하고, 이전 페이지 및 다음 페이지 이동 버튼과 현재 페이지 번호("페이지 N / M")를 렌더링한다.
- [ ] `public-front/src/components/PaymentHistory.tsx`의 결제 내역 행에 `formatDateTime(payment.createdAt)` 유틸을 적용하여 "결제 일시" 항목을 렌더링한다.
- [ ] 백엔드 gRPC 매퍼 및 결제 목록 컨트롤러, 프론트엔드 결제 내역 컴포넌트에 대한 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 결제 금액 계산, 외부 PG사 결제 승인/취소/환불 흐름은 변경하지 않는다.
- 결제 내역 검색, 필터링(날짜별/상태별 필터) 기능은 추가하지 않는다.
- 페이지당 노출 개수(`take`)를 사용자가 동적으로 변경하는 드롭다운 셀렉터는 추가하지 않으며 기본값 20개를 유지한다.
- 사용자 인증(JWT/세션) 및 권한 판정 로직을 변경하지 않는다.

## 백엔드
- `public-server/libs/rpc/proto/payment.proto`:
  - `PaymentReply` 메시지에 `string created_at = 7;` 추가
- `public-server/apps/payment/src/payment/domain/model/payment.ts`:
  - `Payment` 클래스 생성자 및 `restore` 메서드에 `createdAt?: Date` 매개변수 지원
- `public-server/apps/payment/src/payment/infrastructure/mapper/payment.mapper.ts`:
  - `toDomain` 메서드에서 `createdAt: orm.createdAt` 전달
- `public-server/apps/payment/src/payment/rpc/payment.grpc-mapper.ts`:
  - `toReply` 메서드에서 `createdAt: payment.createdAt ? payment.createdAt.toISOString() : ''` 매핑

## 프론트엔드
- `public-front/src/api/payment.ts`:
  - `PaymentReply` 인터페이스에 `createdAt: string;` 필드 추가
- `public-front/src/components/PaymentHistory.tsx`:
  - 현재 페이지 번호(`page`), 총 페이지 수(`pageCount`), 이전/다음 여부(`hasPreviousPage`, `hasNextPage`) 상태 추가
  - 페이지 변경 시 `listPayments(targetPage, 20)` 재조회 처리
  - 결제 카드 내부에 "결제 일시" 행을 추가하고 `formatDateTime(payment.createdAt)` 호출 결과 렌더링
  - 목록 하단에 페이지네이션 컨트롤(이전 버튼, "N / M", 다음 버튼) 추가 및 첫 페이지/마지막 페이지 도달 시 버튼 `disabled` 처리
- `public-front/src/components/Payment.css`:
  - 페이지네이션 컨트롤러 및 결제 일시 행 스타일 추가

## 수용 기준 (Acceptance Criteria)
- Given 결제 내역이 총 25건 존재하는 사용자가 로그인한 상태일 때
  When 결제 내역 탭에 진입하면
  Then 1페이지(최신 20건)가 조회되고, 하단에 "1 / 2" 페이지 표시와 함께 '이전' 버튼은 비활성화되고 '다음' 버튼은 활성화된다.
  When '다음' 버튼을 클릭하면
  Then 2페이지(나머지 5건)가 조회되고, 하단에 "2 / 2" 페이지 표시와 함께 '이전' 버튼은 활성화되고 '다음' 버튼은 비활성화된다.
- Given 결제 내역이 존재하는 상태일 때
  When 결제 내역 카드를 확인하면
  Then 각 결제 항목마다 `YYYY-MM-DD HH:mm:ss` 형식으로 포맷팅된 결제 일시(예: `2026-09-12 14:30:00`)가 표시된다.
- Given `createdAt` 필드가 빈 문자열이거나 유효하지 않은 결제 데이터가 주어졌을 때 (경계 케이스)
  When 결제 내역 카드가 렌더링되면
  Then 오류 없이 빈 문자열 또는 대시(-)로 안전하게 폴백 처리되어 화면이 깨지지 않는다.

## 참고
- 결제 프로토콜 정의: `public-server/libs/rpc/proto/payment.proto:29-46` (`ListPaymentsResponse`, `PaymentReply`)
- 결제 도메인 모델: `public-server/apps/payment/src/payment/domain/model/payment.ts:7-18` (`Payment`)
- 결제 ORM 엔티티 매퍼: `public-server/apps/payment/src/payment/infrastructure/mapper/payment.mapper.ts:5-16` (`PaymentMapper.toDomain`)
- 결제 gRPC 응답 매퍼: `public-server/apps/payment/src/payment/rpc/payment.grpc-mapper.ts:18-27` (`PaymentGrpcMapper.toReply`)
- 게이트웨이 결제 목록 컨트롤러: `public-server/apps/gateway/src/payment/payment-gateway.controller.ts:64-82` (`PaymentGatewayController.listPayments`)
- 프론트 결제 API 정의: `public-front/src/api/payment.ts:6-23` (`PaymentReply`, `PaymentListReply`)
- 프론트 결제 내역 컴포넌트: `public-front/src/components/PaymentHistory.tsx:11-26` (`PaymentHistory`), `:40-65` (`PaymentHistory`)
- 일시 포맷 유틸: `public-front/src/utils/date.ts:71-133` (`formatDateTime`)

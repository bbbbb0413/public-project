---
id: SPEC-020
title: 프론트 결제 확인 화면의 상태 처리 개선 (FAILED 오표시 / 재시도 멱등키 / 에러 메시지)
status: done
targets: [front]
stages: [frontend]
priority: high
---

## 배경 / 문제
[[SPEC-019]] 로드맵으로 결제 서비스가 실제로 `PENDING` / `COMPLETED` / `FAILED` 상태 머신을 갖게 되고, PG 승인 거절이나 재시도 소진 시 백엔드가 진짜 `FAILED`를 반환할 수 있게 되었다. 그런데 이 변경 이전에 작성된 프론트(`public-front/src/components/Payment.tsx`)는 여전히 "HTTP 요청이 성공하면 결제도 성공"이라는 옛 전제로 만들어져 있어, 다음과 같은 문제가 있다.

- **`FAILED` 결제도 성공으로 표시됨**: `handlePurchase`는 `createPayment()`가 예외를 던지지 않으면 `receipt.status` 값을 보지 않고 무조건 `"결제가 완료되었습니다!"` 모달을 띄우고, `status-success`(초록) 클래스를 하드코딩해 렌더링한다. `handleQuery`의 조회 결과도 동일하다. `Payment.css`에는 애초에 `.status-success` 하나만 정의돼 있고 실패/대기 상태에 대응하는 클래스가 없다. 결과적으로 결제가 실패했는데 사용자는 성공했다고 믿는 상황이 발생할 수 있다.
- **재시도 시 멱등키가 재사용되지 않음**: `createIdempotencyKey()`가 `createPayment()` 내부에서 호출마다 새로 생성된다. 네트워크 오류로 응답을 못 받아 사용자가 다시 시도하면, 서버는 이미 처리했을 수 있는데 클라이언트는 새 키로 재요청해 [[SPEC-019]] Phase 1에서 만든 멱등성 보호가 무력화되고 중복 결제로 이어질 수 있다.
- **에러 메시지가 뭉뚱그려짐**: `catch` 블록이 400(검증 실패)/409류(멱등 처리 중 충돌)/5xx를 구분하지 않고 전부 `"결제 처리에 실패했습니다. 잠시 후 다시 시도해 주세요."`로 안내한다. 특히 서버가 "이미 처리 중"이라는 의미로 충돌 응답을 준 경우는 재시도를 유도하는 지금 메시지가 오히려 중복 요청을 부추긴다.
- **비동기 PG 확인 흐름 부재**: 지금은 `MockPgAdapter`가 동기 응답을 주기 때문에 문제가 드러나지 않지만, 실제 PG로 교체되면 승인 결과가 웹훅으로 나중에 도착하는 구조([[SPEC-019]] Phase 3)가 될 수 있다. 프론트는 요청 응답을 받는 즉시 최종 상태로 간주하고 있어, 이후 상태가 바뀌는 흐름을 반영할 자리가 없다.

## 요구사항

1. **결제 상태에 따른 분기 렌더링**
   - `receipt.status`(구매 직후)와 `queryResult.status`(수동 조회) 모두에서 `COMPLETED` / `FAILED` / `PENDING`(그 외 미확정 상태 포함)을 구분해 표시한다.
   - `COMPLETED`일 때만 `"결제가 완료되었습니다!"` + 성공 스타일을 사용한다.
   - `FAILED`일 때는 `"결제가 실패했습니다"` 계열의 실패 전용 문구 + 실패 스타일(`Payment.css`에 `.status-failed` 등 신설)을 사용하고, 성공 모달과 시각적으로 명확히 구분한다.
   - `PENDING`(또는 알 수 없는 상태)일 때는 "확인 중" 계열 문구 + 대기 스타일(`.status-pending`)을 사용한다.

2. **재시도 시 멱등키 재사용**
   - 멱등키를 `createPayment()` 내부가 아니라 "구매 시도" 단위(컴포넌트의 `handlePurchase` 호출 컨텍스트)에서 생성해, 동일 시도에 대한 재시도(에러 후 같은 상품 재클릭 등)에서는 같은 키를 재사용하도록 `api/payment.ts`의 시그니처를 조정한다.
   - 시도가 끝나(성공 모달 표시 또는 사용자가 다른 상품 선택) 새로운 구매를 시작할 때는 새 키를 발급한다.

3. **에러 메시지 세분화**
   - HTTP 상태 코드 기준으로 최소 3가지를 구분한다: 검증 오류(400류) / 처리 중 충돌(409류) / 그 외 네트워크·서버 오류(5xx, 타임아웃 등).
   - 409류(이미 처리 중)에는 재시도를 유도하지 않는 별도 안내를 사용한다.

4. **비동기 확인 대비 폴링(최소 구현)**
   - 구매 응답 상태가 `PENDING`인 경우, `getPayment()`를 짧은 간격으로 재호출해 최종 상태(`COMPLETED`/`FAILED`)가 확정되면 화면을 갱신하는 최소한의 폴링 로직을 추가한다.
   - 폴링은 최종 상태 도달 또는 최대 시도/시간 초과 시 종료하고, 초과 시에는 "확인이 지연되고 있습니다" 안내로 폴백한다.
   - 지금 백엔드(Mock PG)는 동기 응답만 주므로 이 경로는 즉시 트리거되지 않지만, 실제 PG 전환 시 바로 동작하도록 미리 마련해 둔다.

## 비요구사항 (Out of scope)
- 백엔드 API/응답 스키마 변경 (이미 `status` 필드가 내려오고 있어 프론트만 소비하면 됨).
- 실제 PG 웹훅에 맞춘 실시간(WebSocket/SSE) 상태 갱신 — 지금은 폴링 최소 구현까지만.
- 디자인 시스템 차원의 토스트/모달 컴포넌트 재설계 — 기존 컴포넌트 구조를 유지한 채 상태 분기만 추가.

## 구현 기록

**완료 (2026-08-28)**

- `public-front/src/api/payment.ts`
  - `createPayment`에 `idempotencyKey` 선택 인자를 추가해 호출부가 재시도 시 같은 키를 넘길 수 있게 했다(미지정 시 기존처럼 새로 생성).
  - `createIdempotencyKey`를 export해 컴포넌트에서 시도 단위 키를 직접 관리할 수 있게 했다.
  - `classifyPaymentError(error)`를 추가해 `response.status`를 409 / 4xx / 그 외(5xx·네트워크)로 분류한다. `axios.isAxiosError` 대신 duck-typing(`'response' in error`)으로 구현해 기존 axios 모킹 테스트와 충돌하지 않도록 했다.
- `public-front/src/components/Payment.tsx`
  - `receiptCopy`/`statusClassName`/`receiptTitleClassName`으로 `COMPLETED`/`FAILED`/그 외(대기)를 분기해, 영수증 모달과 조회 결과 모두 실제 상태에 맞는 문구·색상을 사용하도록 했다. `FAILED`가 더 이상 `status-success`(초록) + `"결제가 완료되었습니다!"`로 표시되지 않는다.
  - `pendingAttemptRef`로 "응답을 받지 못한 시도"만 같은 `idempotencyKey`를 재사용하도록 했다. 서버로부터 확정 응답(성공/실패 불문)을 받으면 즉시 해제해, 다음 구매 클릭은 새 시도로 취급한다. 다른 상품을 클릭하면 이전 시도와 무관하게 새 키가 발급된다.
  - `purchaseErrorMessage`로 요청 자체가 실패한 경우의 안내 문구를 409(처리 중)/4xx(요청 확인)/그 외(재시도 안내)로 구분했다.
  - 구매 응답이 `PENDING`(또는 알 수 없는 상태)이면 `pollForFinalStatus`가 2초 간격 최대 10회 `getPayment`를 호출해 최종 상태가 확정되면 영수증을 갱신한다. `activePaymentIdRef`로 사용자가 그 사이 다른 결제를 시작한 경우 오래된 폴링 결과가 화면을 덮어쓰지 않도록 막았다. 지금 백엔드는 Mock PG가 동기 응답만 주므로 이 경로는 실제로는 트리거되지 않지만, 실제 PG 전환 시 즉시 동작한다.
- `public-front/src/components/Payment.css`, `public-front/src/index.css`
  - `--color-warning`(#f59e0b) 전역 변수를 추가하고, `.status-failed`/`.status-pending`, `.receipt-title--success`/`--failed`/`--pending`을 신설했다.
- 테스트
  - `payment.test.ts`: 상태값을 실제 백엔드 enum(`PENDING`/`COMPLETED`/`FAILED`)에 맞춰 `SUCCESS` → `COMPLETED`로 고쳤고(기존 값은 백엔드에 존재한 적 없는 값이었음), `idempotencyKey` 재사용 케이스와 `classifyPaymentError` 세 갈래를 검증하는 테스트를 추가했다.
  - `Payment.test.tsx`: `FAILED` 상태 렌더링, 네트워크 오류 후 재시도 시 동일 멱등키 재사용, `PENDING` → 폴링 → `COMPLETED` 갱신(vitest fake timers) 케이스를 추가했다. 전체 10개 테스트 통과.
  - `npx tsc -b`, `npx eslint`로 타입/린트 통과 확인.

**검증 범위와 한계**
- 실제 브라우저로 로그인 후 구매 버튼을 눌러 라이브 E2E까지는 하지 않았다. 이유: (1) 로그인 계정 정보가 없었고, (2) 직전에 사용자가 부하테스트로 쌓인 테스트 결제 레코드를 정리해 달라고 요청한 직후라, 라이브 구매를 다시 트리거하면 같은 문제(테스트성 결제 레코드 재적재)가 재발한다. 대신 컴포넌트 테스트(axios 모킹)로 `COMPLETED`/`FAILED`/`PENDING`→폴링 세 경로를 결정론적으로 검증했다.
- 현재 배포된 Mock PG는 동기 응답만 주므로(`MOCK_PG_DECLINE_SENTINEL_AMOUNT`로만 강제 실패 가능), 실서비스에서 `FAILED`/`PENDING` 응답이 실제로 프론트까지 도달하는 전 구간(서버→프론트) 통합 확인은 하지 못했다. 코드 경로는 단위 테스트로만 확인됨.

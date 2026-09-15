---
id: SPEC-038
title: 로그인/회원가입 응답 내 계정 식별자(accountId) 전달 및 프로필 메일 발송 연동
status: done
targets: [server, front]
stages: [backend, frontend, qa]
priority: normal
---
## 배경 / 문제
현재 인증 및 게임 계정 식별자(`accountId`) 전달 흐름에서 게이트웨이와 프론트엔드 간의 데이터 계약 누락으로 인해, 프로필 화면의 메일 발송 기능이 차단되고 계정 식별자가 올바르게 표시되지 않는 문제가 발생하고 있다.

구체적인 문제 지점은 다음과 같다.
- `public-server/libs/rpc/proto/identity.proto:17-21` (`LoginResponse`) 및 `public-server/libs/rpc/proto/identity.proto:27-31` (`RegisterResponse`)에서 내부 gRPC 서비스(`identity`)는 이미 계정의 고유 식별자(`int64 id = 1`)를 정상적으로 반환하고 있다.
- 그러나 `public-server/apps/gateway/src/identity/identity-gateway.controller.ts:30-34` (`TokenResponse`) 인터페이스 및 `public-server/apps/gateway/src/identity/identity-gateway.controller.ts:54-68` (`login`), `public-server/apps/gateway/src/identity/identity-gateway.controller.ts:73-89` (`register`) 메서드에서 응답 객체 구성 시 `id`(`accountId`)를 누락한 채 `token`, `uuid`, `nickName`만 클라이언트로 내려주고 있다.
- 이로 인해 `public-front/src/api/identity.ts:3-7` (`AuthTokenResponse`) 및 `public-front/src/context/AuthContext.tsx:39-45` (`storeAuthData`)에서 `accountId`를 전달받지 못해 `localStorage`의 `user_info`와 인증 컨텍스트 상태에 `accountId`가 저장되지 않는다.
- 결과적으로 `public-front/src/components/Profile.tsx:55` (`Profile`)에서 `Account ID: N/A`로 노출되며, `public-front/src/components/Profile.tsx:21-25` (`handleSendMail`)의 유효성 검증 로직(`if (!auth.user?.accountId)`)에 걸려 "계정 정보가 올바르지 않습니다." 오류가 발생하고 메일 발송 API 호출이 원천 차단된다.

## 요구사항
- [ ] `public-server/apps/gateway/src/identity/identity-gateway.controller.ts`의 `TokenResponse` 인터페이스에 `accountId: number` 필드를 추가한다.
- [ ] `public-server/apps/gateway/src/identity/identity-gateway.controller.ts`의 `login` 및 `register` 메서드에서 gRPC 응답의 `reply.id`를 응답 본문의 `accountId` 필드로 매핑하여 반환한다.
- [ ] `public-front/src/api/identity.ts`의 `AuthTokenResponse` 인터페이스에 `accountId: number` 필드를 추가한다.
- [ ] `public-front/src/context/AuthContext.tsx`의 `storeAuthData`, `login`, `register` 메서드에서 `accountId`를 인자로 받아 `User` 객체 및 `localStorage`(`user_info`)에 영속화한다.
- [ ] `public-front/src/components/Profile.tsx`에서 로그인된 유저의 `accountId`가 화면에 정상 표시되고, 메일 전송 폼 제출 시 유효성 검사를 통과하여 `sendMail` API가 정상 호출되도록 한다.
- [ ] 게이트웨이 컨트롤러 및 프론트엔드 인증 컨텍스트/프로필 컴포넌트에 대한 단위 테스트를 추가하거나 갱신한다.

## 비요구사항 (Out of scope)
- 토큰 발급, 서명 검증, 권한 판정 로직 자체의 변경
- 메일 발송 백엔드 gRPC 처리 로직(`IdentityService.SendMail`) 및 메일함 조회 기능의 신규 구현
- 게이트웨이 인증 가드(`GatewayAuthGuard`)의 세션 검증 구조 수정

## 백엔드
- `public-server/apps/gateway/src/identity/identity-gateway.controller.ts`
- `TokenResponse` 인터페이스 정의에 `accountId: number` 필드 추가
- `login(@Body() dto: LoginDto)`: `reply.id`를 `accountId: Number(reply.id)`로 매핑하여 응답
- `register(@Body() dto: RegisterDto)`: `reply.id`를 `accountId: Number(reply.id)`로 매핑하여 응답

## 프론트엔드
- `public-front/src/api/identity.ts`
- `AuthTokenResponse` 인터페이스에 `accountId: number` 필드 추가
- `public-front/src/context/AuthContext.tsx`
- `storeAuthData(newToken: string, uuid: string, nickName: string, accountId?: number)`로 시그니처 확장 및 `userInfo`에 `accountId` 반영
- `login`, `register` 호출 완료 시 응답받은 `data.accountId`를 `storeAuthData`로 전달
- `public-front/src/components/Profile.tsx`
- `Account ID: {auth.user.accountId ?? 'N/A'}`가 실제 계정 식별자 숫자로 표시
- `handleSendMail`이 정상적으로 `auth.user.accountId`를 획득하여 `sendMail` 호출 수행

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 유효한 UUID로 로그인을 시도할 때
  When `POST /auth/login` 엔드포인트를 호출하면
  Then HTTP 200 응답과 함께 본문(`data`)에 `token`, `uuid`, `nickName`, `accountId`가 포함되어 반환된다.
- Given 사용자가 닉네임을 입력하고 회원가입을 시도할 때
  When `POST /auth/register` 엔드포인트를 호출하면
  Then HTTP 200 응답과 함께 신규 생성된 계정의 `accountId`가 포함되어 반환된다.
- Given `accountId`가 포함된 인증 상태로 프로필 화면에 진입했을 때
  When 프로필 헤더 정보를 확인할 때
  Then `Account ID` 항목에 'N/A' 대신 사용자의 계정 번호(예: `1`)가 정상 렌더링된다.
- Given 제목과 내용이 입력된 메일 발송 폼에서
  When "메일 보내기" 버튼을 클릭할 때
  Then "계정 정보가 올바르지 않습니다." 오류 없이 `sendMail` API가 호출되고 성공 상태 메시지가 표시된다.
- Given `accountId`가 없는 레거시 `user_info` 로컬스토리지 데이터가 존재하는 경우
  When 세션을 복원하여 화면에 진입할 때
  Then 런타임 오류가 발생하지 않고 `Account ID: N/A`로 안전하게 폴백된다.

## 참고
- gRPC 메시지 정의: `public-server/libs/rpc/proto/identity.proto:17-31` (`LoginResponse`, `RegisterResponse`)
- 게이트웨이 인증 컨트롤러: `public-server/apps/gateway/src/identity/identity-gateway.controller.ts:54-89` (`login`, `register`)
- 프론트엔드 인증 컨텍스트: `public-front/src/context/AuthContext.tsx:39-65` (`storeAuthData`, `login`, `register`)
- 프로필 컴포넌트: `public-front/src/components/Profile.tsx:19-43` (`handleSendMail`), `public-front/src/components/Profile.tsx:55` (`Profile`)
- 프론트엔드 API 클라이언트: `public-front/src/api/identity.ts:3-27` (`AuthTokenResponse`, `login`, `register`)

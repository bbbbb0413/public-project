---
id: SPEC-099
title: 관리자 비밀번호 변경 DTO 불일치 해결 및 사용자 관리 API 예외 처리 개선
status: ready
targets: [server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
어드민 관리 화면의 사용자 관리 기능(`UserManagement.tsx`)은 관리자가 사용자 계정을 조회하고 활성화 상태, 권한, 비밀번호를 변경할 수 있도록 제공되고 있으나, 백엔드 게이트웨이 및 어드민 서버와의 계약 불일치 및 예외 처리 결함으로 인해 핵심 관리 기능이 정상 동작하지 않고 실패한다.
첫째, `public-front/src/components/admin/UserManagement.tsx:97-110` (`handleChangePassword`) 및 `public-front/src/api/admin.ts:62-64` (`changePassword`)에서는 비밀번호 변경 모달에서 사용자의 이메일과 새 비밀번호를 `{ email, password }` 형태로 백엔드 게이트웨이 `POST /admin/user/change/password`에 전달한다. 그러나 게이트웨이 `public-server/apps/gateway/src/admin/admin-gateway.controller.ts:140-155` (`AdminGatewayController.changePassword`)에서는 요청 바디를 `AdminSignupInDto`로 바인딩하고 있다. `public-server/apps/gateway/src/admin/dto/admin-request.dto.ts:4-16` (`AdminSignupInDto`)를 보면 `@IsString() name: string;` 필드가 필수로 선언되어 있어, 프론트엔드가 보낸 유효한 요청이 NestJS 글로벌 ValidationPipe 단계에서 400 Bad Request("name must be a string") 에러로 일괄 거부된다. 이로 인해 관리자는 화면에서 사용자 비밀번호를 전혀 변경할 수 없다.
둘째, `public-front/src/components/admin/UserManagement.tsx:65-72` (`handleRoleChange`) 및 `public-front/src/api/admin.ts:57-60` (`updateUserRole`)에서 관리자가 사용자 권한(admin, user, moderator)을 변경하려고 시도하면 `PUT /admin/user/role`을 호출하지만, `public-server/apps/gateway/src/admin/dto/admin-request.dto.ts:36-39` (`UpdateRoleUserInDto`)에는 `userId: number`만 정의되어 있고 `role` 필드가 누락되어 있다. 또한 어드민 서버의 `public-server/apps/admin-server/src/user/user.service.ts:135-137` (`UserService.updateRole`)는 비즈니스 로직 미구현 상태로 무조건 500 `ServerErrorException`을 던지도록 하드코딩되어 있다. 이로 인해 권한 변경 요청 시 서버 장애(500 Internal Server Error)가 발생하고, 게이트웨이 `public-server/apps/gateway/src/admin/admin-gateway.controller.ts:129-136` (`AdminGatewayController.updateRole`)를 거쳐 화면에 단순 "권한 변경에 실패했습니다."라는 원인 불명의 에러만 노출된다.
셋째, `public-front/src/components/admin/UserManagement.tsx:100-109` (`handleChangePassword`)에서는 비밀번호 변경 실패 시 백엔드가 반환한 구체적인 실패 사유(비밀번호 복잡도 조건 미달, 권한 부족 등)를 에러 메시지로 추출하지 않고 하드코딩된 "비밀번호 변경에 실패했습니다." 단일 문구만 표시하여 관리자가 입력 오류 원인을 파악하기 어렵다.
따라서 비밀번호 변경 전용 DTO를 신설하여 계약 불일치를 바로잡고, 미구현 권한 변경 엔드포인트에 대한 500 서버 장애를 방지하고 명확한 예외 피드백을 제공하도록 정돈해야 한다.

## 요구사항
- [ ] 게이트웨이에 어드민 비밀번호 변경 전용 DTO(`ChangeAdminPasswordInDto`)를 정의하여 `email`(@IsEmail)과 `password`(@IsString) 필드를 필수로 검증하고, `name` 필드 요구를 제거한다.
- [ ] 게이트웨이 `admin-gateway.controller.ts`의 `changePassword` 엔드포인트 바디 바인딩을 `AdminSignupInDto`에서 `ChangeAdminPasswordInDto`로 변경한다.
- [ ] 게이트웨이 `UpdateRoleUserInDto`에 선택적 또는 필수 `role` 필드(@IsString, @IsOptional)를 정의하여 프론트엔드가 전달한 권한 정보가 유효성 검증을 통과하도록 한다.
- [ ] 어드민 서버 `UserService.updateRole`에서 미지원 기능 호출 시 시스템 장애를 나타내는 500 ServerErrorException 대신 명확한 안내 메시지를 포함한 예외(또는 400 Bad Request 계열의 비즈니스 예외)를 반환하도록 개선한다.
- [ ] 게이트웨이 `changePassword` 호출 시 백엔드 유효성 검증 실패(비밀번호 복잡도 오류 등)가 발생하면 세부 에러 메시지가 클라이언트로 온전히 전달되도록 한다.
- [ ] 프론트엔드 `UserManagement.tsx`의 비밀번호 변경 모달에서 백엔드 에러 응답 내 세부 메시지(`response.data.message`)가 존재할 경우 이를 `pwError`에 표시하여 사용자에게 구체적인 실패 원인을 안내한다.
- [ ] 프론트엔드 `UserManagement.tsx`의 권한 변경 실패 시 백엔드 응답 메시지가 존재하면 이를 상단 에러 배너에 구체적으로 표시한다.

## 비요구사항 (Out of scope)
- `user` 데이터베이스 테이블에 `role` 컬럼을 추가하는 DDL 마이그레이션은 이번 범위에 포함하지 않는다. (데이터 마이그레이션 배제)
- `libs/auth` 및 어드민 서버의 비밀번호 복잡도 검증 규칙(`isPasswordComplexity`)이나 bcrypt 해싱 알고리즘 자체의 변경은 포함하지 않는다. (암호화·해싱 규칙 불변)
- 일반 게임 유저 로그인 및 토큰 발급/검증 로직은 건드리지 않는다.

## 백엔드
`public-server/apps/gateway/src/admin/dto/admin-request.dto.ts`
- `ChangeAdminPasswordInDto` 클래스 신설 (`email: string`, `password: string` 선언 및 class-validator 데코레이터 적용)
- `UpdateRoleUserInDto` 클래스에 `@IsOptional() @IsString() role?: string;` 필드 추가

`public-server/apps/gateway/src/admin/admin-gateway.controller.ts`
- `changePassword` 메서드의 `@Body() dto: AdminSignupInDto`를 `@Body() dto: ChangeAdminPasswordInDto`로 수정
- gRPC `changeAdminPassword` 호출 시 `name: dto.email` 또는 빈 문자열을 전달하여 proto 규격 충족

`public-server/apps/admin-server/src/user/user.service.ts`
- `updateRole` 메서드에서 무조건 던지는 500 `ServerErrorException`을 정돈하여, 지원되지 않는 권한 변경 요청에 대해 명확한 비즈니스 에러 메시지("현재 사용자 권한 변경 기능은 지원되지 않습니다.")를 반환하도록 처리

## 프론트엔드
`public-front/src/components/admin/UserManagement.tsx`
- `handleChangePassword`에서 catch 블록의 에러 객체로부터 Axios 응답 에러 메시지(`err?.response?.data?.message || '비밀번호 변경에 실패했습니다.'`)를 추출하여 `setPwError`에 반영
- `handleRoleChange`에서 catch 블록의 에러 객체로부터 Axios 응답 에러 메시지(`err?.response?.data?.message || '권한 변경에 실패했습니다.'`)를 추출하여 상단 에러 상태에 반영

## 수용 기준 (Acceptance Criteria)
- Given 관리자가 사용자 관리 화면에서 특정 유저의 [비번변경] 버튼을 클릭해 모달을 연 상태에서 When 규격에 맞는 새 비밀번호를 입력하고 [변경]을 클릭하면 Then 게이트웨이 ValidationPipe에서 400 name 검증 오류 없이 요청이 정상 처리되고 모달이 닫힌다.
- Given 관리자가 비밀번호 변경 모달에서 복잡도 요건(10~30자, 대/소문자/숫자/특수문자 조합)을 만족하지 않는 비밀번호를 입력하고 [변경]을 클릭하면 Then 백엔드로부터 반환된 구체적인 복잡도 안내 메시지가 모달 내 `pwError` 텍스트로 노출된다.
- Given 관리자가 비밀번호 입력 필드를 빈 값으로 둔 상태에서(경계 케이스) When [변경] 버튼을 클릭하면 Then API 호출이 실행되지 않고 모달 상태가 유지된다.
- Given 사용자 관리 테이블에서 권한 셀렉트박스를 변경하여 `updateRole` API가 호출되는 상황에서 When 백엔드가 미구현 권한 변경 요청을 수신하면 Then 원인 불명의 500 서버 장애 대신 명확한 비즈니스 안내 메시지가 반환되고 화면에 해당 안내가 노출된다.
- Given 어드민 유저 목록을 새로고침하거나 페이지를 이동하는 상황에서 When 기존 사용자 데이터가 로드되면 Then 기존 활성화 토글 및 유저 상세 조회 기능이 기존과 동일하게 정상 동작한다.

## 참고
- 고쳐야 할 자리: `public-server/apps/gateway/src/admin/dto/admin-request.dto.ts:4-16` (`AdminSignupInDto`)
- 고쳐야 할 자리: `public-server/apps/gateway/src/admin/admin-gateway.controller.ts:140-155` (`AdminGatewayController.changePassword`)
- 고쳐야 할 자리: `public-server/apps/admin-server/src/user/user.service.ts:135-137` (`UserService.updateRole`)
- 프론트엔드 모달 자리: `public-front/src/components/admin/UserManagement.tsx:97-110` (`UserManagement.handleChangePassword`)
- 프론트엔드 API 호출부: `public-front/src/api/admin.ts:62-64` (`changePassword`)
- 어드민 gRPC 컨트롤러: `public-server/apps/admin-server/src/auth/admin-auth.grpc-controller.ts:90-103` (`AdminAuthGrpcController.changeAdminPassword`)

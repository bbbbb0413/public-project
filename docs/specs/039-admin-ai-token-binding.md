---
id: SPEC-039
title: 관리자 AI API 클라이언트 토큰 바인딩 수정 및 403 권한 오류 해결
status: done
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
현재 프론트엔드의 관리자 전용 AI 패널(프롬프트 관리, LLM 모니터링, Ragas 평가)에서 호출하는 API 클라이언트가 일반 사용자 인증 토큰을 참조하고 있어 백엔드 관리자 권한 검증을 통과하지 못하는 결함이 존재한다.
- `public-front/src/api/aiAdmin.ts:1` 에서 일반 사용자 토큰(`token`)을 헤더에 주입하는 기본 `client`를 import하여 AI 관리자 API 요청을 전송하고 있다.
- `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:32-46` (`create`) 및 `:74-86` (`activate`)는 `@UseGuards(AdminGuard)`를 적용하여 요청자가 활성화된 관리자 계정(`user.email`, `user.activatedAt`)인지 엄격하게 검증한다.
- `public-server/apps/gateway/src/auth/admin.guard.ts:9-20` (`AdminGuard`)는 관리자 정보가 없으면 403 Forbidden 예외를 반환한다.
- 그 결과 `public-front/src/components/admin/PromptManagement.tsx:79-96` (`handleCreate`) 및 `:52-60` (`handleActivate`)에서 관리자가 로그인한 상태(`admin_token` 보유)여도 `aiAdmin.ts`가 `admin_token`이 아닌 일반 사용자 `token`을 보내거나 토큰 없이 요청을 보내므로 403 권한 오류가 발생하여 프롬프트 생성 및 활성화 기능이 차단된다.
- 반면 `public-front/src/api/admin.ts:5-15` (`identityClient`)는 `admin_token`을 올바르게 바인딩하여 사용하고 있으나, AI 관리자 API 클라이언트는 이 인증 흐름과 분리되어 방치되어 있다.

## 요구사항
- [ ] 관리자 AI API 요청 시 `localStorage`의 `admin_token`을 `Authorization: Bearer <admin_token>` 헤더로 주입하는 관리자 전용 HTTP 클라이언트를 사용하도록 개선한다.
- [ ] `public-front/src/api/aiAdmin.ts`의 모든 엔드포인트 호출(`createPrompt`, `getUserActivePrompt`, `getPromptVersions`, `getActivePrompt`, `activatePromptVersion`, `getLlmCosts`, `getCircuitBreakers`, `getRagasEvals`)이 관리자 전용 클라이언트를 통해 요청되도록 수정한다.
- [ ] 관리자 토큰 만료 또는 인증 실패(401 Unauthorized) 발생 시 `admin_token`과 `admin_info`를 안전하게 정리(제거)한다.
- [ ] 관리자 로그인 상태에서 프롬프트 생성(`createPrompt`) 및 버전 활성화(`activatePromptVersion`) 시 403 Forbidden 오류 없이 정상적으로 요청이 처리되고 성공 메시지가 표시되어야 한다.
- [ ] 관리자 토큰이 없는 상태에서 AI 관리자 API 호출 시 요청 인터셉터에서 불필요한 토큰 주입 없이 요청이 전송되고 401/403 에러가 안전하게 처리되어야 한다.
- [ ] 프론트엔드 API 클라이언트 토큰 바인딩 및 헤더 주입 동작에 대한 단위 테스트를 추가한다.

## 비요구사항 (Out of scope)
- 백엔드 게이트웨이의 `AdminGuard` 로직이나 gRPC 관리자 인증 메커니즘을 수정하지 않는다.
- 일반 사용자 AI 서비스 화면(`AiService.tsx`)에서 사용하는 `client`나 `ai.ts`의 토큰 주입 방식을 변경하지 않는다.
- 어드민 UI 디자인이나 레이아웃 컴포넌트 구조를 변경하지 않는다.

## 프론트엔드
`public-front/src/api/aiAdmin.ts` 및 관리자 API 클라이언트 계층을 수정한다.
- 관리자 토큰 인터셉터가 적용된 `adminClient`(또는 `admin.ts`의 클라이언트 재사용/독립 인스턴스)를 생성하여 `aiAdmin.ts`에서 이를 사용하도록 변경한다.
- `localStorage.getItem('admin_token')`이 존재하는 경우 `Authorization` 헤더에 Bearer 토큰으로 포함한다.
- 401 응답 수신 시 관리자 인증 스토리지 키(`admin_token`, `admin_info`)를 정리하는 인터셉터 에러 처리를 보장한다.

## 수용 기준 (Acceptance Criteria)
- Given 관리자 계정으로 로그인하여 `localStorage`에 유효한 `admin_token`이 저장되어 있을 때, When `PromptManagement` 화면에서 새 프롬프트를 생성하거나 버전을 활성화하면, Then 게이트웨이 `PromptProxyController`의 `AdminGuard`를 정상 통과하여 403 에러 없이 생성/활성화가 완료되고 UI에 성공 메시지가 표시된다.
- Given 관리자 로그인을 하지 않아 `admin_token`이 없을 때, When 관리자 AI API를 호출하면, Then `Authorization` 헤더에 일반 사용자 `token`이 잘못 주입되지 않고 백엔드로부터 401/403 에러를 받아 UI에 실패 메시지가 안전하게 노출된다.
- Given 관리자 토큰이 만료되어 API 호출 시 401 에러가 반환될 때, When 응답 인터셉터가 에러를 감지하면, Then `localStorage`에서 `admin_token`과 `admin_info`가 삭제된다.

## 참고
- 고쳐야 할 자리: `public-front/src/api/aiAdmin.ts:1` (`client` import 및 사용)
- 관리자 토큰 바인딩 참조 구현: `public-front/src/api/admin.ts:5-15` (`identityClient`)
- 권한 검증 백엔드 진입점: `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:32-46` (`create`), `:74-86` (`activate`)
- 관리자 가드 정의: `public-server/apps/gateway/src/auth/admin.guard.ts:9-20` (`AdminGuard`)

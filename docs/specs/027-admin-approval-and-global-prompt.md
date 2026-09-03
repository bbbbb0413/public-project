---
id: SPEC-027
title: 관리자 승인 및 전역 프롬프트 — 전역 프롬프트 API 관리자 권한 체크 부재 수정
status: done
targets: [server, front]
stages: [backend, frontend]
priority: high
---

## 출처
명세서 "5. 관리자 운영 및 접근 관리" 중 **5.1 관리자 승인 및 전역 프롬프트**(5.1.1 관리자 계정 승인, 5.1.2 전역 프롬프트 활성화). 개인 프롬프트는 [[026-personal-system-prompt]] 참고.

## ⚠️ 보안 취약점 (착수 전 최우선 확인)
`PromptProxyController` 전체(`apps/gateway/src/ai/proxy/prompt-proxy.controller.ts`)에 `GatewayAuthGuard`(로그인 여부만 확인)만 걸려 있고, **관리자 role 체크가 전혀 없다.** `apps/gateway/src/ai`, `apps/gateway/src/auth` 어디에도 `RolesGuard`/`@Roles` 데코레이터/admin 체크가 없음(grep 0건 확인). 즉 **로그인만 한 일반 사용자도 `PATCH /ai/prompts/:name/:version/activate`를 호출해 조직 전체 기본 프롬프트를 마음대로 바꿀 수 있다** — 명세서 5.1의 수용 기준("관리자는 개인 설정에 영향을 주지 않고 전역 AI 기본 프롬프트를 활성화할 수 있다" = 관리자만 할 수 있어야 함)을 정면으로 위반한다. 프론트의 `PromptManagement.tsx`는 관리자 패널 안에서만 노출되지만, 백엔드 API 자체가 무방비라 UI로 숨기는 것뿐이고 실제로는 아무 보호가 안 된다.

명세서 요구사항 구현 여부와 별개로, **이 취약점은 지금 당장 고쳐도 되는 독립 이슈**다. gateway에 이미 관리자 인증 체계(`apps/admin-server`, `AUTH_GRPC_URL`)가 있으므로, 그 세션/역할 정보를 gateway의 `PromptProxyController`에도 검증하도록 가드를 추가하면 된다.

## 현재 구현 상태 (조사 결과)

| 항목 | 상태 | 근거 |
|---|---|---|
| 관리자 계정 승인(활성화) | ✅ 구현됨, 명세서와 일치 | `apps/admin-server/src/user/user.service.ts:110`(`activatedAt` 없으면 로그인 차단), `:119-125`(`activate(id)`), `infrastructure/persistence/users.repository-impl.ts:63-70`. 프론트 `components/admin/UserManagement.tsx`에 승인 UI 존재 |
| 승인 전 로그인 차단 | ✅ 구현됨 | 위와 동일 위치 |
| 전역 프롬프트 버전 생성/목록/활성화 로직 | ✅ 구현됨 | `prompt-proxy.controller.ts`, `service.py:76-88`(`user_id=None`일 때 전역 범위로 정확히 동작) |
| 전역 활성화가 개인 오버라이드를 안 건드림 | ✅ 구현됨, 명세서와 일치 | `service.py:76-88` |
| 전역 프롬프트 활성화 API의 관리자 권한 체크 | ❌ **취약점** | 위 "보안 취약점" 항목 참고 |

## 명세서와 현재 구현 간 상충 / 차이
1. **가장 중요한 차이는 상충이 아니라 누락**: 명세서가 요구하는 로직(버전 관리, 개인 오버라이드 미침해)은 이미 다 있다. 없는 건 딱 하나, "관리자만"이라는 권한 경계다.
2. **관리자 role을 어디서 검증할지 구조 확인 필요**: 지금 `GatewayAuthGuard`는 JWT/basic/apikey 인증만 하고 role 개념이 없다(`apps/gateway/src/auth/gateway-auth.guard.ts` — `Session.create()`에 role 필드가 없는 것으로 보임, [[021-payment-history-and-idor-fix]] 조사 때 확인한 `Session` 구조 참고). admin-server 쪽 인증(별도 JWT? `AUTH_GRPC_URL`)과 gateway의 일반 사용자 인증이 분리되어 있을 가능성이 높다 — 착수 시 admin 세션을 gateway가 어떻게 인지할지(별도 헤더, 별도 가드, 아니면 admin-server 프록시 경로를 통째로 분리) 먼저 확인해야 한다.

## 요구사항 (이번 작업 범위)
1. **(선행, 독립 작업)** `PromptProxyController`의 쓰기 작업(버전 활성화 등 전역 상태를 바꾸는 엔드포인트)에 관리자 권한 검증을 추가한다. gateway의 admin 인증 경로가 어떻게 구성돼 있는지 먼저 확인하고, 없으면 admin-server를 거치는 프록시 방식이나 관리자 전용 가드를 새로 만든다.
2. (관리자 승인 자체는 이미 구현되어 있으므로) 명세서가 요구하는 나머지 UI/문구 디테일이 현재 `UserManagement.tsx`와 일치하는지 최종 확인만 한다.

## 비요구사항 (Out of scope)
- 개인 프롬프트 다중 슬롯 전환 — [[026-personal-system-prompt]].
- 세분화된 관리자 role(예: 슈퍼관리자/일반관리자 구분) — 명세서는 단일 "관리자" role만 요구한다.

## 구현 기록
(구현 완료 후 작성)

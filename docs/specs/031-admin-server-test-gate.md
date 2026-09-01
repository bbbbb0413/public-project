---
id: SPEC-031
title: admin-server 단위 테스트 스위트 및 scoped_test 게이트 연결
status: ready
targets: [server]
stages: [backend, qa]
priority: normal
---

## 배경 / 문제

`public-server/apps/admin-server` 서비스가 추가되어 관리자 인증·회원가입·계정 승인 등의 로직을 담당하고 있으나, 테스트 실행 체계와 자동화 파이프라인 게이트에서 누락되어 있다.

1. `public-server/package.json:50-53` (`scripts`)에서 `test:all` 스크립트(`pnpm test:identity && pnpm test:payment && pnpm test:chat && pnpm test:gateway && pnpm test:ai`)에 `admin-server` 실행 명령(`test:admin`)이 포함되어 있지 않다.
2. `public-server/apps/admin-server/src/user/user.service.spec.ts:1-270` (`describe('UserService')`)에 270줄 규모의 단위 테스트가 이미 작성되어 있으나, 이를 실행하기 위한 Jest 설정 파일(`jest-unit.json`)과 npm 스크립트가 없어 테스트 스위트가 실행되지 않는다.
3. `.specflow/config.yaml:75-80` (`scoped_test`) 목록에 `apps/admin-server` 경로 매핑이 누락되어 있어, 자동화 파이프라인에서 admin-server 코드가 수정되어도 관련 테스트가 검사받지 않고 통과하는 문제가 존재한다.
4. `public-server/package.json:52` 의 `test:all`에 이미 파이썬 서버로 이전된 레거시 `test:ai`(`apps/ai-service/jest-unit.json`)가 남아 있어 불필요한 빌드 오버헤드가 발생하고 있는 반면, 활성 서비스인 `admin-server`는 방치되어 있다.

## 요구사항

- [ ] `public-server/apps/admin-server/jest-unit.json` Jest 단위 테스트 설정 파일을 추가한다.
- [ ] `public-server/package.json` 의 scripts에 `test:admin` 및 `test:admin:cov` 명령을 추가한다.
- [ ] `public-server/package.json` 의 `test:all` 및 `test:all:cov`에 `test:admin` 및 `test:admin:cov`를 포함한다.
- [ ] `.specflow/config.yaml` 의 `targets.server.scoped_test` 에 `apps/admin-server: "pnpm test:admin"` 항목을 추가한다.
- [ ] `pnpm test:admin` 실행 시 `apps/admin-server/src/user/user.service.spec.ts`의 모든 단위 테스트가 정상 통과(GREEN)해야 한다.

## 비요구사항 (Out of scope)

- `apps/ai-service` 레거시 디렉토리 및 `test:ai` 스크립트의 강제 삭제 (기존 레거시 보존 정책 준수).
- `apps/admin-server` 내 gRPC 컨트롤러(`admin-auth.grpc-controller.ts`) 등 새로운 신규 테스트 케이스 대규모 추가 (기존 작성된 테스트 스위트의 실행 체계 확립에 집중).
- 관리자 권한 RBAC 인가 가드 신설 (SPEC-027 범위).

## 백엔드

- `public-server/apps/admin-server/jest-unit.json`:
  - `identity` 및 `gateway`의 `jest-unit.json` 표준 구성을 따라 `apps/admin-server/src/**/*.spec.ts`를 탐색하고 `@libs/*` 별칭 모듈을 매핑하는 Jest 설정 생성.
- `public-server/package.json`:
  - `test:admin`: `jest --config apps/admin-server/jest-unit.json`
  - `test:admin:cov`: `jest --config apps/admin-server/jest-unit.json --coverage`
  - `test:all`: `pnpm test:admin` 연동
  - `test:all:cov`: `pnpm test:admin:cov` 연동
- `.specflow/config.yaml`:
  - `targets.server.scoped_test`에 `apps/admin-server: "pnpm test:admin"` 추가.

## 수용 기준 (Acceptance Criteria)

- Given `public-server` 디렉토리에서
  When `pnpm test:admin` 명령을 실행하면
  Then `apps/admin-server/src/user/user.service.spec.ts`의 모든 테스트 케이스가 성공적으로 실행되고 종료 코드 0을 반환한다.
- Given `public-server` 디렉토리에서
  When `pnpm test:all` 명령을 실행하면
  Then `test:identity`, `test:payment`, `test:chat`, `test:gateway`, `test:admin`, `test:ai`가 모두 순차적으로 실행된다.
- Given SpecFlow 파이프라인에서 `apps/admin-server` 내의 파일이 수정되었을 때
  When scoped_test 단계가 실행되면
  Then `pnpm test:admin`이 트리거되어 admin-server의 단위 테스트 검증이 정상 수행된다.

## 참고

- 설정 누락 위치: `public-server/package.json:52`
- 기존 단위 테스트 코드: `public-server/apps/admin-server/src/user/user.service.spec.ts:1-271`
- 파이프라인 게이트 설정: `.specflow/config.yaml:75-80`
- 참고 대상 Jest 설정: `public-server/apps/gateway/jest-unit.json:1-29`

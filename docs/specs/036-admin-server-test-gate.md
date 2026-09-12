---
id: SPEC-036
title: admin-server 단위 테스트 스위트 및 scoped_test 게이트 연결
status: done
targets: [server]
stages: [backend, qa]
priority: normal
---

## 배경 / 문제

`public-server/apps/admin-server` 서비스가 모노레포에 추가되어 관리자 인증·회원가입·계정 승인 등의 핵심 로직을 담당하고 있으나, 빌드/테스트 스크립트와 자동화 파이프라인 게이트에서 누락되어 있다.

1. `public-server/apps/admin-server/src/user/user.service.spec.ts:1-271` (`describe('UserService')`)에 270줄 규모의 단위 테스트 코드가 이미 구현되어 있으나, 이를 실행하기 위한 Jest 설정 파일(`jest-unit.json`) 및 npm 스크립트(`test:admin`)가 없어 테스트가 전혀 실행되지 않는다.
2. `public-server/package.json:50-53` (`scripts`)의 `test:all` 스크립트(`pnpm test:identity && pnpm test:payment && pnpm test:chat && pnpm test:gateway && pnpm test:ai`)에 `admin-server` 테스트 명령이 포함되어 있지 않다.
3. `.specflow/config.yaml:75-81` (`targets.server.scoped_test`) 매핑 목록에 `apps/admin-server` 경로가 등록되어 있지 않아, 자동화 파이프라인에서 admin-server 코드가 수정되더라도 단위 테스트가 트리거되지 않고 검증 없이 통과하는 결함이 존재한다.
4. `public-server/nest-cli.json:110-117` (`projects.admin-server`)에는 admin-server 프로젝트 정의가 등록되어 있으나 Jest 실행 환경과의 연동 설정이 누락되어 있다.

이로 인해 관리자 기능 수정 시 회귀 결함을 사전에 탐지할 수 없으므로, admin-server의 단위 테스트 실행 환경을 구축하고 자동화 게이트에 연결해야 한다.

## 요구사항

- [ ] `public-server/apps/admin-server/jest-unit.json` Jest 단위 테스트 설정 파일을 추가한다.
- [ ] `public-server/package.json` 의 scripts에 `test:admin` 및 `test:admin:cov` 명령을 추가한다.
- [ ] `public-server/package.json` 의 `test:all` 및 `test:all:cov`에 `test:admin` 및 `test:admin:cov`를 포함한다.
- [ ] `.specflow/config.yaml` 의 `targets.server.scoped_test` 에 `apps/admin-server: "pnpm test:admin"` 항목을 추가한다.
- [ ] `pnpm test:admin` 실행 시 `apps/admin-server/src/user/user.service.spec.ts`의 모든 단위 테스트가 정상 통과(GREEN)해야 한다.

## 비요구사항 (Out of scope)

- `apps/ai-service` 레거시 디렉토리 및 `test:ai` 스크립트 정리 (SPEC-032 범위).
- `apps/admin-server` 내 gRPC 컨트롤러(`admin-auth.grpc-controller.ts`) 등 신규 테스트 케이스 대규모 추가 (기존 작성된 `user.service.spec.ts` 테스트 스위트의 실행 체계 확립에 집중).
- 관리자 권한 RBAC 인가 가드 신설 (SPEC-027 범위).

## 백엔드

- `public-server/apps/admin-server/jest-unit.json`:
  - `apps/gateway/jest-unit.json`의 표준 구성을 준용하여 `apps/admin-server/src/**/*.spec.ts`를 탐색하고 `@libs/*` 모듈 별칭을 매핑하는 Jest 설정 파일 생성.
  ```json
  {
    "moduleFileExtensions": ["js", "json", "ts"],
    "rootDir": "../../",
    "setupFiles": ["<rootDir>/test-support/jest-setup.js"],
    "testMatch": [
      "<rootDir>/apps/admin-server/src/**/*.spec.ts"
    ],
    "transform": {
      "^.+\\.ts$": ["ts-jest", { "tsconfig": "./tsconfig.spec.json" }]
    },
    "collectCoverageFrom": [
      "apps/admin-server/src/**/*.ts",
      "!**/*.module.ts",
      "!**/index.ts",
      "!**/*.dto.ts",
      "!**/*.entity.ts"
    ],
    "coverageDirectory": "coverage/admin-server",
    "testEnvironment": "node",
    "moduleNameMapper": {
      "^@libs/shared-kernel(|/.*)$": "<rootDir>/libs/shared-kernel/src/$1",
      "^@libs/common(|/.*)$": "<rootDir>/libs/common/src/$1",
      "^@libs/dao(|/.*)$": "<rootDir>/libs/dao/src/$1",
      "^@libs/auth(|/.*)$": "<rootDir>/libs/auth/src/$1",
      "^@libs/rpc(|/.*)$": "<rootDir>/libs/rpc/src/$1"
    }
  }
  ```
- `public-server/package.json`:
  - `scripts`에 `test:admin` (`jest --config apps/admin-server/jest-unit.json`) 및 `test:admin:cov` (`jest --config apps/admin-server/jest-unit.json --coverage`) 추가.
  - `test:all` 및 `test:all:cov` 스크립트 체인에 `pnpm test:admin` 및 `pnpm test:admin:cov` 연동.
- `.specflow/config.yaml`:
  - `targets.server.scoped_test`에 `apps/admin-server: "pnpm test:admin"` 추가.

## 수용 기준 (Acceptance Criteria)

- Given `public-server` 디렉토리에서
  When `pnpm test:admin` 명령을 실행하면
  Then `apps/admin-server/src/user/user.service.spec.ts`의 모든 테스트 케이스가 성공적으로 실행되고 종료 코드 0을 반환한다.
- Given `public-server` 디렉토리에서
  When `pnpm test:all` 명령을 실행하면
  Then `test:admin`을 포함한 활성 서비스 단위 테스트가 순차적으로 실행되어 모두 통과한다.
- Given SpecFlow 파이프라인에서 `apps/admin-server` 내의 파일이 수정되었을 때
  When scoped_test 단계가 실행되면
  Then `pnpm test:admin`이 트리거되어 admin-server의 단위 테스트 검증이 정상 수행된다.
- Given admin-server 내에 테스트 파일이 없거나 빈 디렉토리일 경우 (경계 조건)
  When `pnpm test:admin`을 실행하면
  Then Jest 설정 에러나 모듈 해석 실패 없이 정상 종료된다.

## 참고

- 기존 단위 테스트 코드: `public-server/apps/admin-server/src/user/user.service.spec.ts:1-271` (`describe('UserService')`)
- 패키지 스크립트 정의: `public-server/package.json:40-53` (`scripts`)
- 게이트웨이 Jest 설정 참조: `public-server/apps/gateway/jest-unit.json:1-29`
- SpecFlow 스코프 테스트 설정: `.specflow/config.yaml:75-81` (`targets.server.scoped_test`)
- Nest CLI 프로젝트 설정: `public-server/nest-cli.json:110-117` (`projects.admin-server`)

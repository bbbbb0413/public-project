---
id: SPEC-032
title: 레거시 NestJS ai-service 테스트 및 빌드 설정 정리
status: ready
targets: [server]
stages: [backend, qa]
priority: normal
---

## 배경 / 문제
Python 기반 RAG 서비스(`public-python-server`)로 AI 도메인이 완전 이관되었으나, `public-server`에 레거시 NestJS ai-service 관련 테스트 스크립트와 프로젝트 설정이 남아 있어 불필요한 빌드 및 테스트 부하를 유발하고 있다.
`docker-compose.yml:330`에서 "구버전 NestJS ai-service(v1)는 폐기. Gateway가 Kafka/HTTP로 ai-service-py에 직접 프록시한다."라고 명시되어 운영 컨테이너에서 이미 제외되었음에도 불구하고 다음 문제가 남아 있다.
- `public-server/package.json:50-53` (`scripts`)의 `test:all` 명령어에 폐기된 `test:ai`(`jest --config apps/ai-service/jest-unit.json`)가 포함되어 있어, 전체 단위 테스트 수행 시 불필요하게 128여 개 레거시 모듈에 대한 테스트가 실행된다.
- `public-server/nest-cli.json:101-109` (`projects.ai-service`)에 폐기된 `apps/ai-service` 프로젝트 정의가 남아 있어 모노레포 관리 대상에 포함되어 있다.
- `.specflow/config.yaml:75-81` (`targets.server.scoped_test`)에 폐기된 `apps/ai-service: "pnpm test:ai"` 항목이 여전히 등록되어 있어 자동화 파이프라인의 불필요한 테스트 게이트 실행을 유발한다.
- `public-server/apps/ai-service/src/main.ts:7-44` (`bootstrap`)을 비롯한 `apps/ai-service` 디렉터리 내의 코드가 더 이상 런타임에 호출되지 않으므로, 테스트 실행 대상 및 모노레포 설정에서 안전하게 분리하고 제외해야 한다.

## 요구사항
- [ ] `public-server/package.json`의 `test:all` 스크립트에서 `pnpm test:ai` 실행을 제거한다.
- [ ] `public-server/package.json`의 `test:ai` 및 `test:ai:cov` 단독 스크립트는 필요시 레거시 검증용으로 유지하거나 안전하게 정리한다.
- [ ] `public-server/nest-cli.json`의 `projects` 항목에서 폐기된 `ai-service` 정의를 제거하거나 비활성화한다.
- [ ] `.specflow/config.yaml`의 `targets.server.scoped_test`에서 `apps/ai-service` 항목을 제거한다.
- [ ] `public-server`에서 `pnpm test:all` 실행 시 `apps/identity`, `apps/payment`, `apps/chat-service`, `apps/gateway` 4개 서비스의 단위 테스트만 정상 실행되어 모두 통과(GREEN)해야 한다.
- [ ] `public-server`의 `pnpm build`가 에러 없이 성공해야 한다.

## 비요구사항 (Out of scope)
- `public-python-server`의 RAG 서비스 로직 수정이나 신규 기능 추가는 이번 범위에 포함하지 않는다.
- `public-server/libs/` 내의 공용 라이브러리 및 Gateway의 AI 프록시 컨트롤러 코드는 수정하지 않는다.
- `public-server/apps/admin-server`의 단위 테스트 추가 작업은 SPEC-031의 범위이므로 이번에 다루지 않는다.

## 백엔드
- `public-server/package.json`
  - `test:all` 스크립트 수정: `pnpm test:identity && pnpm test:payment && pnpm test:chat && pnpm test:gateway`
- `public-server/nest-cli.json`
  - `projects.ai-service` 엔트리 정리
- `.specflow/config.yaml`
  - `targets.server.scoped_test`에서 `apps/ai-service` 매핑 제거

## 수용 기준 (Acceptance Criteria)
- Given `public-server` 저장소 루트 디렉터리에서
- When `pnpm test:all` 명령을 실행하면
- Then `identity`, `payment`, `chat-service`, `gateway` 4개 서비스의 테스트만 실행되고 `ai-service` 테스트는 실행되지 않으며 전체 테스트가 성공해야 한다.
- Given `.specflow/config.yaml`의 타깃 설정에서
- When `server` 타깃의 `scoped_test` 목록을 조회하면
- Then `apps/ai-service` 키가 존재하지 않고 유효한 활성 서비스들만 매핑되어 있어야 한다.
- Given `apps/ai-service` 프로젝트가 `nest-cli.json`에서 정리된 상태에서
- When `pnpm build`를 실행하면
- Then 빌드가 실패하지 않고 정상 완료되어야 한다.

## 참고
- 패키지 스크립트 정의: `public-server/package.json:50-53`
- 폐기 안내 주석: `docker-compose.yml:330`
- Nest CLI 프로젝트 설정: `public-server/nest-cli.json:101-109`
- SpecFlow 스코프 테스트 설정: `.specflow/config.yaml:75-81`
- 레거시 메인 진입점: `public-server/apps/ai-service/src/main.ts:7-44`

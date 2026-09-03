---
id: SPEC-033
title: 프롬프트 목록 조회 시 사용자 격리 필터 적용 및 타인 프롬프트 노출 취약점 수정
status: ready
targets: [python-server, server]
stages: [backend, qa]
priority: high
---

## 배경 / 문제

시스템 프롬프트 버전 관리 체계에서 프롬프트 목록 조회 API가 사용자 격리(`userId`) 필터링 없이 전체 레코드를 반환하여, 특정 프롬프트 이름(`name`)에 속한 타인의 커스텀 시스템 프롬프트가 모두 노출되는 취약점(IDOR/정보 노출)이 존재한다.

1. `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:29-35` (`list`)에서 `GET /ai/prompts/:name` 요청을 ai-service-py로 중계할 때, 인증 세션의 `req.session.uuid`를 전달하지 않고 단순 path parameter(`prompts/${name}`)만 호출한다.
2. `public-python-server/src/ai_service/prompt/router.py:19-23` (`list_versions`)에서 `userId` 쿼리 파라미터를 수신하지 않고 `service.list_versions(name)`를 호출한다.
3. `public-python-server/src/ai_service/prompt/service.py:73-75` (`list_versions`)가 `user_id` 컨텍스트 없이 `self._repo.find_all_by_name(name)`를 호출한다.
4. `public-python-server/src/ai_service/prompt/repository.py:44-47` (`find_all_by_name`)가 `{"name": name}` 조건으로만 MongoDB 컬렉션(`prompt_templates`)을 조회하여, 전역 프롬프트(`userId` 없음)뿐만 아니라 다른 모든 사용자가 생성한 개인 프롬프트(`userId: "other-user-uuid"`) 레코드까지 정렬하여 전부 반환한다.

이로 인해 일반 로그인 사용자가 특정 템플릿 이름(예: `rag-qa-system`)의 목록을 조회하면 다른 사용자들이 설정한 시스템 프롬프트 내용(`content`), 변수 목록, 작성자 식별자(`userId`)를 그대로 열람할 수 있다. 이는 SPEC-026 및 SPEC-030에서 정의한 데이터 소유권 격리 및 안전장치 원칙에 위배된다.

## 요구사항

- [ ] `public-python-server`의 프롬프트 저장소 계층에 사용자 격리 조회 메서드(`find_all_by_name_and_user` 또는 `find_all_by_name`의 `user_id` 옵션)를 추가/개선한다.
- [ ] `user_id`가 지정된 경우, 해당 사용자가 생성한 프롬프트(`userId: user_id`) 및 전역 기본 프롬프트(`userId: {"$exists": False}`)만 필터링하여 반환하고 타인의 프롬프트는 제외한다.
- [ ] `user_id`가 지정되지 않은 경우(관리자 전역 조회 컨텍스트), 전역 프롬프트(`userId: {"$exists": False}`) 목록만 반환한다.
- [ ] `public-python-server/src/ai_service/prompt/router.py`의 `list_versions` 엔드포인트(`GET /prompts/{name}`)에서 `userId` 쿼리 파라미터(`Query(default=None, alias="userId")`)를 지원하도록 수정한다.
- [ ] `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts`의 `list` 메서드(`GET /ai/prompts/:name`)에서 요청자의 `req.session.uuid`를 `userId` 파라미터로 ai-service-py에 전달하도록 수정한다.
- [ ] 프롬프트 목록 조회 시 사용자별 격리가 올바르게 동작하는지 검증하는 단위/통합/E2E 테스트를 추가한다.

## 비요구사항 (Out of scope)

- 프롬프트 다중 슬롯 UI 및 최대 슬롯 개수 제한 구현 (SPEC-026 범위).
- 관리자 권한 RBAC 인가 가드 신설 및 전역 프롬프트 활성화 권한 제어 (SPEC-027 범위).
- 프롬프트 생성/수정/삭제 등 쓰기 인터페이스의 변경.

## 백엔드

- `public-python-server/src/ai_service/prompt/repository.py`:
  - `find_all_by_name` 메서드에 `user_id: str | None = None` 매개변수를 추가하거나 전용 조회 로직을 구성한다.
  - `user_id` 전달 시 MongoDB 쿼리 조건을 `{"name": name, "$or": [{"userId": user_id}, {"userId": {"$exists": False}}]}` 형태로 구성하여 타인 프롬프트가 조회되지 않도록 격리한다.
  - `user_id` 미전달 시 MongoDB 쿼리 조건을 `{"name": name, "userId": {"$exists": False}}`로 구성하여 전역 프롬프트만 반환한다.
- `public-python-server/src/ai_service/prompt/service.py`:
  - `list_versions(name: str, user_id: str | None = None)` 형태로 시그니처를 확장하고 리포지토리에 `user_id`를 전달한다.
- `public-python-server/src/ai_service/prompt/router.py`:
  - `list_versions(name: str, service: PromptServiceDep, user_id: str | None = Query(default=None, alias="userId"))`로 쿼리 파라미터를 수신하여 서비스 계층에 전달한다.
- `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts`:
  - `list(@Param('name') name: string, @Req() req: AuthenticatedRequest)`로 세션 정보를 주입받아 `this.aiServicePy.get({ method: \`prompts/\${name}\`, params: { userId: req.session.uuid } })` 형태로 호출한다.

## 수용 기준 (Acceptance Criteria)

- Given 전역 프롬프트 v1과 사용자 A의 프롬프트 v2, 사용자 B의 프롬프트 v3가 동일한 `rag-qa-system` 이름으로 저장되어 있을 때
  When 사용자 A의 세션(`userId="user-A"`)으로 `GET /ai/prompts/rag-qa-system` 목록 조회를 요청하면
  Then 전역 프롬프트 v1과 사용자 A의 프롬프트 v2만 반환되고, 사용자 B의 프롬프트 v3는 목록에 포함되지 않는다.
- Given 사용자 A와 사용자 B의 프롬프트만 존재하고 전역 프롬프트가 없을 때
  When 사용자 A의 세션으로 `GET /ai/prompts/rag-qa-system` 목록 조회를 요청하면
  Then 사용자 A의 프롬프트만 반환되며 응답 상태 코드 200을 수신한다.
- Given 전역 프롬프트 및 다수 사용자의 프롬프트가 저장되어 있을 때
  When `userId` 파라미터 없이 `GET /prompts/rag-qa-system`을 호출하면
  Then `userId`가 없는 전역 프롬프트 목록만 반환되고 특정 사용자의 개인 프롬프트는 일체 포함되지 않는다.

## 참고

- 게이트웨이 프록시 컨트롤러: `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:29-35`
- Python 라우터: `public-python-server/src/ai_service/prompt/router.py:19-23`
- Python 서비스: `public-python-server/src/ai_service/prompt/service.py:73-75`
- Python 리포지토리: `public-python-server/src/ai_service/prompt/repository.py:44-47`

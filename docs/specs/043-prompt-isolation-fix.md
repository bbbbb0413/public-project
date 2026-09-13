---
id: SPEC-043
title: 프롬프트 버전 목록 조회 시 사용자 격리 필터 적용 및 타인 프롬프트 노출 취약점 수정
status: ready
targets: [python-server, server]
stages: [backend, qa]
priority: high
---

## 배경 / 문제
사용자별 커스텀 시스템 프롬프트 버전 관리 기능에서 프롬프트 버전 목록 조회 API가 사용자 격리(`userId`) 필터링 없이 전체 레코드를 반환하여, 특정 프롬프트 이름(`name`)에 속한 타인의 개인 시스템 프롬프트가 모두 노출되는 IDOR 및 정보 유출 취약점이 존재한다.

1. `public-server/apps/gateway/src/ai/proxy/my-prompt-proxy.controller.ts:48-53` (`listMine`) 및 `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:48-60` (`list`)에서 게이트웨이는 `userId: req.session.uuid`를 쿼리 파라미터로 전달한다.
2. 하지만 `public-python-server/src/ai_service/prompt/router.py:19-23` (`list_versions`)는 `userId` 쿼리 파라미터를 수신하지 않고 `service.list_versions(name)`만을 호출한다.
3. `public-python-server/src/ai_service/prompt/service.py:73-75` (`list_versions`) 역시 `user_id` 컨텍스트 없이 `self._repo.find_all_by_name(name)`를 호출한다.
4. `public-python-server/src/ai_service/prompt/repository.py:44-47` (`find_all_by_name`)는 `{"name": name}` 조건으로만 MongoDB 컬렉션(`prompt_templates`)을 조회하여, 전역 기본 프롬프트뿐만 아니라 다른 모든 사용자가 작성한 개인 프롬프트(`userId: "other-user-uuid"`) 레코드까지 정렬하여 전부 반환한다.

이로 인해 일반 로그인 사용자가 내 프롬프트 목록을 조회(`GET /ai/my-prompt/list`)하거나 프롬프트 목록을 조회(`GET /ai/prompts/:name`)할 때 다른 사용자가 등록한 비공개 프롬프트 본문(`content`), 변수 목록, 작성자 식별자가 그대로 노출되는 심각한 보안 결함이 발생한다.

## 요구사항
- [ ] `public-python-server/src/ai_service/prompt/repository.py`의 프롬프트 조회 로직에 사용자 격리 매개변수(`user_id: str | None = None`)를 추가한다.
- [ ] `user_id`가 지정된 경우(일반 사용자 컨텍스트), 해당 사용자의 프롬프트(`userId: user_id`) 및 전역 프롬프트(`userId: {"$exists": False}`)만 필터링하여 반환하고 타인의 개인 프롬프트는 제외한다.
- [ ] `user_id`가 지정되지 않은 경우(관리자 전역 조회 컨텍스트), 전역 프롬프트(`userId: {"$exists": False}`) 목록만 반환한다.
- [ ] `public-python-server/src/ai_service/prompt/service.py`의 `list_versions` 메서드가 `user_id: str | None = None` 매개변수를 지원하고 리포지토리에 전달한다.
- [ ] `public-python-server/src/ai_service/prompt/router.py`의 `list_versions` 엔드포인트(`GET /prompts/{name}`)에서 `userId` 쿼리 파라미터(`Query(default=None, alias="userId")`)를 수신하여 서비스로 전달한다.
- [ ] 사용자별 프롬프트 목록 격리 동작에 대한 백엔드 단위/통합 테스트를 추가하여 타인 프롬프트가 노출되지 않음을 검증한다.

## 비요구사항 (Out of scope)
- 프롬프트 생성/수정/삭제 등 쓰기 엔드포인트의 비즈니스 로직 변경.
- 프롬프트 다중 슬롯 UI 및 프론트엔드 스타일 변경 (SPEC-026 구현 범위).
- 관리자 권한 가드(`AdminGuard`)의 인증 로직 자체 변경.

## 백엔드
`public-python-server`에서 다음 파일들을 수정한다.

- `src/ai_service/prompt/repository.py`:
  - `find_all_by_name(self, name: str, user_id: str | None = None) -> list[PromptTemplate]` 메서드 시그니처를 확장한다.
  - `user_id`가 주어지면 `{"name": name, "$or": [{"userId": user_id}, {"userId": {"$exists": False}}]}` 쿼리 필터를 적용한다.
  - `user_id`가 `None`이면 `{"name": name, "userId": {"$exists": False}}` 쿼리 필터를 적용한다.
- `src/ai_service/prompt/service.py`:
  - `list_versions(self, name: str, user_id: str | None = None) -> list[PromptTemplate]` 메서드로 `user_id`를 수신하여 리포지토리에 전달한다.
- `src/ai_service/prompt/router.py`:
  - `list_versions(name: str, service: PromptServiceDep, user_id: str | None = Query(default=None, alias="userId"))`로 쿼리 파라미터를 수신하여 `service.list_versions(name, user_id)`를 호출한다.

`public-server`에서 다음 파일의 연동을 확인한다.

- `apps/gateway/src/ai/proxy/my-prompt-proxy.controller.ts`:
  - `listMine`에서 `prompts/${RAG_PROMPT_NAME}` 호출 시 `params: { userId: req.session.uuid }`가 정상 전달되는지 확인한다.
- `apps/gateway/src/ai/proxy/prompt-proxy.controller.ts`:
  - `list`에서 `prompts/${name}` 호출 시 `params: { userId: req.session?.uuid }`가 정상 전달되는지 확인한다.

## 수용 기준 (Acceptance Criteria)
- Given 전역 기본 프롬프트 v1과 사용자 A의 프롬프트 v2, 사용자 B의 프롬프트 v3가 동일한 `rag-qa-system` 이름으로 저장되어 있을 때
  When 사용자 A의 세션(`userId="user-A"`)으로 `GET /prompts/rag-qa-system?userId=user-A` 목록 조회를 요청하면
  Then 전역 기본 프롬프트 v1과 사용자 A의 프롬프트 v2만 반환되고, 사용자 B의 프롬프트 v3는 목록에서 완전히 제외된다.
- Given 사용자 A와 사용자 B의 프롬프트만 존재하고 전역 프롬프트가 없을 때
  When 사용자 A의 세션으로 `GET /prompts/rag-qa-system?userId=user-A` 목록 조회를 요청하면
  Then 사용자 A의 프롬프트만 반환되며 200 OK 응답을 수신한다.
- Given 전역 프롬프트 및 다수 사용자의 프롬프트가 데이터베이스에 저장되어 있을 때
  When `userId` 파라미터 없이 `GET /prompts/rag-qa-system`을 호출하면
  Then `userId`가 없는 전역 프롬프트 목록만 반환되고 특정 사용자의 개인 프롬프트는 단 하나도 반환되지 않는다.
- Given 일치하는 프롬프트가 전혀 없는 이름으로 조회를 요청할 때
  When `GET /prompts/non-existent?userId=user-A`를 호출하면
  Then 오류 없이 빈 배열 `[]`과 200 OK 응답을 반환한다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/prompt/repository.py:44-47` (`find_all_by_name`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/prompt/service.py:73-75` (`list_versions`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/prompt/router.py:19-23` (`list_versions`)
- 관련 정의나 기존 구현: `public-server/apps/gateway/src/ai/proxy/my-prompt-proxy.controller.ts:48-53` (`listMine`)
- 관련 정의나 기존 구현: `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:48-60` (`list`)

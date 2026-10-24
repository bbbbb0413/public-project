---
id: SPEC-084
title: 관리자 전역 프롬프트 관리 프록시의 사용자 격리 오염 수정 및 개인 슬롯 분리
status: ready
targets: [server, python-server]
stages: [backend, qa]
priority: normal
---
## 배경 / 문제
현재 NestJS 게이트웨이의 전역 프롬프트 프록시 컨트롤러와 Python AI 서비스의 프롬프트 저장소 간 사용자 격리 불일치로 인해, 관리자가 설정한 전역 프롬프트가 관리자 개인 프롬프트로 오염되고 일반 사용자의 개인 프롬프트 슬롯에 전역 프롬프트가 혼입되는 심각한 결함이 발생하고 있다.

게이트웨이 `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:35-46` (`PromptProxyController.create`)는 관리자가 전역 시스템 프롬프트를 생성할 때 `userId: dto.userId ?? req.session?.uuid`로 관리자의 세션 식별자를 강제 주입한다. 이로 인해 Python 백엔드 `public-python-server/src/ai_service/prompt/repository.py:46` (`PromptTemplateRepository.find_by_name_and_version`)에서 `userId`가 없는 전역 프롬프트(`userId: {"$exists": False}`)로 저장되어야 할 템플릿이 관리자 특정 개인 계정의 프롬프트로 저장된다.
또한 `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:52-60` (`PromptProxyController.list`), `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:64-72` (`PromptProxyController.getActive`), `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:77-86` (`PromptProxyController.activate`), `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:104-113` (`PromptProxyController.deleteVersion`)에서도 모든 요청에 `params: { userId: req.session?.uuid }`를 무조건 바인딩하고 있다. 그 결과 관리자가 전역 프롬프트를 활성화(`activate`)하더라도 전역 활성 상태(`find_active`)가 아닌 해당 관리자 개인 활성 상태(`find_active_for_user`)만 갱신되어, 일반 사용자에게는 관리자가 구성한 최신 전역 프롬프트가 전혀 반영되지 않고 코드 내장 하드코딩 기본값(버전 0)만 고정 적용된다.

더불어 Python 백엔드 `public-python-server/src/ai_service/prompt/repository.py:62-75` (`PromptTemplateRepository.find_all_by_name`)는 `user_id`가 주어졌을 때 `"$or": [{"userId": user_id}, {"userId": {"$exists": False}}]` 조건으로 조회한다. 이로 인해 일반 사용자가 개인 슬롯 목록을 조회(`GET /ai/my-prompt/list`)할 때 전역 프롬프트까지 함께 반환된다.
프론트엔드 `public-front/src/components/ai/PromptSettingsModal.tsx:31-33` (`PromptSettingsModal`)에서는 반환된 목록의 길이로 10개 슬롯 한도(`promptList.length >= 10`)를 검사하므로, 전역 프롬프트가 일반 사용자의 최대 10개 개인 슬롯 허용량을 부당하게 잠식한다. 또한 사용자가 모달에서 전역 프롬프트 슬롯의 삭제를 시도할 경우 `public-front/src/components/ai/PromptSettingsModal.tsx:123-140` (`PromptSettingsModal.handleDeleteSlot`)가 호출되어 게이트웨이를 거쳐 Python 백엔드의 `delete_by_name_and_version`으로 이어지지만, 사용자의 `userId` 조건과 일치하지 않아 `PromptTemplateNotFoundError`(HTTP 404) 에러를 발생시키며 사용자 화면에 프롬프트 삭제 실패 에러가 노출된다.

이 결함은 이미 배송된 SPEC-026(개인 시스템 프롬프트 관리), SPEC-027(관리자 승인 및 전역 프롬프트), SPEC-078(개인 프롬프트 버전 삭제)의 성과를 지우는 회귀 버그이며, "관리자가 검증된 RAG 시스템 프롬프트를 전체 배포하여 모든 사용자의 답변 품질과 신뢰성을 일관되게 보장한다"는 제품 방향에 직결된다.

## 요구사항
- [ ] 게이트웨이 `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts`의 `create` 메서드에서 `dto.userId`가 명시되지 않은 경우 세션 `uuid`를 대체 주입하지 않고 `undefined`로 전달하여 전역 프롬프트가 `userId` 없이 생성되도록 수정한다.
- [ ] `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts`의 `list`, `getActive`, `activate`, `deleteVersion` 메서드에서 전역 프롬프트 관리 요청 시 세션 `uuid`를 `userId` 파라미터로 강제 주입하지 않고, 쿼리/요청에 지정된 경우에만 전달하도록 수정한다.
- [ ] `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts`의 `deactivateActive`(`DELETE /ai/prompts/:name/active`) 메서드에서 세션 `uuid` 강제 주입을 제거하고 전역 활성 프롬프트 비활성화 요청이 가능하도록 파라미터 전달 방식을 조정한다.
- [ ] Python 백엔드 `public-python-server/src/ai_service/prompt/repository.py`의 `find_all_by_name` 메서드에서 `user_id`가 지정된 경우 `$or` 조건 대신 오직 해당 사용자의 프롬프트(`userId: user_id`)만 조회하여 반환하도록 쿼리를 수정한다.
- [ ] Python 백엔드 `public-python-server/src/ai_service/prompt/service.py`와 `router.py`에서 관리자 전역 프롬프트 비활성화(`user_id`가 `None`인 경우 `deactivate_all_by_name`)를 지원하도록 서비스 메서드 및 라우트 쿼리 매개변수(`user_id: str | None = None`)를 보완한다.
- [ ] 게이트웨이 단위 테스트 `prompt-proxy.controller.spec.ts`를 갱신하여 관리자 전역 프롬프트 생성, 조회, 활성화, 삭제 시 ai-service-py 호출에 불필요한 `userId`가 주입되지 않음을 검증한다.
- [ ] Python 백엔드 단위/통합 테스트를 갱신하여 `find_all_by_name` 호출 시 `user_id`가 지정되었을 때 전역 프롬프트가 사용자 슬롯 목록에 섞이지 않고 엄격히 분리됨을 검증한다.

## 비요구사항 (Out of scope)
- 일반 사용자 개인 프롬프트를 전담하는 게이트웨이 컨트롤러 `public-server/apps/gateway/src/ai/proxy/my-prompt-proxy.controller.ts`와 프론트엔드 모달 UI `public-front/src/components/ai/PromptSettingsModal.tsx`의 기존 규격 및 10개 슬롯 제한 규칙은 변경하지 않는다.
- 관리자 권한을 판정하는 `AdminGuard` 및 인증 가드(`GatewayAuthGuard`)의 내부 검증 로직은 변경하지 않는다.
- 기존 MongoDB에 잘못 저장된 관리자 `userId` 보유 레코드에 대한 데이터베이스 마이그레이션 스크립트는 작성하지 않는다.

## 백엔드
- `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts`:
  - `create`: `data: { ...dto, userId: dto.userId }`로 수정하여 `dto.userId`가 없을 때 `req.session?.uuid`가 폴백 주입되지 않도록 처리.
  - `list`: `@Query('userId') userId?: string`을 선언하고 파라미터가 있을 때만 `params: { userId }`를 전달.
  - `getActive`: `@Query('userId') userId?: string`을 선언하고 파라미터가 있을 때만 `params: { userId }`를 전달.
  - `activate`: `@Query('userId') userId?: string`을 선언하고 파라미터가 있을 때만 `params: { userId }`를 전달.
  - `deactivateActive`: `@Query('userId') userId?: string`을 선언하고 파라미터가 있을 때만 `params: { userId }`를 전달.
  - `deleteVersion`: `@Query('userId') userId?: string`을 선언하고 파라미터가 있을 때만 `params: { userId }`를 전달.
- `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.spec.ts`:
  - 관리자 전역 프롬프트 동작 테스트에서 `params` 또는 `data`에 관리자 `userId`가 주입되지 않고 호출되는지 검증하도록 테스트 단언 갱신.
- `public-python-server/src/ai_service/prompt/repository.py`:
  - `find_all_by_name`: `user_id is not None`일 때 `query = {"name": name, "userId": user_id}`로 변경하여 전역 프롬프트 혼입을 제거.
- `public-python-server/src/ai_service/prompt/service.py`:
  - `deactivate_active_prompt`: `user_id: str | None = None`으로 확장하여 `user_id`가 있으면 `deactivate_active_for_user`, 없으면 `deactivate_all_by_name` 호출.
- `public-python-server/src/ai_service/prompt/router.py`:
  - `reset_active_for_user`: `user_id: str | None = Query(default=None, alias="userId")`로 변경하고 서비스 호출.
- `public-python-server/tests/integration/test_prompt_template_repository_impl.py`:
  - `test_find_all_by_name_user_isolation`에서 `user-A` 조회 시 전역 프롬프트(v1)가 포함되지 않고 본인 프롬프트(v2)만 격리되어 반환되는지 검증하도록 테스트 수정.

## 수용 기준 (Acceptance Criteria)
- Given 관리자 세션으로 로그인된 상태일 때 When `POST /ai/prompts`로 `userId` 없이 프롬프트 신규 생성을 요청하면 Then Python 백엔드에 `userId`가 없는 전역 프롬프트 레코드로 저장되고 201 응답을 반환한다.
- Given 전역 프롬프트 버전 1이 등록된 상태일 때 When 관리자가 `PATCH /ai/prompts/:name/1/activate`를 요청하면 Then `userId` 파라미터 없이 요청이 전달되어 전역 활성 프롬프트로 전환된다.
- Given 전역 프롬프트 2개와 사용자 A의 개인 프롬프트 1개가 등록되어 있을 때 When 사용자 A가 `GET /ai/my-prompt/list`를 호출하면 Then 전역 프롬프트를 제외한 사용자 A 본인의 프롬프트 1개만 반환된다.
- Given 사용자 A가 개인 프롬프트를 전혀 등록하지 않은 상태(0개)일 때 When `GET /ai/my-prompt/list`를 호출하면 Then 전역 프롬프트가 존재하더라도 빈 배열(`[]`)이 반환되고 10개 슬롯 용량을 전혀 차지하지 않는다.
- Given 일반 사용자가 프롬프트 설정 모달을 열었을 때 When 개인 슬롯 목록을 확인하면 Then 본인이 생성한 프롬프트만 노출되고, 임의의 개인 슬롯 삭제 시 404 오류 없이 정상 삭제된다.

## 참고
- 고쳐야 할 자리: `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:35-114 (PromptProxyController)`
- 고쳐야 할 자리: `public-python-server/src/ai_service/prompt/repository.py:62-75 (PromptTemplateRepository.find_all_by_name)`
- 관련 정의 및 컴포넌트: `public-front/src/components/ai/PromptSettingsModal.tsx:31-33 (PromptSettingsModal)`
- 관련 프록시 컨트롤러: `public-server/apps/gateway/src/ai/proxy/my-prompt-proxy.controller.ts:39-114 (MyPromptProxyController)`
- 관련 엔드포인트 라우터: `public-python-server/src/ai_service/prompt/router.py:14-72 (router)`

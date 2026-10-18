---
id: SPEC-078
title: 개인 프롬프트 버전 삭제 엔드포인트 구현 및 버전 단건 조회 사용자 격리 수정
status: done
targets: [python-server]
stages: [backend, qa]
priority: high
---
## 배경 / 문제
SPEC-026과 SPEC-045를 통해 사용자가 AI 답변 스타일(시스템 프롬프트)을 최대 10개 슬롯까지 저장·전환·삭제할 수 있는 다중 슬롯 관리 UI와 게이트웨이 프록시를 배송했다. 프론트엔드 `public-front/src/components/ai/PromptSettingsModal.tsx:123-140` (`handleDeleteSlot`) 및 `public-front/src/api/ai.ts:253-255` (`deleteMyPromptVersion`)는 슬롯 삭제 버튼 클릭 시 `DELETE /ai/my-prompt/{version}`을 호출하며, 게이트웨이 `public-server/apps/gateway/src/ai/proxy/my-prompt-proxy.controller.ts:102-113` (`MyPromptProxyController.deleteMine`) 및 `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:101-113` (`PromptProxyController.deleteVersion`)는 이를 Python 백엔드의 `DELETE /prompts/{name}/{version}?userId=...`로 전달한다.

그러나 정작 Python 백엔드(`public-python-server`)에는 해당 버전 삭제 엔드포인트가 빠져 있고, 단건 버전 조회 시 사용자 격리 조건이 누락되어 이미 배송된 다중 슬롯 관리 기능이 정상 작동하지 않는다.
- `public-python-server/src/ai_service/prompt/router.py:53-60` (`reset_active_for_user`)에는 활성 프롬프트를 비활성화하는 `DELETE /prompts/{name}/active`만 정의되어 있을 뿐, 특정 버전을 삭제하는 `DELETE /prompts/{name}/{version}` 라우트가 존재하지 않는다. `public-python-server/src/ai_service/prompt/service.py:78-94` (`PromptService.activate_prompt`, `PromptService.deactivate_active_prompt`)와 `public-python-server/src/ai_service/prompt/repository.py:40-42` (`PromptTemplateRepository.find_by_name_and_version`)에도 버전 삭제 메서드가 없어, 사용자가 모달에서 `[v{version} 삭제]` 버튼을 누르면 항상 실패하며 슬롯 10개 한도(`public-front/src/components/ai/PromptSettingsModal.tsx:32` (`PromptSettingsModal`))에 도달한 뒤에는 기존 슬롯을 지울 수도 없고 새 프롬프트를 저장할 수도 없다.
- 또한 `public-python-server/src/ai_service/prompt/repository.py:40-42` (`PromptTemplateRepository.find_by_name_and_version`)는 `userId` 조건 없이 `{"name": name, "version": version}`으로만 단건 문서를 조회한다. 다른 사용자나 관리자의 동일 버전 번호(예: `version=1`) 문서가 먼저 조회되면 `public-python-server/src/ai_service/prompt/service.py:78-84` (`PromptService.activate_prompt`)의 `target.user_id != user_id` 검증에 걸려 `PromptTemplateNotFoundError`(HTTP 404)가 발생한다. 그 결과 `public-server/apps/gateway/src/ai/proxy/my-prompt-proxy.controller.ts:57-78` (`MyPromptProxyController.saveMine`)에서 새 프롬프트를 생성한 직후 활성화하거나 기존 슬롯을 활성화할 때 본인의 버전이 존재함에도 404 오류가 발생한다.

이는 "이미 배송된 작업(SPEC-026, SPEC-045)의 성과를 지우는 결함"에 해당하며, 사용자가 문서 기반 RAG 답변의 출력 형식과 검증 기준을 제어하는 개인 프롬프트 설정 흐름을 복구하기 위해 즉시 수정이 필요하다.

## 요구사항
- [ ] `public-python-server/src/ai_service/prompt/repository.py`의 `PromptTemplateRepository.find_by_name_and_version` 메서드에 `user_id: str | None = None` 선택적 매개변수를 추가하여, `user_id`가 주어지면 `{"name": name, "version": version, "userId": user_id}`로 조회하고 `user_id`가 `None`이면 `{"name": name, "version": version, "userId": {"$exists": False}}`로 조회하도록 격리한다.
- [ ] `public-python-server/src/ai_service/prompt/repository.py`의 `PromptTemplateRepository`에 `delete_by_name_and_version(self, name: str, version: int, user_id: str | None = None) -> bool` 메서드를 추가하여 소유자 조건(`userId: user_id` 또는 `userId: {"$exists": False}`)이 일치하는 단일 프롬프트 버전 문서를 삭제하고 삭제 성공 여부를 반환한다.
- [ ] `public-python-server/src/ai_service/prompt/service.py`의 `PromptService.activate_prompt`에서 `self._repo.find_by_name_and_version(name, version, user_id)`를 호출하도록 수정하여, 다른 사용자나 전역 프롬프트에 동일 버전 번호가 존재하더라도 본인의 프롬프트 버전을 정확히 찾아 활성화하도록 한다.
- [ ] `public-python-server/src/ai_service/prompt/service.py`의 `PromptService`에 `delete_prompt(self, name: str, version: int, user_id: str | None = None) -> None` 메서드를 추가하고, 삭제 대상 버전이 존재하지 않거나 소유자가 다를 경우 `PromptTemplateNotFoundError(name, version)`를 발생시킨다.
- [ ] `public-python-server/src/ai_service/prompt/router.py`에 `@router.delete("/{name}/{version}", status_code=204)` 엔드포인트(`delete_version`)를 추가하여 `userId` 쿼리 파라미터(`Query(default=None, alias="userId")`)와 함께 `service.delete_prompt(name, version, user_id)`를 호출하고, `PromptTemplateNotFoundError` 발생 시 HTTP 404를 반환한다.
- [ ] 기존 `DELETE /prompts/{name}/active`(`reset_active_for_user`) 라우트가 `DELETE /prompts/{name}/{version}` 라우트와 경로 충돌 없이 정상 동작하도록 라우트 순서 및 타입(`version: int`)을 유지한다.
- [ ] 단위 테스트 더블(`FakePromptTemplateRepository`) 및 단위·라우터 테스트를 갱신·추가하여 동일 버전 번호 충돌 시 활성화 격리와 프롬프트 버전 삭제(성공, 미존재·타인 소유 시 404, 활성 버전 삭제 후 기본값 폴백)를 검증한다.

## 비요구사항 (Out of scope)
- 프론트엔드(`public-front`) 및 NestJS 게이트웨이(`public-server`) 코드는 이미 `DELETE /prompts/{name}/{version}?userId=...`와 `PATCH /prompts/{name}/{version}/activate?userId=...` 규격으로 구현되어 있으므로 수정하지 않는다.
- SPEC-033 및 SPEC-043에서 정의되고 테스트된 `PromptTemplateRepository.find_all_by_name` 및 `PromptService.list_versions`의 목록 조회 필터 동작(`user_id` 지정 시 본인 프롬프트와 전역 프롬프트 함께 반환)은 변경하지 않는다.
- 기존 MongoDB `prompt_templates` 컬렉션 데이터에 대한 별도 마이그레이션은 수행하지 않는다.

## 백엔드
- `public-python-server/src/ai_service/prompt/repository.py`:
  - `PromptTemplateRepository.find_by_name_and_version`(`repository.py:40-42`) 시그니처를 `async def find_by_name_and_version(self, name: str, version: int, user_id: str | None = None) -> PromptTemplate | None:`으로 확장한다.
  - `user_id is not None`일 때 `{"name": name, "version": version, "userId": user_id}`, `user_id is None`일 때 `{"name": name, "version": version, "userId": {"$exists": False}}` 조건으로 `find_one`을 수행한다.
  - `PromptTemplateRepository.delete_by_name_and_version(self, name: str, version: int, user_id: str | None = None) -> bool`을 추가하고, 위와 동일한 소유자 격리 쿼리로 `self._collection.delete_one(query)`를 호출한 뒤 `result.deleted_count > 0`을 반환한다.
- `public-python-server/src/ai_service/prompt/service.py`:
  - `PromptService.activate_prompt`(`service.py:78-90`)에서 `self._repo.find_by_name_and_version(name, version, user_id)`에 `user_id`를 전달한다.
  - `PromptService.delete_prompt(self, name: str, version: int, user_id: str | None = None) -> None`을 추가하여 `deleted = await self._repo.delete_by_name_and_version(name, version, user_id)`가 `False`이면 `raise PromptTemplateNotFoundError(name, version)`를 수행한다.
- `public-python-server/src/ai_service/prompt/router.py`:
  - `reset_active_for_user`(`router.py:53-60`) 아래에 `@router.delete("/{name}/{version}", status_code=204)` 핸들러 `delete_version(name: str, version: int, service: PromptServiceDep, user_id: str | None = Query(default=None, alias="userId")) -> None`을 추가한다.
  - `PromptTemplateNotFoundError`를 잡아 `HTTPException(status_code=404, detail=str(e))`로 변환한다.
- `public-python-server/tests/unit/prompt/fakes.py` 및 `public-python-server/tests/unit/prompt/test_prompt_service.py`:
  - `FakePromptTemplateRepository.find_by_name_and_version`(`fakes.py:15-19`)이 `user_id: str | None = None`을 받아 `(name, version, user_id)` 키와 일치하는 항목만 반환하도록 수정하고, `delete_by_name_and_version` 메서드를 추가한다.
  - 다중 사용자 동일 버전 번호 활성화 테스트 및 버전 삭제 단위·라우터 테스트를 추가한다.

## 수용 기준 (Acceptance Criteria)
- Given 전역 프롬프트 `v1`(`userId=None`)과 사용자 A의 프롬프트 `v1`(`userId="user-A"`), 사용자 B의 프롬프트 `v1`(`userId="user-B"`)이 모두 `"rag-qa-system"` 이름으로 저장되어 있을 때, When 사용자 B가 `PATCH /prompts/rag-qa-system/1/activate?userId=user-B`를 호출하면, Then 404 오류가 발생하지 않고 사용자 B의 `v1` 프롬프트가 정상 활성화(`isActive: true`)되어 반환된다.
- Given 사용자 A가 `"rag-qa-system"` 이름의 개인 프롬프트 `v1`과 `v2`(활성)를 보유하고 있을 때, When 사용자 A가 `DELETE /prompts/rag-qa-system/2?userId=user-A`를 호출하면, Then HTTP 204 No Content가 반환되고 사용자 A의 `v2` 프롬프트가 삭제되며, 이후 `get_active_prompt("rag-qa-system", user_id="user-A")` 호출 시 전역 활성 프롬프트 또는 하드코딩 기본 프롬프트로 안전하게 폴백된다.
- Given 사용자 A의 개인 프롬프트 `v1`(`userId="user-A"`)이 저장되어 있을 때(경계 케이스), When 다른 사용자 B(`userId="user-B"`)가 `DELETE /prompts/rag-qa-system/1?userId=user-B`를 호출하거나 존재하지 않는 버전 번호(`version=999`)로 삭제를 요청하면, Then HTTP 404 Not Found가 반환되고 사용자 A의 `v1` 프롬프트는 삭제되지 않고 그대로 유지된다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/prompt/router.py:39-60` (`activate`, `reset_active_for_user`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/prompt/service.py:78-94` (`PromptService.activate_prompt`, `PromptService.deactivate_active_prompt`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/prompt/repository.py:40-42` (`PromptTemplateRepository.find_by_name_and_version`)
- 고쳐야 할 자리: `public-python-server/tests/unit/prompt/fakes.py:15-19` (`FakePromptTemplateRepository.find_by_name_and_version`)
- 관련 정의나 기존 구현: `public-server/apps/gateway/src/ai/proxy/my-prompt-proxy.controller.ts:80-113` (`MyPromptProxyController.activateMine`, `MyPromptProxyController.deleteMine`), `public-server/apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:101-113` (`PromptProxyController.deleteVersion`), `public-front/src/components/ai/PromptSettingsModal.tsx:104-140` (`PromptSettingsModal`), `public-front/src/api/ai.ts:253-255` (`deleteMyPromptVersion`)

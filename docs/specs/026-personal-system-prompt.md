---
id: SPEC-026
title: 개인 시스템 프롬프트 관리 — 다중 슬롯 전환, 소유자 필터 누락 취약점 수정
status: done
targets: [server, front]
stages: [backend, frontend]
priority: high
---

## 출처
명세서 "3. AI 개인화 설정" 중 **3.1 개인 시스템 프롬프트 관리**(3.1.1 개인 프롬프트 저장 및 활성화). 관리자 전역 프롬프트는 [[027-admin-approval-and-global-prompt]] 참고 — 같은 컨트롤러 계열이라 함께 확인해야 한다.

## ⚠️ 보안 취약점 (착수 전 최우선 확인)
조사 중 명세서 범위와 무관하게 **실제 정보 노출 취약점**을 발견했다. `PromptProxyController.list()`(`apps/gateway/src/ai/proxy/prompt-proxy.controller.ts:29-35`, `GET /ai/prompts/:name`)가 `public-python-server/.../repository.py:43-46`의 `find_all_by_name`을 그대로 호출하는데, 이 쿼리는 **userId 필터가 전혀 없다**. 즉 로그인한 아무 사용자나 이 엔드포인트로 **다른 사용자의 개인 프롬프트 버전을 전부 조회**할 수 있다 — 명세서 3.1의 수용 기준("사용자는 다른 사용자가 소유한 프롬프트를 활성화하거나 조회할 수 없다")을 정면으로 위반한다.

이 구조는 [[021-payment-history-and-idor-fix|SPEC-021]]에서 고친 결제 IDOR과 같은 패턴(소유권 검증 없이 식별자만으로 조회)이다. 프롬프트 저장 슬롯 구조를 바꾸는 작업(아래 요구사항)과 별개로, **이 취약점 수정은 이번 명세서 작업 범위와 무관하게 독립적으로 먼저 처리하는 걸 권장한다.**

## 현재 구현 상태 (조사 결과)

| 항목 | 상태 | 근거 |
|---|---|---|
| 저장/조회/활성화/초기화 API | ✅ 구현됨 | `apps/gateway/src/ai/proxy/my-prompt-proxy.controller.ts`(`GET/POST/DELETE /ai/my-prompt`), 로직은 `public-python-server/src/ai_service/prompt/service.py` |
| 우선순위(개인 활성 > 전역 활성 > 기본값) | ✅ 구현됨, 명세서와 정확히 일치 | `service.py:38-56` `get_active_prompt` |
| 활성화 시 소유권 검증 | ✅ 구현됨 | `activate_prompt`(`service.py:76-88`)에서 `target.user_id != user_id`면 거부 |
| 목록 조회 시 소유권 필터 | ❌ **취약점** | 위 "보안 취약점" 항목 참고 |
| 다중 프롬프트 저장(최대 10개) + 목록에서 선택 활성화 | ❌ 미구현 | 실제 모델은 **슬롯 1개** — 저장하면 즉시 기존 걸 덮어쓰고 바로 활성화(`my-prompt-proxy.controller.ts:34-53`). `create_prompt`에 개수 제한 로직 없음(`service.py`) |
| 프론트 "내 프롬프트 목록" 화면 | ❌ 미구현 | `AiService.tsx`는 단일 슬롯 편집 UI만 있음 |

## 명세서와 현재 구현 간 상충 / 차이
1. **데이터 모델이 근본적으로 다르다**: 명세서는 "여러 개 저장해두고 그중 하나를 활성화"를 전제하는데, 지금은 "슬롯 1개, 저장=활성화"다. 단순 UI 추가가 아니라 저장 스키마 자체를 바꿔야 한다(현재 프롬프트를 버전 히스토리로 유지하면서 여러 개를 병렬 저장하는 구조로).
2. **취약점은 이번 명세서와 별개 이슈**: 목록 API에 소유권 필터를 추가하는 건 명세서 구현과 상관없이 지금 당장 해도 되는 독립 수정이다. 다중 슬롯 구조로 바꾸면서 이 필터를 놓치면 취약점이 그대로 새 구조에도 옮겨갈 위험이 있으니, 새 목록 API를 설계할 때 처음부터 `WHERE user_id = :userId`를 박아 넣어야 한다.

## 요구사항 (이번 작업 범위)
1. **(선행, 독립 작업)** `PromptProxyController.list()` 또는 그 하위의 `find_all_by_name`에 소유권 필터를 추가해 다른 사용자의 개인 프롬프트가 노출되지 않도록 막는다.
2. 개인 프롬프트를 여러 개(최대 10개) 저장할 수 있도록 저장 API/모델을 확장한다.
3. 저장된 프롬프트 목록에서 하나를 선택해 활성화하는 API/UI를 추가한다(활성 프롬프트는 항상 1개 이하 유지).
4. 프론트에 "내 프롬프트 목록" 화면(이름, 활성 여부, 선택 시 활성화)을 추가한다.
5. 활성 프롬프트를 삭제하면 전역 기본값으로 자동 전환되는 기존 규칙이 다중 슬롯 구조에서도 유지되는지 확인한다.

## 비요구사항 (Out of scope)
- 조직 전역 프롬프트 관리 — [[027-admin-approval-and-global-prompt]].
- 프롬프트 버전 diff/롤백 UI.

## 구현 기록
(구현 완료 후 작성)

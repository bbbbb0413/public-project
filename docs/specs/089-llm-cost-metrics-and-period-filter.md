---
id: SPEC-089
title: LLM 비용 집계 상세 지표(토큰·호출 수) 보존 및 기간별 필터 연동
status: ready
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
`public-python-server/src/ai_service/llm_gateway/repository.py:29-42` (`LlmCostLogRepository.persist`)에서는 LLM 호출 시마다 `model`, `feature`, `promptTokens`, `completionTokens`, `costUsd`, `fallbackUsed`, `attemptedModels`, `createdAt`을 상세히 기록하여 MongoDB에 영속화하고 있다.
또한 게이트웨이 프록시 `public-server/apps/gateway/src/ai/proxy/llm-gateway-proxy.controller.ts:13-23` (`LlmGatewayProxyController.getCosts`)와 파이썬 백엔드 라우터 `public-python-server/src/ai_service/llm_gateway/router.py:16-27` (`get_costs`)는 이미 `from`/`to` 쿼리 파라미터를 받아 기간 조회를 위임할 수 있도록 엔드포인트가 준비되어 있다.
그러나 백엔드 집계 쿼리 `public-python-server/src/ai_service/llm_gateway/repository.py:44-53` (`LlmCostLogRepository.sum_by_model`)에서는 `$group` 시 `{"totalCostUsd": {"$sum": "$costUsd"}}`만 계산하고, 함께 기록된 프롬프트 토큰(`promptTokens`), 완료 토큰(`completionTokens`), 전체 호출 건수(`callCount`)를 모두 버린다. 이에 따라 도메인 모델 및 DTO `public-python-server/src/ai_service/llm_gateway/schemas.py:70-74` (`ModelCostSum`) 및 `schemas.py:88-93` (`CostSummaryItem`)에도 `totalCostUsd` 외의 사용량 지표가 전혀 정의되어 있지 않다.
더불어 프론트엔드 API 클라이언트 `public-front/src/api/aiAdmin.ts:70-73` (`getLlmCosts`)는 파라미터를 지원하지 않고 게이트웨이를 인자 없이 호출하며, 모니터링 컴포넌트 `public-front/src/components/admin/LlmMonitor.tsx:14-22` (`LlmMonitor.loadData`), `LlmMonitor.tsx:57-65` (`LlmMonitor`)에는 기간 선택 UI나 토큰 소모량 표시가 전무하여 관리자가 최근 7일치 모델별 비용 총합 외에는 모델별 실제 토큰 사용량이나 호출 빈도, 원하는 기간별 비용 추이를 전혀 파악할 수 없는 결함이 있다.

## 요구사항
- [ ] `public-python-server/src/ai_service/llm_gateway/schemas.py`의 `ModelCostSum` 및 `CostSummaryItem`에 `prompt_tokens: int` (alias: `promptTokens`), `completion_tokens: int` (alias: `completionTokens`), `total_tokens: int` (alias: `totalTokens`), `call_count: int` (alias: `callCount`) 필드를 추가한다. (기존 필드가 없는 경우를 위해 기본값 0 지원)
- [ ] `public-python-server/src/ai_service/llm_gateway/repository.py`의 `LlmCostLogRepository.sum_by_model` aggregate 파이프라인에서 `$group` 스테이지에 `promptTokens`, `completionTokens`의 합계(`$sum`) 및 호출 건수(`{"$sum": 1}`)를 집계하여 `ModelCostSum` 객체에 매핑한다.
- [ ] `public-python-server/src/ai_service/llm_gateway/schemas.py`의 `CostSummaryOut.of`에서 집계된 토큰 및 호출 수 필드를 `CostSummaryItem`으로 올바르게 변환한다.
- [ ] `public-front/src/api/aiAdmin.ts`의 `LlmCost` 인터페이스에 `promptTokens: number`, `completionTokens: number`, `totalTokens: number`, `callCount: number` 선택적 필드를 추가한다.
- [ ] `public-front/src/api/aiAdmin.ts`의 `getLlmCosts` 함수가 `from` 및 `to` 날짜 문자열(ISO 8601)을 선택적 매개변수로 받아 게이트웨이 쿼리 파라미터(`params: { from, to }`)로 전달하도록 확장한다.
- [ ] `public-front/src/components/admin/LlmMonitor.tsx`에 기간 선택 컨트롤(최근 24시간, 최근 7일, 최근 30일 옵션을 가진 드롭다운 또는 버튼 그룹)을 추가하고, 기간 변경 시 해당 날짜 범위로 `getLlmCosts`를 다시 호출한다.
- [ ] `public-front/src/components/admin/LlmMonitor.tsx`의 모델별 비용 요약 카드 및 테이블에 모델별 총 토큰 수, 프롬프트/완료 토큰 세부치, 호출 횟수를 함께 렌더링한다.
- [ ] 기간 내 비용 기록이 전혀 없는 경우(`items.length === 0`) 테이블 대신 "해당 기간의 LLM 사용 기록이 없습니다." 안내 문구를 정상 표시한다.

## 비요구사항 (Out of scope)
- 모델별 비용 로그 원본 단건 상세 내역(테넌트/세션별 개별 raw log) 실시간 스트리밍
- LLM 모델별 단가(토큰당 USD) 관리자 수동 재계산 또는 수정 기능
- LLM 예산 초과 시 자동 서킷 브레이킹 트리거 로직 변경
- 관리자 권한 변경이나 로그인/인증 체계 변경

## 백엔드
- `public-python-server/src/ai_service/llm_gateway/schemas.py`:
  - `ModelCostSum` 데이터클래스에 `prompt_tokens: int = 0`, `completion_tokens: int = 0`, `call_count: int = 0` 필드 추가
  - `CostSummaryItem` Pydantic 모델에 `prompt_tokens: int = Field(default=0, alias="promptTokens")`, `completion_tokens: int = Field(default=0, alias="completionTokens")`, `total_tokens: int = Field(default=0, alias="totalTokens")`, `call_count: int = Field(default=0, alias="callCount")` 필드 추가
  - `CostSummaryOut.of`에서 `ModelCostSum`의 토큰 및 호출 건수를 `CostSummaryItem`으로 매핑
- `public-python-server/src/ai_service/llm_gateway/repository.py`:
  - `LlmCostLogRepository.sum_by_model` 내 aggregate 쿼리의 `$group` 스테이지에 `totalPromptTokens: {"$sum": "$promptTokens"}`, `totalCompletionTokens: {"$sum": "$completionTokens"}`, `callCount: {"$sum": 1}` 집계 추가
  - 쿼리 결과를 `ModelCostSum`으로 생성할 때 집계된 토큰과 호출 수를 안전하게 주입

## 프론트엔드
- `public-front/src/api/aiAdmin.ts`:
  - `LlmCost` 인터페이스에 `promptTokens?: number`, `completionTokens?: number`, `totalTokens?: number`, `callCount?: number` 필드 추가
  - `getLlmCosts(from?: string, to?: string): Promise<LlmCost[]>` 시그니처 확장 및 쿼리 파라미터 전달
- `public-front/src/components/admin/LlmMonitor.tsx`:
  - 선택된 기간 상태(`periodRange`: `'24h' | '7d' | '30d'`) 관리
  - 상단 툴바에 기간 선택 드롭다운(`select` 또는 라디오/버튼 그룹) 배치
  - 모델별 비용 테이블 헤더에 "호출 수", "프롬프트 토큰", "완료 토큰", "총 토큰" 컬럼 추가 및 데이터 셀 바인딩

## 수용 기준 (Acceptance Criteria)
- Given MongoDB `llm_cost_logs` 컬렉션에 토큰 사용량과 비용이 기록된 로그가 존재할 때
  When `GET /ai/llm-gateway/costs`를 호출하면
  Then 각 모델별 `totalCostUsd`뿐 아니라 `promptTokens`, `completionTokens`, `totalTokens`, `callCount`가 함께 계산되어 정상 반환된다.
- Given 관리자가 LLM 모니터링 화면(`LlmMonitor`)에 접근했을 때
  When 기본값(최근 7일) 또는 기간 선택 옵션(최근 24시간, 최근 30일)을 변경하면
  Then 선택한 기간의 `from`/`to` ISO 날짜 파라미터로 비용 조회가 다시 실행되고 화면의 총 비용 및 모델별 토큰·호출 수 통계가 갱신된다.
- Given 선택한 기간 내에 LLM 사용 로그가 1건도 존재하지 않을 때
  When 데이터를 조회하면
  Then 오류(500/NaN)가 발생하지 않고 총 비용은 `$0.0000`으로 계산되며 "해당 기간의 LLM 사용 기록이 없습니다." 안내 문구가 렌더링된다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/llm_gateway/repository.py:44-53` (`LlmCostLogRepository.sum_by_model`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/llm_gateway/schemas.py:70-74` (`ModelCostSum`), `schemas.py:88-112` (`CostSummaryItem`, `CostSummaryOut`)
- 고쳐야 할 자리: `public-front/src/api/aiAdmin.ts:5-8` (`LlmCost`), `aiAdmin.ts:70-73` (`getLlmCosts`)
- 고쳐야 할 자리: `public-front/src/components/admin/LlmMonitor.tsx:14-22` (`LlmMonitor.loadData`), `LlmMonitor.tsx:57-65` (`LlmMonitor`)
- 관련 기존 구현: `public-server/apps/gateway/src/ai/proxy/llm-gateway-proxy.controller.ts:13-23` (`LlmGatewayProxyController.getCosts`)
- 관련 기존 구현: `public-python-server/src/ai_service/llm_gateway/router.py:16-27` (`get_costs`)

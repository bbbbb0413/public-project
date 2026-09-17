---
id: SPEC-040
title: LLM Circuit Breaker 상태값 대소문자 및 형식 불일치 수정과 배지 렌더링 정상화
status: done
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
현재 백엔드 Python 서비스의 LLM 게이트웨이 스키마 `public-python-server/src/ai_service/llm_gateway/schemas.py:7` (`BreakerStatus`) 및 `public-python-server/src/ai_service/llm_gateway/schemas.py:116-130` (`BreakerStatusOut`)에서는 Circuit Breaker 상태값을 소문자 케밥케이스 형태인 `"closed"`, `"open"`, `"half-open"` 문자열로 정의하고 반환한다.
반면 프론트엔드 타입 정의인 `public-front/src/api/aiAdmin.ts:18-23` (`CircuitBreaker`)에서는 대문자 스네이크케이스 형태인 `'CLOSED' | 'OPEN' | 'HALF_OPEN'`으로 선언되어 있고, 실제 모니터링 컴포넌트인 `public-front/src/components/admin/LlmMonitor.tsx:4-8` (`stateColor`)의 배지 색상 매핑 객체 역시 대문자 키(`CLOSED`, `OPEN`, `HALF_OPEN`)만 정의되어 있다.
이로 인해 `public-front/src/components/admin/LlmMonitor.tsx:94-103` (`LlmMonitor`)에서 백엔드 응답의 `b.status`("open" 또는 "half-open")를 조회할 때 `stateColor[b.status]`가 `undefined`로 평가된다. 그 결과 `stateColor[b.status] ?? stateColor.CLOSED` 폴백에 의해 장애 상태(`open`)나 복구 시도 상태(`half-open`)임에도 불구하고 항상 정상 상태인 `CLOSED`(녹색 배경/텍스트) 배지 스타일로 잘못 렌더링된다.
관리자가 LLM 공급자 장애 및 서킷 브레이커 차단 상태를 실시간으로 모니터링할 때 실제 장애 상황을 정상 작동으로 오인하게 만드는 상태 왜곡 결함이 발생하므로 이를 바로잡아야 한다.

## 요구사항
- [ ] 프론트엔드 Circuit Breaker 상태값 매핑 로직에서 백엔드 반환값(`closed`, `open`, `half-open`) 및 대소문자/구분자 변형에 상관없이 정확한 상태 배지 색상이 매핑되도록 정규화한다.
- [ ] 서킷 브레이커가 `OPEN` / `open` 상태일 때 위험 상태 배지 스타일(빨간색 배경 및 텍스트)이 올바르게 렌더링된다.
- [ ] 서킷 브레이커가 `HALF_OPEN` / `half-open` 상태일 때 경고 상태 배지 스타일(주황색/황색 배경 및 텍스트)이 올바르게 렌더링된다.
- [ ] 서킷 브레이커가 `CLOSED` / `closed` 상태일 때 정상 상태 배지 스타일(녹색 배경 및 텍스트)이 올바르게 렌더링된다.
- [ ] 화면에 표시되는 텍스트를 일관되게 대문자(`CLOSED`, `OPEN`, `HALF_OPEN`)로 포맷팅하여 시각적 가독성을 확보한다.
- [ ] `public-front/src/api/aiAdmin.ts`의 `CircuitBreaker` 인터페이스의 `status` 타입을 백엔드 실제 계약과 호환되도록 정리한다.
- [ ] 상태값 매핑 및 배지 색상 결정 로직에 대한 프론트엔드 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 백엔드 Python 서버의 서킷 브레이커 상태 전이 및 임계치 로직 변경
- 서킷 브레이커 수동 재설정/트리거 관리자 기능 추가
- LLM 모니터링 화면 내 비용 및 레이턴시 등 타 메트릭 차트 추가

## 프론트엔드
- `public-front/src/api/aiAdmin.ts`: `CircuitBreaker` 인터페이스의 `status` 타입을 `string` 또는 백엔드/프론트엔드 공통 유니온 타입(`'closed' | 'open' | 'half-open' | 'CLOSED' | 'OPEN' | 'HALF_OPEN'`)으로 호환성 보장
- `public-front/src/components/admin/LlmMonitor.tsx`:
  - 상태 문자열 정규화 함수(예: 대문자 변환 및 하이픈을 언더스코어로 변환)를 추가하여 `stateColor` 맵 조회 시 대소문자/하이픈 불일치 문제 해결
  - 배지 텍스트 렌더링 시 정규화된 대문자 문자열 표시
  - 알 수 없는 상태값 수신 시 안전한 기본 스타일 폴백 적용

## 수용 기준 (Acceptance Criteria)
- Given LLM 게이트웨이 API가 `status: "open"` 상태인 서킷 브레이커 데이터를 반환할 때, When 관리자가 LLM 모니터링 화면을 조회하면, Then 해당 모델의 배지가 빨간색(`OPEN`) 스타일로 렌더링되어야 한다.
- Given LLM 게이트웨이 API가 `status: "half-open"` 상태인 서킷 브레이커 데이터를 반환할 때, When 관리자가 LLM 모니터링 화면을 조회하면, Then 해당 모델의 배지가 주황색(`HALF_OPEN`) 스타일로 렌더링되어야 한다.
- Given LLM 게이트웨이 API가 `status: "closed"` 상태인 서킷 브레이커 데이터를 반환할 때, When 관리자가 LLM 모니터링 화면을 조회하면, Then 해당 모델의 배지가 녹색(`CLOSED`) 스타일로 렌더링되어야 한다.
- Given 예상치 못한 상태값(예: `""` 빈 문자열, `null`, 또는 정의되지 않은 문자열)이 전달될 때, When 배지를 렌더링하면, Then 런타임 예외 없이 기본 CLOSED 스타일로 안전하게 폴백되어야 한다.

## 참고
- 고쳐야 할 자리: `public-front/src/components/admin/LlmMonitor.tsx:4-8` (`stateColor`), `:94-103` (`LlmMonitor`)
- 프론트 타입 정의: `public-front/src/api/aiAdmin.ts:18-23` (`CircuitBreaker`)
- 백엔드 스키마 정의: `public-python-server/src/ai_service/llm_gateway/schemas.py:7` (`BreakerStatus`), `:116-130` (`BreakerStatusOut`)

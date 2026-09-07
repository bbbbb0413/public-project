---
id: SPEC-037
title: RAG 비평 파싱 실패 시 기본 신뢰도 왜곡 방지 및 안전한 폴백 처리
status: ready
targets: [python-server]
stages: [backend, qa]
priority: normal
---

## 배경 / 문제
현재 `public-python-server/src/ai_service/rag/application/critique_generator_service.py:122-147` (`CritiqueGeneratorService._parse_critique`) 및 `public-python-server/src/ai_service/rag/application/critique_generator_service.py:149-151` (`CritiqueGeneratorService._fallback_critique`)에서 LLM의 비평 응답 파싱이 실패하거나 JSON 디코딩에 실패할 경우, 무조건 `Critique.of(True, [], "", 0.7)`을 반환하여 정상 통과로 간주하고 삼켜버리는 문제가 있다.

설정상 기본 임계값(`public-python-server/src/ai_service/core/config.py:74` 의 `agentic_confidence_threshold: float = 0.6`)보다 높은 `0.7`이 하드코딩되어 있어, `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:126-131` (`AgenticAskUseCase.execute`)의 `critique.is_satisfied(command.confidence_threshold)` 조건을 무조건 만족하게 된다.

이로 인해 다음과 같은 결함이 발생한다:
- LLM이 마크다운 코드블록 래핑 실패, 비정형 텍스트 출력, 또는 환각 응답을 내어 JSON 파싱에 실패했을 때, 검증되지 않은 불완전하거나 틀린 답변이 70% 신뢰도의 정상 답변으로 둔갑하여 사용자에게 전달된다.
- 예산(`IterationBudgetProps`) 및 반복 기회가 남아 있음에도 불구하고 재검색(`refining`)이나 쿼리 정제 루프를 전혀 시도하지 못하고 즉시 종료된다.
- SPEC-010 / SPEC-011 / SPEC-018을 통해 화면과 세션 DB에 영속화되는 신뢰도 배지가 허위 높은 신뢰도(70%)로 표시되어 "답변을 믿을 수 있게 만든다"는 제품의 핵심 가치와 상치된다.

## 요구사항
- [ ] `CritiqueGeneratorService._parse_critique`에서 JSON 정규식 매칭 실패 또는 `json.JSONDecodeError` 발생 시, 왜곡된 높은 신뢰도(`0.7`) 및 `answered=True`를 반환하지 않고 검증 실패 상태(`answered=False`, `confidence=0.0`, `missing=["비평 응답 분석 실패"]`)를 나타내는 안전한 폴백 객체를 반환한다.
- [ ] `_parse_critique`의 `confidence` 필드 누락 또는 잘못된 타입/범위 초과 시 기본값을 0.7 대신 0.0으로 처리한다.
- [ ] `CritiqueGeneratorService`에 파싱 실패 및 예외 상황에 대한 경고 로그(`logger.warning`)를 남겨 추적 가능하도록 한다.
- [ ] `CritiqueGeneratorService`의 파싱 실패 폴백 및 유효성 검증 로직에 대한 단위 테스트를 추가한다.

## 비요구사항 (Out of scope)
- RAG 비평 프롬프트 템플릿의 전면 교체나 Few-shot 예시 대량 추가
- 신규 LLM 공급자 연동 또는 외부 서드파티 평가 프레임워크(Ragas 등) 도입
- 프론트엔드 UI 컴포넌트나 게이트웨이 프로토콜의 스키마 변경

## 백엔드
- `public-python-server/src/ai_service/rag/application/critique_generator_service.py`
  - `_fallback_critique`: `Critique.of(answered=False, missing=["비평 파싱 실패"], next_query="", confidence=0.0)` 형태로 변경하여 비평 실패 시 조기 정상 종료를 방지하고 재시도 루프를 유도.
  - `_parse_critique`: `json.loads` 실패 시 에러 로깅 후 변경된 fallback 반환, 파싱된 dict 내 필드 검증 강화.
- `public-python-server/tests/unit/rag/test_critique_generator_service.py`
  - 유효하지 않은 JSON 응답, 필드 누락, 빈 문자열 응답 시 안전하게 `answered=False`, `confidence=0.0` 폴백이 반환되는지 검증하는 단위 테스트 스위트 작성.

## 수용 기준 (Acceptance Criteria)
- Given LLM의 비평 스트림 결과가 유효하지 않은 JSON 형식이거나 빈 문자열일 때, When `CritiqueGeneratorService.generate`를 호출하면, Then `Critique` 객체의 `answered`는 `False`이고 `confidence`는 `0.0`이어야 한다.
- Given 비평 응답 JSON에 `confidence` 필드가 누락되었거나 음수/1초과의 잘못된 값일 때, When `CritiqueGeneratorService.generate`를 호출하면, Then `Critique` 객체의 `confidence`는 `0.0`으로 정규화되어 반환되어야 한다.
- Given 에이전틱 질의 실행 중 1차 비평 파싱이 실패했을 때, When 반복 예산(`budget`)이 남아있다면, Then `AgenticAskUseCase`는 1차에서 즉시 반환하지 않고 다음 반복(refining)을 수행해야 한다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/application/critique_generator_service.py:122-152` (`CritiqueGeneratorService._parse_critique`, `CritiqueGeneratorService._fallback_critique`)
- 관련 정의 및 임계값 설정: `public-python-server/src/ai_service/core/config.py:71-75` (`Settings.agentic_confidence_threshold`)
- 사용처 루프: `public-python-server/src/ai_service/rag/application/agentic_ask_use_case.py:120-138` (`AgenticAskUseCase.execute`)
- 모델 스키마 정의: `public-python-server/src/ai_service/rag/schemas.py:223-247` (`Critique`)

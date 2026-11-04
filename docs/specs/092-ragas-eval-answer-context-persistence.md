---
id: SPEC-092
title: RAGAS 품질 평가 결과의 답변·컨텍스트 영속화 및 관리자 상세 열람 연동
status: done
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
현재 지식베이스 기반 질의응답 시스템은 생성된 답변의 신뢰성을 검증하기 위해 RAGAS 기반 품질 평가 도메인(`RagasEvalService`)과 관리자 모니터링 화면(`RagasEvalPanel`)을 구축해 두었다. 그러나 실제 운영 환경에서 평가 결과를 저장하고 열람하는 과정에 데이터 유실과 흐름 단절이 존재하여 시스템 답변의 신뢰도를 관리자가 올바르게 진단할 수 없다.
`public-python-server/src/ai_service/observability/service.py:71-90` (`RagasEvalService.evaluate`)에서는 `RagasEvalPayload`로 `question`, `answer`, `contexts`를 모두 전달받아 충실도(`faithfulness`), 관련도(`answer_relevancy`), 문맥 정확도(`context_precision`)를 산출하지만, 리포지토리 영속화 호출 시 `RagasEvaluation` 객체 생성 과정에서 `answer`와 `contexts`를 버리고 질문과 점수 3종만 저장한다.
`public-python-server/src/ai_service/observability/schemas.py:16-23` (`RagasEvaluation`) 도메인 데이터클래스와 `public-python-server/src/ai_service/observability/schemas.py:32-51` (`RagasEvaluationOut`) DTO 스키마에 `answer` 및 `contexts` 필드가 정의되어 있지 않아 백엔드 내부 모델 단계에서 답변 본문과 근거 청크가 유실된다.
`public-python-server/src/ai_service/observability/repository.py:14-24` (`RagasEvaluationRepository.persist`) 및 `public-python-server/src/ai_service/observability/repository.py:26-39` (`RagasEvaluationRepository.find_recent`)에서도 MongoDB 컬렉션(`ragas_evaluations`)에 답변과 컨텍스트 필드를 저장하지 않으며 조회 시 복원하지 않는다.
`public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:209-235` (`AskRequestedConsumer._process`)에서는 질의 처리가 성공적으로 끝나 세션 턴을 영속화하고 완료 이벤트를 발행하지만, `RagasEvalService.evaluate` 평가 호출이 파이프라인에 연결되어 있지 않아 실제 질문 처리 흐름에서 품질 평가 레코드가 적재되지 않는다.
`public-front/src/components/admin/RagasEvalPanel.tsx:131-164` (`RagasEvalPanel`) 관리자 화면에서는 목록 테이블에 질문과 점수 막대만 표시되며, 특정 평가 항목의 점수가 낮게 나왔을 때(예: faithfulness가 0%로 떨어져 할루시네이션이 의심되는 경우) 어떤 답변이 생성되었고 어떤 검색 청크를 근거로 평가되었는지 확인할 수 있는 상세 열람 수단이 부재하다.
이로 인해 "답변을 믿을 수 있게 만든다"는 이번 분기 핵심 목표에도 불구하고, 관리자가 불량 답변의 원인이 검색 컨텍스트의 부정확성인지 LLM의 환각인지 식별할 수 없는 상태다.

## 요구사항
- [ ] `public-python-server`의 `RagasEvaluation` 도메인 엔티티에 `answer: str` 및 `contexts: list[str]` 필드를 추가한다.
- [ ] `public-python-server`의 `RagasEvalService.evaluate`에서 평가 점수 산출 후 영속화 시 `payload.answer`와 `payload.contexts`를 `RagasEvaluation` 인스턴스에 온전히 보존하여 전달한다.
- [ ] `public-python-server`의 `RagasEvaluationRepository.persist`에서 MongoDB 문서에 `answer` 및 `contexts` 필드를 저장한다.
- [ ] `public-python-server`의 `RagasEvaluationRepository.find_recent`에서 `answer`와 `contexts` 필드를 복원하며, 기존 필드가 없는 레거시 문서의 경우 `answer=""`, `contexts=[]`로 안전하게 폴백한다.
- [ ] `public-python-server`의 `RagasEvaluationOut` 스키마에 `answer: str` 및 `contexts: list[str]` 필드를 추가하고 `from_domain` 매핑을 지원한다.
- [ ] `public-python-server`의 `AskRequestedConsumer._process`에서 정상 답변 생성 완료 시(`not was_cancelled`이며 답변 텍스트가 수신된 경우) `RagasEvalService.evaluate`를 안전하게 호출하여 평가 레코드를 적재한다. 평가 실패 시 전체 질의 흐름에 영향을 주지 않도록 예외를 로깅하고 격리한다.
- [ ] `public-front`의 `aiAdmin.ts` 내 `RagasEval` 인터페이스에 `answer?: string` 및 `contexts?: string[]` 필드를 추가한다.
- [ ] `public-front`의 `RagasEvalPanel.tsx`에서 평가 목록의 행을 클릭하거나 상세 버튼을 클릭했을 때 선택된 평가 건의 질문, 답변 본문(`answer`), 참조된 컨텍스트 목록(`contexts`), 평가 점수 3종을 확인할 수 있는 상세 보기 모달 또는 펼침 뷰를 제공한다.
- [ ] 상세 모달 또는 펼침 뷰에서 참조 컨텍스트가 비어 있거나 없을 경우 "참조된 지식베이스 컨텍스트가 없습니다." 안내 문구를 렌더링한다.
- [ ] 상세 모달에서 닫기 버튼, 배경 클릭, 또는 `Escape` 키 입력 시 상세 보기가 닫힌다.

## 비요구사항 (Out of scope)
- RAGAS 평가 점수 알고리즘 자체를 외부 RAGAS 파이썬 라이브러리로 교체하거나 새로운 평가지표(answer_correctness 등)를 추가하지 않는다.
- 일반 사용자 화면(`AiService.tsx`)에 RAGAS 지표나 관리자용 평가 점수를 노출하지 않는다.
- RAG 질의 처리 시 LLM 기반 RAGAS 평가를 강제 활성화하지 않으며, 기존의 설정 플래그(`ragas_llm_eval_enabled`) 및 휴리스틱 폴백 구조를 유지한다.
- MongoDB 컬렉션의 과거 평가 레코드에 대한 데이터 마이그레이션 스크립트를 작성하지 않는다. (스키마 및 리포지토리 레벨의 기본값 폴백으로 해결한다)

## 백엔드
- `public-python-server/src/ai_service/observability/schemas.py`:
  - `RagasEvaluation` 데이터클래스에 `answer: str = ""` 및 `contexts: list[str] = field(default_factory=list)` 필드 추가
  - `RagasEvaluationOut` 모델에 `answer: str = Field(default="")` 및 `contexts: list[str] = Field(default_factory=list)` 필드 추가
  - `RagasEvaluationOut.from_domain` 매핑 로직에 `answer`와 `contexts` 연결
- `public-python-server/src/ai_service/observability/service.py`:
  - `RagasEvalService.evaluate`에서 `RagasEvaluation` 생성 시 `answer=payload.answer`, `contexts=payload.contexts` 전달
- `public-python-server/src/ai_service/observability/repository.py`:
  - `RagasEvaluationRepository.persist`에서 `answer: evaluation.answer`, `contexts: evaluation.contexts` 저장
  - `RagasEvaluationRepository.find_recent`에서 `answer=r.get("answer", "")`, `contexts=r.get("contexts", [])`로 복원
- `public-python-server/src/ai_service/rag/rag_composition.py`:
  - `RagComposition`에 `ragas_eval_service: RagasEvalService | None = None` 필드 추가
  - `build_rag_composition`에서 `RagasEvalService(RagasEvaluationRepository(mongo_db), settings)`를 조립하여 주입
- `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py`:
  - `_process` 내 정상 답변 완료 시점에 `composition.ragas_eval_service.evaluate`를 `RagasEvalPayload(trace_id=message.job_id, question=message.question, answer=full_response, contexts=context_texts)`로 호출

## 프론트엔드
- `public-front/src/api/aiAdmin.ts`:
  - `RagasEval` 인터페이스에 `answer?: string; contexts?: string[];` 추가
- `public-front/src/components/admin/RagasEvalPanel.tsx`:
  - 평가 목록 각 행에 상세 보기 진입 인터랙션(행 클릭 또는 [상세] 버튼) 추가
  - 선택된 평가 건의 `selectedEval` 상태 관리
  - 상세 모달 컴포넌트 추가:
    - 질문 전문
    - 생성된 AI 답변 본문
    - 세부 평가 지표 3종 수치
    - 참조된 컨텍스트 스니펫 목록 (각 컨텍스트 번호 및 본문 카드 표시)
    - 닫기 버튼 및 키보드 `Escape` 이벤트 지원

## 수용 기준 (Acceptance Criteria)
- Given `RagasEvalService`에 답변 본문과 2개의 컨텍스트 조각이 포함된 `RagasEvalPayload`가 주어졌을 때
  When `evaluate` 메서드가 실행되면
  Then 리포지토리에 저장된 `RagasEvaluation` 엔티티의 `answer`와 `contexts` 필드에 전달된 값이 온전히 보존되어야 한다.
- Given 과거에 생성되어 `answer`와 `contexts` 필드가 없는 레거시 MongoDB 평가 문서가 존재할 때
  When `RagasEvaluationRepository.find_recent`를 호출하면
  Then 오류 없이 `answer=""`, `contexts=[]`가 설정된 도메인 객체로 정상 복원되어야 한다.
- Given 관리자 화면의 `RagasEvalPanel`에서 평가 목록이 렌더링되어 있을 때
  When 관리자가 특정 평가 항목의 [상세] 버튼 또는 행을 클릭하면
  Then 해당 평가의 질문, 생성된 답변 본문, 참조 컨텍스트 스니펫 목록이 표시되는 상세 모달이 열려야 한다.
- Given 상세 모달에 표시할 참조 컨텍스트 목록이 빈 배열(`contexts: []`)이거나 누락된 평가 건일 때
  When 상세 모달을 열면
  Then 오류 없이 "참조된 지식베이스 컨텍스트가 없습니다." 안내 문구가 정상 렌더링되어야 한다.
- Given 상세 모달이 열려 있는 상태일 때
  When 관리자가 [닫기] 버튼을 누르거나 `Escape` 키를 누르면
  Then 상세 모달이 닫히고 목록 화면으로 복귀해야 한다.
- Given RAG 질의 처리 중 사용자가 중단 버튼을 눌러 답변 생성이 취소되었을 때(`was_cancelled === True`)
  When 질의 컨슈머 처리가 종료되면
  Then 취소된 불완전 답변에 대해서는 RAGAS 평가가 트리거되지 않아야 한다.

## 참고
- 고쳐야 할 자리: `public-python-server/src/ai_service/observability/schemas.py:16-23` (`RagasEvaluation`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/observability/service.py:71-90` (`RagasEvalService.evaluate`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/observability/repository.py:14-39` (`RagasEvaluationRepository.persist`, `find_recent`)
- 고쳐야 할 자리: `public-python-server/src/ai_service/rag/infrastructure/messaging/ask_requested_consumer.py:209-235` (`AskRequestedConsumer._process`)
- 고쳐야 할 자리: `public-front/src/api/aiAdmin.ts:80-87` (`RagasEval`)
- 고쳐야 할 자리: `public-front/src/components/admin/RagasEvalPanel.tsx:131-164` (`RagasEvalPanel`)
- 관련 정의나 기존 구현: `public-python-server/src/ai_service/observability/schemas.py:7-13` (`RagasEvalPayload`)
- 관련 정의나 기존 구현: `public-server/apps/gateway/src/ai/proxy/observability-proxy.controller.ts:13-20` (`ObservabilityProxyController.getEvals`)

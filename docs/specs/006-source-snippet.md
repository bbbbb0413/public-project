---
id: SPEC-006
title: 답변 근거 미리보기
status: done
targets: [python-server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제

답변과 함께 출처가 내려온다. 그런데 `public-front/src/api/ai.ts:26-30` 의
`SourceRef` 가 담는 것은 셋뿐이다.

```ts
export interface SourceRef {
  fileName: string;
  chunkIndex: number;
  documentId: string;
}
```

파일 이름과 몇 번째 조각인지만 안다. **그 조각에 무엇이 적혀 있었는지는 알 수
없다.** 사용자가 "이 답변이 정말 저 문서에 근거한 것인가" 를 확인하려면 원본
문서를 따로 열어 해당 위치를 찾아야 한다.

RAG 시스템에서 답변의 근거를 확인할 수 없다는 것은 곧 환각을 발견할 수 없다는
의미다. 그럴듯하게 지어낸 문장과 문서에서 가져온 문장이 화면에서 똑같이 보인다.

검색된 조각의 본문은 백엔드가 이미 손에 쥐고 있다.
`agentic_ask_use_case.py:61-68` 에서 `chunks` 를 받아 출처 목록을 만들 때
본문을 버리고 메타데이터만 남긴다. 내려보내지 않을 이유가 없다.

## 요구사항

- [ ] `SourceRef` 에 `snippet` 필드를 더해 검색된 조각의 본문 일부를 담는다
- [ ] `snippet` 은 최대 300자로 자르고, 잘렸으면 끝에 말줄임표를 붙인다
- [ ] 조각 본문에 비밀정보·개인정보가 있으면 마스킹한 뒤 내려보낸다
- [ ] 프론트엔드에서 출처를 눌러 펼치면 `snippet` 이 보인다
- [ ] 기본은 접힌 상태다. 답변을 읽는 흐름을 끊지 않는다
- [ ] `snippet` 이 없는 응답(옛 백엔드)을 받아도 화면이 깨지지 않는다
- [ ] 위 규칙을 검증하는 테스트를 작성한다

## 비요구사항 (Out of scope)

- 답변의 특정 문장과 출처를 잇는 문장 단위 인용은 하지 않는다. 이번에는 답변 전체에 대한 출처 목록까지만 다룬다
- 원본 문서 전문을 여는 뷰어나 다운로드 기능
- 조각 안에서 질의어를 강조 표시하는 처리
- 출처의 관련도 점수 노출
- 인제스트 단계에서 조각을 나누는 방식 변경

## 백엔드

`public-python-server` 에서 출처를 만드는 지점을 고친다.

`src/ai_service/rag/application/agentic_ask_use_case.py` 와
`src/ai_service/rag/application/ask_use_case.py` 두 곳 모두 출처 목록을
만든다. 양쪽이 같은 형태를 내보내야 프론트엔드가 갈라지지 않는다.

**마스킹을 빠뜨리지 않는다.** `SecretPiiScanner` 가 이미 답변 토큰에 적용되고
있다(`agentic_ask_use_case.py:87`). 조각 본문은 원본 문서에서 그대로 온 것이라
오히려 위험이 크다. 같은 스캐너를 통과시킨다.

자르는 길이 300자는 상수로 둔다.

## 프론트엔드

`public-front/src/api/ai.ts`
`SourceRef` 에 `snippet?: string` 을 더한다. 선택 필드로 두어 옛 응답과도
맞물리게 한다.

`public-front/src/components/AiService.tsx`
출처 항목을 눌러 펼치면 본문이 보이는 형태로 만든다. 접기·펼치기 상태는
출처마다 따로 관리한다.

키보드로도 펼칠 수 있어야 한다. 마우스 클릭만 받는 `div` 를 쓰지 말고
`button` 이나 `details` 를 쓴다.

## 수용 기준 (Acceptance Criteria)

- Given 답변에 출처 3건이 딸린 경우 When 사용자가 첫 번째 출처를 누르면 Then 그 조각의 본문이 펼쳐지고 나머지 둘은 접힌 상태를 유지한다
- Given 조각 본문이 500자인 경우 When 출처를 펼치면 Then 300자까지만 보이고 끝에 말줄임표가 붙는다
- Given 조각 본문에 `010-1234-5678` 이 포함된 경우 When 출처를 펼치면 Then 그 번호가 마스킹된 상태로 보인다
- Given `snippet` 필드가 없는 응답 When 출처 목록을 그리면 Then 파일 이름만 표시되고 오류가 나지 않는다
- Given 출처 항목에 키보드 포커스가 있을 때 When Enter 를 누르면 Then 마우스로 누른 것과 같이 펼쳐진다

## 참고

- 출처 생성 지점: `src/ai_service/rag/application/agentic_ask_use_case.py:61-68`
- 마스킹: `src/ai_service/rag/application/filter/secret_pii_scanner.py`
- 프론트 타입: `public-front/src/api/ai.ts:26-30`
- 함께 보면 좋은 문서: `005-agentic-progress.md` (추론 과정 노출)

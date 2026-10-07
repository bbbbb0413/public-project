---
id: SPEC-061
title: 지식베이스 문서 목록 업로드 일시 표시 및 일시 포맷팅 통일
status: done
targets: [front]
stages: [frontend, qa]
priority: normal
---
## 배경 / 문제
현재 지식베이스 문서 관리 영역(`public-front/src/components/AiService.tsx:805-853`, `AiService`)의 문서 목록 테이블(`doc-table`)에는 파일명, 처리 상태, 청크 수, 작업 버튼만 표시되고 문서가 언제 등록되었는지 확인 가능한 업로드 일시 정보가 누락되어 있다.

1. 백엔드 API 스키마 `public-python-server/src/ai_service/knowledge/schemas.py:152-171` (`DocumentOut`) 및 게이트웨이 프록시(`GET /ai/knowledge/documents`)는 각 문서의 생성 일시(`createdAt`, ISO 8601 형식)를 이미 정상적으로 제공하고 있다.
2. 프론트엔드 API 인터페이스 `public-front/src/api/ai.ts:115-121` (`DocumentItem`) 및 컴포넌트 내부 모델 `public-front/src/components/AiService.tsx:40-48` (`DocumentInfo`)에도 `createdAt: string` 필드가 이미 정의되어 수신되고 있다.
3. 그러나 `public-front/src/components/AiService.tsx:811-850` (`AiService`)의 테이블 헤더(`<thead>`) 및 본문(`<tbody>`)에서 `createdAt` 렌더링이 누락되어 있어 사용자가 동일한 파일명의 개정 문서나 여러 문서를 관리할 때 최신 문서 등록 시점을 구분하기 어렵다.
4. 프로젝트의 공통 일시 표기 규약(SPEC-003)으로 정의된 `public-front/src/utils/date.ts:71-140` (`formatDateTime`) 함수를 활용해 `YYYY-MM-DD HH:mm:ss` 형식으로 일관되게 포맷팅하여 표시할 필요가 있다.

## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 지식베이스 문서 목록 테이블(`doc-table`) 헤더에 "업로드 일시" 컬럼(`<th>업로드 일시</th>`)을 추가한다.
- [ ] `public-front/src/components/AiService.tsx`의 각 문서 행(`<tr>`)에 포맷팅된 업로드 일시 셀(`<td>`)을 렌더링한다.
- [ ] 문서의 `createdAt` 일시 포맷팅은 `public-front/src/utils/date.ts`의 `formatDateTime` 유틸리티 함수를 사용해 `YYYY-MM-DD HH:mm:ss` 형식으로 일관되게 표시한다.
- [ ] `createdAt` 값이 없거나(undefined, null, 빈 문자열) 유효하지 않은 날짜 형식인 경우 빈 값(`""`) 또는 대시(`"-"`)를 안전하게 표시하여 화면 깨짐을 방지한다.
- [ ] 지식베이스 문서 목록 테이블 렌더링 시 업로드 일시 표시 및 포맷팅 처리에 대한 단위 테스트(`AiService.test.tsx`)를 추가 또는 보강한다.

## 비요구사항 (Out of scope)
- 백엔드 Python 서버(`public-python-server`) 및 게이트웨이(`public-server`)의 API 응답 스키마 변경은 진행하지 않는다. (기존 `createdAt` 필드 그대로 사용)
- 문서 목록의 업로드 일시 기준 정렬(Sorting) 기능이나 페이지네이션은 이번 범위에 포함하지 않는다.
- 문서 수정 일시(updatedAt) 추가나 메타데이터 편집 기능은 다루지 않는다.

## 프론트엔드
`public-front/src/components/AiService.tsx`
- `src/utils/date.ts`에서 `formatDateTime` 함수를 import한다.
- 문서 목록 테이블(`doc-table`)의 `<thead>`에 `<th>업로드 일시</th>` 컬럼 헤더를 추가한다.
- `<tbody>`의 문서 렌더링 루프에서 `<td>{formatDateTime(doc.createdAt) || '-'}</td>` 셀을 청크 수 컬럼 앞/뒤 적절한 위치에 추가한다.
- 테이블 레이아웃 및 반응형 스타일에 이상이 없는지 확인한다.

## 수용 기준 (Acceptance Criteria)
- Given 사용자가 지식베이스 문서 관리 영역에 진입하여 문서 목록을 불러왔을 때
  When 문서 목록 테이블(`doc-table`)이 렌더링되면
  Then 테이블 헤더에 "업로드 일시" 컬럼이 표시되고, 각 문서 행에 `YYYY-MM-DD HH:mm:ss` 형식으로 변환된 생성 일시가 표시된다.
- Given 특정 문서의 `createdAt` 값이 `"2026-10-02T01:30:00.000Z"`와 같은 유효한 ISO 8601 문자열일 때
  When 문서 행이 렌더링되면
  Then `formatDateTime`을 거쳐 로컬 시간대 기준 `YYYY-MM-DD HH:mm:ss` 문자열로 변환되어 테이블에 표시된다.
- Given 레거시 데이터 등으로 인해 문서 객체의 `createdAt`이 `null`, `undefined`, 또는 빈 문자열(`""`)이거나 유효하지 않은 문자열인 경우
  When 문서 행이 렌더링되면
  Then 오류를 발생시키지 않고 빈 문자열 또는 `"-"`로 안전하게 폴백 표시된다.
- Given 업로드된 문서가 없는 빈 상태(`documents.length === 0`)일 때
  When 문서 목록 영역이 렌더링되면
  Then 기존의 "업로드된 문서가 없습니다." 안내 문구가 정상 노출된다.

## 참고
- 고쳐야 할 자리: `public-front/src/components/AiService.tsx:811-850` (`AiService`)
- 모델 정의: `public-front/src/components/AiService.tsx:40-48` (`DocumentInfo`)
- API 인터페이스: `public-front/src/api/ai.ts:115-121` (`DocumentItem`)
- 일시 포맷팅 함수: `public-front/src/utils/date.ts:71-140` (`formatDateTime`)
- 백엔드 스키마 정의: `public-python-server/src/ai_service/knowledge/schemas.py:152-171` (`DocumentOut`)

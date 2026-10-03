---
id: SPEC-063
title: 지식베이스 문서 목록 검색(필터링) 및 업로드 일시 표시
status: ready
targets: [front]
stages: [frontend, qa]
priority: normal
---

## 배경 / 문제
지식베이스 문서는 RAG 질의응답의 근거가 되는 핵심 자산이다. 사용자가 문서를 업로드하여 지식베이스를 구축하지만, 현재 프론트엔드 UI에서는 문서 관리 시 다음과 같은 불편과 정보 결손이 존재한다.
- `public-front/src/components/AiService.tsx:40-48` (`DocumentInfo`) 인터페이스에 `createdAt: string` 필드가 이미 정의되어 백엔드로부터 업로드 일시를 수신하고 있음에도, `public-front/src/components/AiService.tsx:811-850` (`AiService`)의 문서 목록 테이블(`doc-table`)에는 파일명, 상태, 청크 수, 작업 4개 컬럼만 존재하고 업로드 일시 컬럼이 누락되어 있다. 이로 인해 동일 파일명의 개정본이나 업로드 시점을 화면에서 분별할 수 없다.
- `public-front/src/components/AiService.tsx:78-83` (`AiService`)에서 문서 목록 상태(`documents`)를 단순 배열로 관리할 뿐, 문서 검색이나 필터링 입력 상태가 존재하지 않는다. 등록된 문서가 많아질 경우 특정 문서를 찾기 위해 전체 테이블을 육안으로 훑어야 하는 비효율이 발생한다.
- `public-front/src/utils/date.ts:71-100` (`formatDateTime`)에 프로젝트 표준 일시 포맷팅(`YYYY-MM-DD HH:mm:ss`) 함수가 이미 배송되어 검증되었음에도 지식베이스 문서 관리 영역에 연결되지 않고 방치되어 있다.

## 요구사항
- [ ] `public-front/src/components/AiService.tsx`의 문서 관리 헤더 영역(`kb-header`) 또는 문서 목록 상단에 문서 검색창(placeholder: "문서 검색...")을 추가한다.
- [ ] 검색창에 입력한 키워드를 바탕으로 문서 목록(`documents`)의 파일명(`fileName`)에 대해 대소문자 구분 없이 부분 일치 필터링을 실시간 적용한다.
- [ ] 검색창에 입력된 내용이 있을 때 우측에 검색어 지우기 버튼 또는 `Escape` 키 입력을 통한 초기화 기능을 제공한다.
- [ ] 검색 결과가 없을 경우 테이블 대신 또는 테이블 내에 "검색된 문서가 없습니다." 안내 문구를 렌더링한다.
- [ ] `public-front/src/components/AiService.tsx`의 문서 목록 테이블(`doc-table`) 헤더에 "업로드 일시" 컬럼(`<th>업로드 일시</th>`)을 추가한다.
- [ ] 각 문서 행의 업로드 일시 셀(`<td>`)에 `public-front/src/utils/date.ts`의 `formatDateTime` 유틸리티를 적용하여 `YYYY-MM-DD HH:mm:ss` 형식으로 렌더링한다.
- [ ] `createdAt` 값이 없거나 유효하지 않은 날짜 문자열인 경우 대시(`"-"`)를 안전하게 표시하여 화면 깨짐을 방지한다.
- [ ] 문서 검색 및 업로드 일시 표시에 대한 프론트엔드 단위 테스트를 작성한다.

## 비요구사항 (Out of scope)
- 백엔드 지식베이스 문서 목록 API의 서버 사이드 검색 파라미터 추가나 DB 쿼리 변경은 하지 않는다. (클라이언트 사이드 실시간 필터링으로 처리)
- 문서 청크 본문 내용 전체 텍스트 검색은 지원하지 않는다. (파일명 부분 일치 검색으로 한정)
- 문서 상태별(처리 완료, 진행 중, 실패 등) 다중 드롭다운 필터링 UI는 추가하지 않는다.
- 날짜 범위 지정(기간 검색) 필터는 추가하지 않는다.

## 프론트엔드
`public-front/src/components/AiService.tsx` 및 스타일 파일:
- `documentSearchQuery` 검색어 상태를 추가하고, 렌더링 시 `documents.filter(...)`로 필터링된 목록을 테이블에 전달.
- 문서 관리 섹션(`kb-header`) 상단 또는 목록 상단에 검색 입력창(`<input className="doc-search-input" />`) 추가.
- `doc-table` 헤더 및 바디에 "업로드 일시" 컬럼 추가 (`formatDateTime(doc.createdAt) || '-'`).
- `AiService.css`에 문서 검색 입력창 및 업로드 일시 컬럼 스타일 추가.

## 수용 기준 (Acceptance Criteria)
- Given 지식베이스에 "guide.pdf", "report.txt", "manual.md" 3개의 문서가 등록되어 있을 때
  When 검색창에 "guide"를 입력하면
  Then 파일명에 "guide"가 포함된 "guide.pdf" 1건만 테이블에 표시되어야 한다.
- Given 지식베이스에 "Report_2026.pdf" 문서가 등록되어 있을 때
  When 검색창에 소문자 "report"를 입력하면
  Then 대소문자 구분 없이 "Report_2026.pdf"가 필터링되어 화면에 표시되어야 한다.
- Given 검색창에 "nonexistent"와 같이 일치하는 문서가 없는 키워드를 입력했을 때
  When 필터링이 수행되면
  Then "검색된 문서가 없습니다." 안내 문구가 표시되어야 한다.
- Given 업로드 일시 `createdAt: "2026-08-26T14:30:00.000Z"`를 가진 문서가 있을 때
  When 문서 목록 테이블이 렌더링되면
  Then 업로드 일시 셀에 `formatDateTime` 포맷팅이 적용된 일시 문자열이 정확히 표시되어야 한다.
- Given 경계 케이스: `createdAt` 필드가 누락(undefined)되었거나 빈 문자열(`""`)인 문서가 존재할 때
  When 문서 목록이 렌더링되면
  Then 에러나 화면 깨짐 없이 대시(`"-"`)가 표시되어야 한다.

## 참고
- 문서 목록 렌더링: `public-front/src/components/AiService.tsx:811-850` (`AiService`)
- 문서 정보 인터페이스: `public-front/src/components/AiService.tsx:40-48` (`DocumentInfo`)
- 일시 포맷팅 유틸리티: `public-front/src/utils/date.ts:71-100` (`formatDateTime`)
- 문서 목록 조회 함수: `public-front/src/components/AiService.tsx:356-367` (`fetchDocuments`)

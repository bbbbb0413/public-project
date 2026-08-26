---
id: SPEC-004
title: 문서 업로드 유효성 검증 및 상세 에러 노출 개선
status: done
targets: [server, front]
stages: [backend, frontend, qa]
priority: normal
---

## 배경 / 문제
AI 지식베이스 Q&A 서비스에서 사용자가 문서를 업로드할 때, 파일이 10MB 크기 제한을 초과하거나 허용되지 않은 확장자(예: .png, .docx 등)일 경우 직관적인 에러 피드백을 받지 못한다.
구체적인 문제는 다음과 같다.
- `public-front/src/components/AiService.tsx`의 `handleFileUpload` 함수는 API 에러 발생 시 에러 원인에 상관없이 항상 고정된 "파일 업로드에 실패했습니다." 메시지만 노출한다.
- `public-server/apps/gateway/src/ai/knowledge/knowledge-job.controller.ts`는 Multer의 `fileFilter`에 의해 차단된 파일이나 업로드된 파일이 없을 때 둘 다 단순히 "허용되지 않는 파일이거나 파일이 누락되었습니다." 라는 메시지로 통합하여 반환하며, 10MB 크기 초과 에러(MulterError) 발생 시에는 게이트웨이가 기본 서버 오류로 응답할 수 있어 원인 파악이 어렵다.
- 브라우저 클라이언트 사이드에서 API 요청을 보내기 전에 파일 유효성 검사(크기, 확장자)가 수행되지 않아, 불필요한 네트워크 트래픽이 발생하고 백엔드 서버에 부하를 줄 수 있다.
따라서 클라이언트와 서버 양방향에서 유효성 검증을 고도화하고 상세한 에러 피드백을 제공하도록 개선한다.

## 요구사항
- [ ] 프론트엔드에서 파일 업로드 시, 클라이언트 사이드에서 파일 크기(최대 10MB) 및 지원 파일 형식(.txt, .pdf, .md)을 1차로 검증한다
- [ ] 클라이언트 검증 실패 시, 각각 "파일 크기는 최대 10MB까지 허용됩니다.", "지원하지 않는 파일 형식입니다. (TXT, PDF, MD 파일만 지원)" 에러 메시지를 화면에 노출하고 API 호출을 차단한다
- [ ] 프론트엔드에서 업로드 API 호출 실패 시, 백엔드로부터 전달받은 구체적인 에러 메시지(message)를 에러 배너에 표시한다. 백엔드 메시지가 없거나 알 수 없는 에러인 경우 기존의 "파일 업로드에 실패했습니다."를 표시한다
- [ ] 백엔드 게이트웨이에서 업로드된 파일이 없을 경우 "파일이 누락되었습니다." 예외를 던진다
- [ ] 백엔드 게이트웨이에서 허용되지 않은 mime-type일 경우 "지원하지 않는 파일 형식입니다. (TXT, PDF, MD 파일만 지원)" 예외를 던진다
- [ ] 백엔드 게이트웨이에서 파일 크기가 10MB를 초과한 경우(Multer LIMIT_FILE_SIZE 에러) "파일 크기가 10MB 제한을 초과했습니다." 예외를 던진다
- [ ] 위 검증 규칙 및 예외 처리 로직에 대한 테스트 코드를 작성한다

## 비요구사항 (Out of scope)
- 백엔드 내부 인제스트 처리 단계(Kafka 컨슈머 및 RAG 파이프라인 내부)에서의 문서 파싱 실패 관련 에러 전파 개선. 이번 개선은 업로드 API 요청을 검증하고 즉각 반환하는 게이트웨이 레이어와 프론트엔드에 한정한다.
- 파일 다중 업로드 기능 지원.
- 파일 업로드 진행률(Progress Bar) 표시 기능 추가.

## 백엔드
`public-server` 의 `apps/gateway` 프로젝트를 수정한다.
- `apps/gateway/src/ai/knowledge/knowledge-job.controller.ts` 의 파일 업로드 엔드포인트 수정:
  - NestJS의 `FileInterceptor`에서 발생하는 MulterError(`LIMIT_FILE_SIZE`) 및 파일 누락/형식 에러를 처리하기 위해, 커스텀 Exception Filter나 인터셉터 레벨에서 예외를 가공하여 응답 본문(`{ statusCode: 400, message: "..." }`)에 상세 사유를 담아 반환하도록 구현한다.
  - 구체적인 에러 상황별 응답 메시지 정의:
    - 파일 미포함: `400 Bad Request`, `message: "업로드할 파일이 누락되었습니다."`
    - 허용되지 않은 형식: `400 Bad Request`, `message: "지원하지 않는 파일 형식입니다. (TXT, PDF, MD 파일만 지원)"`
    - 10MB 초과: `413 Payload Too Large` 또는 `400 Bad Request`, `message: "파일 크기가 10MB 제한을 초과했습니다."`
  - 테스트 코드는 `apps/gateway/test/` 또는 `apps/gateway/src/ai/knowledge/` 하위에 해당 컨트롤러 검증을 위한 unit/e2e 테스트로 추가한다.

## 프론트엔드
`public-front` 프로젝트를 수정한다.
- `src/components/AiService.tsx` 의 `handleFileUpload` 함수 수정:
  - 파일 업로드 시작 시 클라이언트 사이드 검증 로직 추가.
  - 업로드 API 호출 실패(`catch` 블록) 시:
    ```typescript
    const serverMessage = error.response?.data?.message;
    setErrorMsg(typeof serverMessage === 'string' ? serverMessage : '파일 업로드에 실패했습니다.');
    ```
    형태로 서버의 구체적인 에러 메시지를 수용하도록 개선한다.
  - `src/components/AiService.test.tsx` 에 클라이언트 사이드 유효성 검사 실패 시 각각의 에러 메시지가 렌더링되고 API가 호출되지 않는지 검증하는 테스트 케이스를 추가한다.

## 수용 기준 (Acceptance Criteria)
- Given 11MB 크기의 `test.pdf` 파일이 있을 때 When 사용자가 프론트엔드에서 해당 파일을 업로드하면 Then API 요청을 보내지 않고 즉시 화면에 "파일 크기는 최대 10MB까지 허용됩니다." 에러 메시지를 노출한다
- Given `.jpg` 확장자를 가진 `avatar.jpg` 파일이 있을 때 When 사용자가 프론트엔드에서 해당 파일을 업로드하면 Then API 요청을 보내지 않고 즉시 화면에 "지원하지 않는 파일 형식입니다. (TXT, PDF, MD 파일만 지원)" 에러 메시지를 노출한다
- Given 프론트엔드를 거치지 않고 직접 API(`/ai/knowledge/jobs`)로 11MB 크기의 파일을 전송했을 때 When 백엔드가 요청을 받으면 Then `413 Payload Too Large` 또는 `400 Bad Request` 에러와 함께 "파일 크기가 10MB 제한을 초과했습니다." 응답 메시지를 반환한다
- Given 프론트엔드를 거치지 않고 직접 API(`/ai/knowledge/jobs`)로 `.png` 파일을 전송했을 때 When 백엔드가 요청을 받으면 Then `400 Bad Request` 에러와 함께 "지원하지 않는 파일 형식입니다. (TXT, PDF, MD 파일만 지원)" 응답 메시지를 반환한다
- Given 정상적인 5MB 크기의 `manual.txt` 파일을 전송했을 때 When 파일 업로드에 성공하면 Then 정상적으로 202 응답을 반환하고 화면에 에러가 노출되지 않는다

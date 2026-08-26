# 유저별 프롬프트 설정 기능 계획 문서

## 1. 현재 상태 (As-Is)

### 문제점
- `RAG_QA_DEFAULT_PROMPT` 상수가 `get-active-prompt.use-case.ts`에 하드코딩됨
- `PromptTemplate` 도메인 모델에 `userId` 필드 없음
- `GetActivePromptUseCase.execute(name)` — userId 파라미터 없음
- `AskUseCase.buildRagMessages()` — userId 없이 글로벌 프롬프트만 조회
- `QaController` — 인증 사용자 정보를 파이프라인에 전달하지 않음
- 모든 사용자가 동일한 RAG 시스템 프롬프트를 사용함

### 현재 파일 목록 (변경 대상)

#### 백엔드 (public-server)
| 파일 | 현재 상태 |
|------|----------|
| `prompt/domain/model/prompt-template.ts` | userId 필드 없음 |
| `prompt/domain/repository/prompt-template.repository.ts` | findActiveForUser 메서드 없음 |
| `prompt/infrastructure/persistence/prompt-template.repository-impl.ts` | userId 쿼리 없음 |
| `prompt/application/get-active-prompt.use-case.ts` | userId 파라미터 없음, 하드코딩 폴백 |
| `prompt/application/create-prompt.use-case.ts` | userId 없음 |
| `prompt/application/command/create-prompt.command.ts` | userId 없음 |
| `prompt/presentation/prompt.controller.ts` | userId 라우트 없음 |
| `prompt/presentation/dto/create-prompt-in.dto.ts` | userId 없음 |
| `qa/application/ask.command.ts` | userId 없음 |
| `qa/application/ask.use-case.ts` | userId 없이 getActivePrompt 호출 |
| `qa/presentation/dto/ask-in.dto.ts` | userId 없음 |
| `qa/presentation/qa.controller.ts` | userId 추출 없음 |

#### 프론트엔드 (public-front)
| 파일 | 현재 상태 |
|------|----------|
| `src/api/aiAdmin.ts` | 유저별 프롬프트 API 없음 |

---

## 2. 목표 상태 (To-Be)

### 폴백 체인
```
유저별 활성 프롬프트 (userId + name + isActive=true)
    ↓ 없으면
글로벌 활성 프롬프트 (name + isActive=true, userId 없음)
    ↓ 없으면
RAG_QA_DEFAULT_PROMPT (하드코딩 상수)
```

### 사용자 경험
1. 관리자/사용자가 자신만의 RAG 시스템 프롬프트를 등록·활성화할 수 있다
2. 프롬프트가 없는 사용자는 글로벌 설정 또는 기본값을 사용한다
3. 질문 API 호출 시 userId를 함께 전달하면 유저 전용 프롬프트가 적용된다

---

## 3. 변경 범위

### 3-1. Domain Layer

**`prompt-template.ts`**
- `PromptTemplateProps`에 `userId?: string` 추가
- `create()` factory에 userId 지원
- `restore()` factory에 userId 지원

### 3-2. Repository Interface

**`prompt-template.repository.ts`**
- `findActiveForUser(name: string, userId: string): Promise<PromptTemplate | null>` 메서드 추가

### 3-3. Infrastructure Layer

**`prompt-template.repository-impl.ts`**
- `PromptTemplateRecord`에 `userId?: string` 추가
- `findActiveForUser(name, userId)` 구현: `{ name, userId, isActive: true }` 쿼리
- MongoDB 복합 인덱스: `{ name: 1, userId: 1, isActive: 1 }`

### 3-4. Application Layer

**`get-active-prompt.use-case.ts`**
- `execute(name: string, userId?: string)` 시그니처 변경
- userId 있으면 `findActiveForUser(name, userId)` 먼저 시도
- 없으면 기존 `findActive(name)` → `RAG_QA_DEFAULT_PROMPT` 폴백

**`create-prompt.command.ts`**
- `userId?: string` 필드 추가

**`create-prompt.use-case.ts`**
- command.userId를 `PromptTemplate.create()` 시 전달

### 3-5. Presentation Layer (백엔드)

**`create-prompt-in.dto.ts`**
- `userId?: string` 옵션 추가 (`@IsString() @IsOptional()`)

**`prompt.controller.ts`**
- `POST /prompts` — userId 받아서 command에 전달
- `GET /prompts/:name/active?userId=xxx` — 유저별 활성 프롬프트 조회 엔드포인트 추가

**`ask-in.dto.ts`**
- `userId?: string` 옵션 추가

**`ask.command.ts`**
- `userId?: string` 필드 추가

**`qa.controller.ts`**
- `AskCommand` 생성 시 `dto.userId` 전달

**`ask.use-case.ts`**
- `buildRagMessages()` 시 `command.userId`를 `getActivePrompt.execute(RAG_PROMPT_NAME, command.userId)`로 전달

### 3-6. Frontend

**`src/api/aiAdmin.ts`**
- `getUserPrompts(userId)` 추가
- `createUserPrompt(userId, data)` 추가
- `activateUserPrompt(userId, name, version)` 추가

---

## 4. MongoDB 인덱스 전략

현재 `prompt_templates` 컬렉션에 `{ name: 1, isActive: 1 }` 인덱스 사용.

추가할 인덱스:
```json
{ "name": 1, "userId": 1, "isActive": 1 }
```

- `userId`가 null/undefined인 레코드 → 글로벌 프롬프트
- `userId`가 있는 레코드 → 유저별 프롬프트

---

## 5. 구현 순서

1. `PromptTemplate` 도메인 모델 — userId 추가
2. `IPromptTemplateRepository` 인터페이스 — findActiveForUser 추가
3. `PromptTemplateRepositoryImpl` — findActiveForUser 구현, 인덱스 추가
4. `GetActivePromptUseCase` — userId 파라미터 + 폴백 체인
5. `CreatePromptCommand` + `CreatePromptUseCase` — userId 지원
6. `CreatePromptInDto` + `PromptController` — userId 라우트
7. `AskCommand` + `AskInDto` + `QaController` + `AskUseCase` — userId 파이프라인
8. 프론트엔드 API 클라이언트

---

## 6. 하위 호환성

- 기존 글로벌 프롬프트(userId 없음)는 그대로 동작
- userId 없이 질문하면 기존과 동일하게 글로벌 → 기본값 폴백
- 기존 API 클라이언트 코드 변경 불필요 (userId는 옵션 필드)

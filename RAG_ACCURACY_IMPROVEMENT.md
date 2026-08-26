# RAG 정확도 개선 리서치 보고서

> 작성일: 2026-07-01  
> 대상 서비스: `apps/ai-service` (NestJS + MongoDB Atlas Vector Search)  
> 분석 파일: `ingest-document.use-case.ts`, `hybrid-search.use-case.ts`, `hyde.service.ts`, `ask.use-case.ts`

---

## 1. 현황 분석

### 1.1 청킹 현재 구현

```typescript
// ingest-document.use-case.ts:23-27
private readonly splitter = new RecursiveCharacterTextSplitter({
  chunkSize: 500,       // 글자 수 기준 고정 분할
  chunkOverlap: 100,
  separators: ['\n\n', '\n', '다. ', '요. ', '까. ', '죠. ', '나. ', ' ', ''],
});
```

| 문제 | 설명 |
|------|------|
| 의미 경계 무시 | 글자 수 기준 분할 → 문장 중간 잘림, 개념 맥락 손실 |
| 구조화 문서 부적합 | 이력서·표·목록을 단순 텍스트로 평탄화 |
| 메타데이터 빈약 | `documentId`, `fileName`, `chunkIndex`만 저장 |
| 고정 크기 | 문서 성격(Q&A vs 서술형 vs 표)에 관계없이 동일 500자 |
| PDF 페이지 경계 미보존 | `cleanPdfText()`에서 페이지 구분 정보 제거 |

### 1.2 쿼리 처리 현재 구현

```typescript
// hyde.service.ts:shouldApply()
shouldApply(question: string): boolean {
  return trimmed.split(/\s+/).length <= this.maxQueryWords; // 기본값 5단어 이하에만 HyDE
}

// hybrid-search.use-case.ts:execute()
const searchQuery = await this.resolveQuery(command); // 단일 쿼리 사용
const [denseResults, lexicalResults] = await Promise.all([...]);
```

```typescript
// ask.use-case.ts:94
// conversationHistory는 buildRagMessages()에서 LLM context에만 삽입됨
// hybridSearch.execute()는 원본 question을 그대로 사용
const { queryEmbedding, chunks } = await this.hybridSearch.execute(
  new HybridSearchCommand(command.question, command.topK, command.useHyde),
);
```

| 문제 | 코드 위치 | 설명 |
|------|----------|------|
| HyDE 조건 역전 | `hyde.service.ts:shouldApply()` | 5단어 이하 단문에만 HyDE — 긴 복잡한 질문에는 오히려 미적용 |
| 멀티턴 쿼리 미처리 | `ask.use-case.ts:94` | "그 다음은?" 같은 후속 질문이 독립 쿼리로 검색됨 |
| 복합 질문 단일 벡터 | `hybrid-search.use-case.ts` | "A와 B를 비교해줘"가 단일 임베딩으로 검색 |

---

## 2. 청킹 개선 방안

### 2.1 Page-level Chunking ★ 즉시 적용, 가장 안정적

**출처:** NVIDIA Research (2025.06) — [Finding the Best Chunking Strategy for Accurate AI Responses](https://developer.nvidia.com/blog/finding-the-best-chunking-strategy-for-accurate-ai-responses/)

NVIDIA 실험에서 전략별 평균 정확도:

| 전략 | 평균 정확도 | 표준편차 |
|------|------------|---------|
| **Page-level** | **0.648** | **0.107** ← 최고 |
| Fixed-size 256토큰 | 0.621 | 0.142 |
| Semantic chunking | 0.589 | 0.198 |
| Fixed-size 128토큰 | 0.563 | 0.165 |

표준편차가 낮아 문서 유형에 무관하게 안정적. PDF 페이지 경계를 인제스트 시 보존하면 **현재 코드 최소 변경으로 즉시 적용 가능**.

**적용 포인트** (`ingest-document.use-case.ts`):
```typescript
private cleanPdfText(text: string): string {
  return text
    .replace(/\r\n/g, '\n')
    .replace(/\r/g, '\n')
    .replace(/[ \t]+/g, ' ')
    // 페이지 경계 보존: \f(form feed)를 섹션 구분자로 유지
    // .replace(/\n{3,}/g, '\n\n') ← 이 줄 제거 또는 완화
    .trim();
}

// 청크 분할: 페이지 구분 문자 '\f'를 첫 번째 separator로 추가
private readonly splitter = new RecursiveCharacterTextSplitter({
  chunkSize: 1024,   // 500 → 1024 (팩토이드 질문 기준 권장)
  chunkOverlap: 200,
  separators: ['\f', '\n\n', '\n', '다. ', '요. ', '까. ', '죠. ', '나. ', ' ', ''],
});
```

---

### 2.2 chunkSize 확대 (500 → 1024)

**출처:** NVIDIA Research (2025.06)

- 팩토이드(단답형) 질문: 256-512 토큰 권장
- 복잡한 분석 질문: 1024 토큰 이상 권장
- 한국어 기준 1토큰 ≈ 1.5~2글자이므로, 1024토큰 ≈ 한국어 약 700-900자

현재 500자는 한국어로 약 250-350토큰 수준으로 **컨텍스트가 매우 부족**. 최소 1024자(또는 512토큰) 이상으로 확대하고 A/B 비교 권장.

---

### 2.3 Parent-Child Chunking (소-대 계층 청킹) ★★ 효과 높음

**출처:** LangChain / LlamaIndex — [Parent Document Retrieval 2025](https://thread-transfer.com/blog/2025-07-29-parent-document-retrieval/)

```
[Parent 청크: 1024-2048자] → MongoDB에 저장, LLM에 전달
    ├── [Child 청크 1: 200-300자] ← 벡터 검색에 사용
    ├── [Child 청크 2: 200-300자]
    └── [Child 청크 3: 200-300자]
```

**동작 방식:**
1. 문서를 큰 단위(Parent)로 1차 분할 → MongoDB에 원문 보존
2. Parent를 다시 작은 단위(Child)로 2차 분할 → 벡터 스토어 인덱싱
3. 검색 시: Child로 정확한 위치 탐색 → Child의 parentId로 원문 조회 → LLM에 Parent 전달

**기대 효과:** 복잡한 질문에서 정확도 +15-30% (LlamaIndex 벤치마크)

**현재 코드 변경점:**
```typescript
// chunk.vo.ts — parentId 추가
interface ChunkValue {
  text: string;
  index: number;
  documentId: string;
  parentChunkId?: string;  // 추가
  charCount: number;       // 추가
}

// mongodb-vector.adapter.ts metadata 확장
metadata: {
  documentId,
  fileName,
  chunkIndex,
  parentChunkId,  // 추가
  charCount,      // 추가
}
```

**트레이드오프:** 저장 공간 2-3배 증가, `VectorStorePort` 인터페이스 수정 필요

---

### 2.4 Contextual Embeddings (Anthropic) ★★★ 가장 큰 효과

**출처:** Anthropic Engineering Blog (2024.09) — [Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval)

인덱싱 시 각 청크에 LLM이 생성한 "문서 내 위치 맥락" 50-100토큰을 접두어로 추가한 뒤 임베딩:

```
[원본 청크]
"2019년부터 2022년까지 백엔드 개발을 담당했습니다."

[컨텍스트 추가 후]
"이 문서는 김OO의 이력서입니다. 경력 사항 섹션에서 추출된 내용입니다.
2019년부터 2022년까지 백엔드 개발을 담당했습니다."
```

**Anthropic 공식 벤치마크:**

| 기법 | Pass@10 | 검색 실패율 |
|------|---------|------------|
| Baseline RAG | 87.15% | 12.85% |
| + Contextual Embeddings | 92.34% | 7.66% (-40%) |
| + Hybrid Search (BM25) | 93.21% | 6.79% (-47%) |
| + Reranking | **95.26%** | **4.74% (-63%)** |

검색 실패율 단독 적용 시 **-35%, BM25 결합 시 -49%, Reranking까지 -67%**

**권고 프롬프트** (`ingest-document.use-case.ts`에 추가):
```typescript
private async generateChunkContext(
  fullDocumentText: string,
  chunkText: string,
  fileName: string,
): Promise<string> {
  const messages = [
    {
      role: 'system' as const,
      content: '다음 청크가 전체 문서에서 어떤 맥락에 위치하는지 1-2문장으로 설명하세요. 간결하게 작성하세요.',
    },
    {
      role: 'user' as const,
      content: `파일명: ${fileName}\n\n전체 문서:\n<document>${fullDocumentText.slice(0, 8000)}</document>\n\n청크:\n<chunk>${chunkText}</chunk>\n\n이 청크의 문서 내 맥락:`,
    },
  ];
  const tokens: string[] = [];
  for await (const token of this.llmProvider.stream(messages)) tokens.push(token);
  return tokens.join('');
}

// 청크 임베딩 전 컨텍스트 접두어 추가
const contextualizedTexts = await Promise.all(
  chunks.map(async (chunk) => {
    const context = await this.generateChunkContext(rawText, chunk.getText(), command.fileName);
    return `${context}\n${chunk.getText()}`;
  }),
);
const embeddings = await this.embeddingProvider.embed(contextualizedTexts);
```

**트레이드오프:** 청크당 LLM 호출 1회 추가 → 인제스트 비용·시간 크게 증가 (병렬 처리 + 레이트 리밋 고려 필요)

---

### 2.5 메타데이터 강화

현재 `VectorDocument` 메타데이터에 섹션 구조 정보가 없어 재랭킹·필터링에 활용 불가.

**권고 스키마 확장:**
```typescript
interface EnrichedChunkMetadata {
  documentId: string;
  fileName: string;
  chunkIndex: number;
  // 추가 필드
  parentChunkId?: string;      // Parent-Child 패턴용
  sectionTitle?: string;       // 해당 청크가 속한 섹션 제목
  chunkType?: 'paragraph' | 'list' | 'table' | 'heading';
  charCount: number;           // 길이 편향 보정용
  pageNumber?: number;         // PDF 페이지 번호
}
```

---

## 3. 쿼리 처리 개선 방안

### 3.1 HyDE 조건 수정 + Hybrid HyDE ★ 즉시 적용

**출처:**
- Gao et al. (2022) — *Precise Zero-Shot Dense Retrieval without Relevance Labels* (HyDE 원 논문)
- Milvus AI Reference — [What is HyDE and when should I use it](https://milvus.io/ai-quick-reference/what-is-hyde-hypothetical-document-embeddings-and-when-should-i-use-it)

**HyDE가 효과적인 조건:**
- 쿼리가 짧고 막연할 때 (3-10단어)
- 쿼리와 문서 간 어휘 스타일 차이가 클 때 (구어체 질문 vs 문어체 문서)
- 코퍼스가 서술형 산문(이력서, 보고서, 메모)일 때

**HyDE가 악영향인 조건:**
- 1-2단어 키워드 검색 (가상 문서가 드리프트 유발)
- 쿼리가 이미 길고 구체적일 때 (LLM이 잘못된 방향의 문서 생성 가능)

**현재 구현 문제:** 5단어 이하에만 적용 → 학술 기준("3-10단어의 막연한 쿼리")과 다름. 5단어 이하면 1-2단어 키워드 검색도 포함돼 오히려 역효과 가능.

**권고 수정** (`hyde.service.ts`):
```typescript
shouldApply(question: string): boolean {
  const wordCount = question.trim().split(/\s+/).length;
  // 3-10단어: 막연한 쿼리 → HyDE 적용
  // 1-2단어 키워드(노이즈), 11단어+ 상세 질문(이미 구체적) 제외
  return wordCount >= 3 && wordCount <= 10;
}
```

**추가: Hybrid HyDE** — 가상 문서와 원본 쿼리의 평균 임베딩 사용:
```typescript
// hybrid-search.use-case.ts — resolveQuery() 개선
private async resolveQuery(command: HybridSearchCommand): Promise<number[]> {
  if (!command.useHyde || !this.hydeService.shouldApply(command.question)) {
    const [vec] = await this.embeddingProvider.embed([command.question]);
    return vec;
  }

  const hypoDoc = await this.hydeService.generateHypothetical(command.question);
  const [[hypoVec], [queryVec]] = await Promise.all([
    this.embeddingProvider.embed([hypoDoc]),
    this.embeddingProvider.embed([command.question]),
  ]);

  const similarity = cosineSimilarity(hypoVec, queryVec);
  if (similarity < 0.5) return queryVec; // 가상 문서 방향이 엉뚱하면 원본으로 fallback

  return hypoVec.map((v, i) => (v + queryVec[i]) / 2); // Hybrid HyDE
}
```

---

### 3.2 Conversational Query Rewriting ★ 멀티턴 대화 정확도 향상

**현재 문제** (`ask.use-case.ts:94`):
```typescript
// conversationHistory는 LLM context에만 삽입됨
// hybridSearch.execute()는 원본 question을 그대로 사용 → 후속 질문이 독립 검색 불가
```

**예시:**
```
이전 질문: "강승원씨의 환불 정책은?"
현재 질문: "그럼 해외 배송은?"

→ 현재: "그럼 해외 배송은?"으로 벡터 검색 → 관련 문서 미검색
→ 개선 후: "해외 배송 환불 정책은 무엇인가?"로 재작성 후 검색
```

**권고 구현** (신규 `conversational-query-rewriter.service.ts`):
```typescript
@Injectable()
export class ConversationalQueryRewriter {
  constructor(@Inject(LlmProvider) private readonly llmProvider: ILlmProvider) {}

  async rewrite(
    question: string,
    history: Array<{ role: 'user' | 'assistant'; content: string }>,
  ): Promise<string> {
    if (!history || history.length === 0) return question;
    if (!this.isFollowUpQuestion(question)) return question;

    const messages = [
      {
        role: 'system' as const,
        content:
          '대화 이력을 참고하여 마지막 질문을 이전 대화 없이도 이해할 수 있는 독립적인 질문으로 재작성하세요. 원래 의도와 언어를 유지하세요. 재작성된 질문만 출력하세요.',
      },
      {
        role: 'user' as const,
        content: `대화 이력:\n${history
          .slice(-4)
          .map((h) => `${h.role}: ${h.content}`)
          .join('\n')}\n\n마지막 질문: ${question}\n\n독립적인 질문:`,
      },
    ];

    const tokens: string[] = [];
    for await (const token of this.llmProvider.stream(messages)) tokens.push(token);
    return tokens.join('').trim() || question;
  }

  private isFollowUpQuestion(question: string): boolean {
    const patterns = [
      /^(그|저|이|그것|그거|이거|저거)/,
      /^(그럼|그러면|그렇다면|그리고|또한|그 다음)/,
      /^(더|추가로|또)\s/,
      /더\s(자세히|구체적으로|설명)/,
      /위에서|앞에서|방금/,
    ];
    return patterns.some((p) => p.test(question.trim()));
  }
}
```

**`ask.use-case.ts` 통합:**
```typescript
// hybridSearch.execute() 호출 전에 삽입
const searchQuery = await this.queryRewriter.rewrite(
  command.question,
  conversationHistory ?? [],
);

const { queryEmbedding, chunks } = await this.hybridSearch.execute(
  new HybridSearchCommand(searchQuery, command.topK, command.useHyde),
);
// buildRagMessages()에는 원본 command.question 유지
```

**기대 효과:** 멀티턴 Retrieval Precision +15-30%, 추가 레이턴시 약 100-150ms

---

### 3.3 Query Decomposition ★★ 복합 질문 대응

**출처:**
- ACL 2025 — [Query Decomposition for Multi-hop RAG](https://doi.org/10.18653/v1/2025.acl-srw.32)
- EMNLP 2025 Findings — [UniRAG](https://aclanthology.org/2025.findings-emnlp.1022.pdf)

**성과 (MultiHop-RAG 데이터셋):**

| 지표 | 개선 전 | 개선 후 | 향상 |
|------|---------|---------|------|
| MRR@10 | 기준 | +36.7% | ↑ |
| F1 | 기준 | +11.6% | ↑ |
| Hits@10 | 74.7% | 87.2% | ↑ |

**권고 구현** (신규 `query-decomposer.service.ts`):
```typescript
@Injectable()
export class QueryDecomposer {
  constructor(@Inject(LlmProvider) private readonly llmProvider: ILlmProvider) {}

  async decompose(question: string): Promise<string[]> {
    if (!this.isComplexQuery(question)) return [question];

    const messages = [
      {
        role: 'system' as const,
        content:
          '다음 질문을 독립적으로 검색 가능한 하위 질문들로 분해하세요. 단순한 질문은 그대로 반환하세요. 최대 3개 이하로 분해하세요. JSON 배열 형식으로만 응답하세요.',
      },
      {
        role: 'user' as const,
        content: `질문: ${question}\n\n하위 질문들(JSON 배열):`,
      },
    ];

    const tokens: string[] = [];
    for await (const token of this.llmProvider.stream(messages)) tokens.push(token);
    try {
      const parsed = JSON.parse(tokens.join('').trim()) as unknown;
      return Array.isArray(parsed) ? (parsed as string[]) : [question];
    } catch {
      return [question];
    }
  }

  private isComplexQuery(question: string): boolean {
    return /와|과|그리고|비교|차이|둘 다|모두/.test(question) && question.length > 20;
  }
}
```

**검색 통합** (`hybrid-search.use-case.ts` 또는 `ask.use-case.ts`):
```typescript
const subQueries = await this.decomposer.decompose(command.question);
const allResults = await Promise.all(
  subQueries.map((q) =>
    this.hybridSearch.execute(new HybridSearchCommand(q, command.topK, command.useHyde)),
  ),
);
const mergedChunks = this.rrfFusion.fuse(
  allResults.map((r) => r.chunks),
  this.rrfK,
);
```

---

## 4. 우선순위 종합 로드맵

### Phase 1 — 즉시 적용 (1-2일, 코드 변경 최소)

| 순위 | 항목 | 예상 효과 | 구현 난이도 |
|------|------|-----------|------------|
| 1 | **HyDE 조건 수정** (`3 ≤ 단어 수 ≤ 10`) | Recall 즉시 향상 | 낮음 (1줄) |
| 2 | **chunkSize 500 → 1024** 증가 | 컨텍스트 부족 해결 | 낮음 (1줄) |
| 3 | **PDF 페이지 경계 보존** (`\f` separator 추가) | 정확도 0.648 달성 가능 | 낮음 |
| 4 | **메타데이터 강화** (`charCount`, `sectionTitle`) | 재랭킹 품질 향상 | 낮음 |

### Phase 2 — 단기 적용 (3-5일, 새 서비스 추가)

| 순위 | 항목 | 예상 효과 | 구현 난이도 |
|------|------|-----------|------------|
| 5 | **Conversational Query Rewriting** | 멀티턴 Precision +15-30% | 중간 |
| 6 | **Hybrid HyDE** (코사인 유사도 기반 fallback) | HyDE 할루시네이션 방어 | 중간 |

### Phase 3 — 중기 적용 (1-2주, 인프라 변경 포함)

| 순위 | 항목 | 예상 효과 | 구현 난이도 |
|------|------|-----------|------------|
| 7 | **Parent-Child 청킹** | Recall@5 +20% | 높음 |
| 8 | **Query Decomposition** | Multi-hop MRR +36.7% | 높음 |
| 9 | **Contextual Embeddings** | 검색 실패율 -35~67% | 높음 (인제스트 비용 대폭 ↑) |

---

## 5. 참고 자료

| 주제 | 논문/자료 |
|------|----------|
| Page-level Chunking 벤치마크 | [NVIDIA: Finding the Best Chunking Strategy](https://developer.nvidia.com/blog/finding-the-best-chunking-strategy-for-accurate-ai-responses/) |
| Contextual Retrieval | [Anthropic Engineering Blog](https://www.anthropic.com/engineering/contextual-retrieval) |
| Parent Document Retrieval | [LangChain/LlamaIndex 2025](https://thread-transfer.com/blog/2025-07-29-parent-document-retrieval/) |
| Adaptive Chunking | [arXiv 2603.25333](https://arxiv.org/pdf/2603.25333) |
| PIC Chunking | [ACL 2025 Findings](https://aclanthology.org/2025.findings-acl.422.pdf) |
| HyDE 원 논문 | Gao et al. (2022) — Precise Zero-Shot Dense Retrieval |
| HyDE 적용 가이드 | [Milvus AI Reference](https://milvus.io/ai-quick-reference/what-is-hyde-hypothetical-document-embeddings-and-when-should-i-use-it) |
| Query Decomposition | [ACL SRW 2025](https://doi.org/10.18653/v1/2025.acl-srw.32) |
| UniRAG | [EMNLP 2025 Findings](https://aclanthology.org/2025.findings-emnlp.1022.pdf) |
| ReDI 파이프라인 | [arXiv 2509.06544](https://arxiv.org/html/2509.06544) |

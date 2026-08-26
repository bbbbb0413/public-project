# RAG 정확도 개선 리서치 보고서

> 작성일: 2026-07-01  
> 대상 시스템: `apps/ai-service` (NestJS + MongoDB Atlas Vector Search)

---

## 현재 시스템 문제 진단

### 청킹 (Chunking)

```typescript
// 현재 구현: ingest-document.use-case.ts
private readonly splitter = new RecursiveCharacterTextSplitter({
  chunkSize: 500,       // 문자 수 기준 — 의미 경계 무시
  chunkOverlap: 100,
  separators: ['\n\n', '\n', '다. ', '요. ', ...],
});
```

**문제점:**
- 500자 고정 분할은 문단 중간에서 의미를 끊음
- metadata에 `{documentId, fileName, chunkIndex}`만 있고 섹션 구조 없음
- 단일 크기 청크라 복잡한 질문에 컨텍스트가 부족

### 쿼리 이해 (Query Processing)

```typescript
// 현재 구현: hyde.service.ts
shouldApply(question: string): boolean {
  return trimmed.split(/\s+/).length <= this.maxQueryWords; // 기본값 5단어
}
```

**문제점:**
- 단어 수(5개) 기준으로만 HyDE 적용 — 복잡하고 긴 질문에는 오히려 미적용
- Multi-hop 질문(여러 문서에 걸친 정보 필요)에 단일 쿼리만 사용
- 대화 이력이 있어도 쿼리 재작성 없음

---

## Part 1: 청킹 방식 개선

### 연구 결과 요약

#### 1. Page-level Chunking (NVIDIA, 2025.06) ★ 가장 안정적

**출처:** https://developer.nvidia.com/blog/finding-the-best-chunking-strategy-for-accurate-ai-responses/

| 전략 | 평균 정확도 | 표준편차 |
|------|------------|---------|
| **Page-level** | **0.648** | **0.107** ← 최고 |
| Fixed-size 256토큰 | 0.621 | 0.142 |
| Semantic chunking | 0.589 | 0.198 |
| Fixed-size 128토큰 | 0.563 | 0.165 |

- 팩토이드 질문(단답형) → 256-512 토큰 권장
- 복잡한 분석 질문 → 1024 토큰 이상 권장
- 표준편차가 낮아 문서 유형에 관계없이 안정적

**현재 시스템 적용:** PDF 페이지 경계를 `extractText()`에서 보존하면 즉시 적용 가능

---

#### 2. Parent Document Retrieval / Small-to-Big (LangChain/LlamaIndex) ★★ 효과 높음

**출처:** https://thread-transfer.com/blog/2025-07-29-parent-document-retrieval/

```
검색 단계: 소형 청크(100-200 토큰) → 높은 정밀도
LLM 전달: 부모 청크(500-2000 토큰) → 충분한 컨텍스트
```

- 복잡한 질문에서 **15-30% 정확도 향상** 보고
- 계층 구조 권장: `chunkSizes = [2048, 512, 128]`
- 현재 500자 단일 청크 대비 검색 정밀도와 컨텍스트 풍부함을 동시에 달성

**현재 시스템 적용:** VectorDocument에 `parentChunkId` 필드 추가, MongoDB에 부모 청크 저장

---

#### 3. Contextual Embeddings / Contextual BM25 (Anthropic, 2024.09) ★★★ 가장 큰 효과

**출처:** https://www.anthropic.com/engineering/contextual-retrieval

각 청크를 임베딩하기 전에 Claude로 문서 전체 맥락을 50-100 토큰의 컨텍스트로 요약해 청크 앞에 붙이는 방식:

```
[원본 청크]
"회사의 Q3 수익은 $3B였다."

[컨텍스트 추가 후]
"이 문서는 ACME Corp의 2023 연간 보고서다. 해당 청크는 Q3 재무 실적을 다룬다."
"회사의 Q3 수익은 $3B였다."
```

| 기법 | Pass@10 | 검색 실패율 |
|------|---------|------------|
| Baseline RAG | 87.15% | 12.85% |
| + Contextual Embeddings | 92.34% | 7.66% |
| + Hybrid Search (BM25) | 93.21% | 6.79% |
| + Reranking | **95.26%** | **4.74%** |

- Contextual Embeddings만으로 검색 실패율 **35% 감소** (5.7% → 3.7%)
- Contextual BM25까지 적용 시 **49% 감소** (5.7% → 2.9%)
- Reranking까지 모두 적용 시 **67% 감소** (5.7% → 1.9%)

**현재 시스템 적용:** `ingest-document.use-case.ts`에서 청크 생성 후 Claude API 호출로 컨텍스트 생성 → `chunk.getText()` 앞에 붙여 임베딩

---

#### 4. Adaptive Chunking (arXiv, 2026.03)

**출처:** https://arxiv.org/pdf/2603.25333

문서마다 5개 지표(RC, ICC, DCC, BI, SC)를 측정해 최적 청킹 전략을 자동 선택:

- 정확도: 62-64% → **72%** (+30% 성공 질문 수)
- 구현 복잡도 높음 — 단기 적용보다는 장기 목표로 적합

---

#### 5. PIC: Pseudo-Instruction Chunking (ACL, 2025)

**출처:** https://aclanthology.org/2025.findings-acl.422.pdf

문서 전체 요약을 "pseudo-instruction"으로 사용해 문장-요약 유사도 기반 청킹:

- 추가 학습 없이 Hits@k 및 Exact Match 유의미하게 향상
- Contextual Embeddings와 방향이 유사 — 같이 적용 가능

---

### 청킹 개선 우선순위 로드맵

```
Phase 1 (즉시 적용, 1-2일)
  ├── PDF 페이지 경계 보존 → Page-level chunking
  └── chunkSize 500 → 1024-2048 증가 시도 및 A/B 비교

Phase 2 (단기, 1주)
  ├── Parent-Child 청크 구조 도입
  │   ├── 검색: child chunk (128-256 토큰)
  │   └── LLM 전달: parent chunk (512-1024 토큰)
  └── chunk.vo.ts에 parentChunkId, sectionHeader 필드 추가

Phase 3 (중기, 2-3주) ← 효과 가장 큼
  ├── Contextual Embeddings 적용
  │   ├── 인제스트 시 각 청크에 Claude로 컨텍스트 생성
  │   └── "문서 전체에서 이 청크의 역할" 50-100 토큰 요약
  └── Contextual BM25 — 컨텍스트 포함 텍스트로 BM25 인덱스 생성
```

---

## Part 2: 쿼리 이해(Query Processing) 개선

### 연구 결과 요약

#### 1. HyDE 적용 조건 개선 ★ 즉시 적용 가능

**현재 문제:** 단어 수 5개 이하에만 HyDE 적용 → 긴 복잡한 질문에 오히려 미적용

**개선 방향:**
- 단어 수 기준 제거 → 모든 쿼리에 HyDE 적용 (짧은 쿼리에 특히 효과적)
- 단, 가상 문서와 원본 쿼리 간 코사인 유사도 < 0.5이면 HyDE 결과 폐기, 원본 쿼리로 fallback
- Hybrid HyDE: `(hypo_embedding + query_embedding) / 2` — 할루시네이션 방어

```typescript
// 개선된 HyDE 전략 (hyde.service.ts)
shouldApply(question: string): boolean {
  return question.trim().length > 0; // 단어 수 기준 제거, 항상 적용
}

// hybrid HyDE: 유사도 체크 후 평균 임베딩 반환
async resolveEmbedding(question: string, hypoDoc: string): Promise<number[]> {
  const [hypoVec, queryVec] = await Promise.all([
    this.embed(hypoDoc),
    this.embed(question),
  ]);
  const similarity = cosineSimilarity(hypoVec, queryVec);
  if (similarity < 0.5) return queryVec; // 가상 문서가 엉뚱한 방향이면 원본 쿼리로 fallback
  return average(hypoVec, queryVec);    // Hybrid HyDE
}
```

---

#### 2. Query Decomposition (ACL 2025, EMNLP 2025) ★★ Multi-hop 질문 대응

**출처:**
- https://doi.org/10.18653/v1/2025.acl-srw.32 (ACL 2025)
- https://aclanthology.org/2025.findings-emnlp.1022.pdf (UniRAG, EMNLP 2025)

복잡한 질문을 단순 서브 질문 여러 개로 분해해 각각 검색 후 결합:

```
원본: "A 제품과 B 제품의 2023년 매출을 비교하고, 성장률이 더 높은 제품의 핵심 전략을 설명해"
  ↓
서브 질문 1: "A 제품의 2023년 매출은?"
서브 질문 2: "B 제품의 2023년 매출은?"
서브 질문 3: "성장률이 더 높은 제품의 핵심 전략은?"
```

**성과 (MultiHop-RAG 데이터셋):**
- MRR@10: +36.7%
- F1: +11.6%
- Hits@10: 74.7% → 87.2%

**적용 조건:** 질문에 AND/또는/비교/차이/그리고 등 복합 의도 감지 시 트리거

---

#### 3. Conversational Query Rewriting ★ 대화 이력 활용

현재 시스템은 `conversationHistory`를 LLM 프롬프트에 넣지만, **쿼리 재작성에는 미활용**:

```typescript
// 현재: history는 LLM 답변 생성에만 사용
const messages = [
  { role: 'system', content: systemContent },
  ...(history ?? []),
  { role: 'user', content: question },  // 원본 질문 그대로 검색
];
```

**개선:** 대화 이력이 있을 때 질문을 독립적 쿼리로 재작성:

```
이전 대화: "우리 회사 환불 정책이 뭐야?"
현재 질문: "그럼 해외 배송은?"

재작성 후: "해외 배송 환불 정책은 무엇인가?"
```

- `ask.use-case.ts`의 `hybridSearch.execute()` 호출 전에 Claude로 쿼리 재작성
- `conversationHistory` 있을 때만 적용 (신규 대화는 불필요)

---

#### 4. ReDI: 쿼리 분해 + 의미 해석 (arXiv 2025.09)

**출처:** https://arxiv.org/html/2509.06544

3단계 파이프라인:
1. 복잡한 쿼리 → 서브 쿼리 분해
2. 각 서브 쿼리에 상세 의미 해석 추가 (embedding space 보강)
3. 각각 독립 검색 → fusion 전략으로 최종 순위 결정

BRIGHT, BEIR 벤치마크에서 sparse/dense 검색 모두 일관된 성능 향상

---

### 쿼리 이해 개선 우선순위 로드맵

```
Phase 1 (즉시 적용, 1일)
  └── HyDE: 단어 수 기준 제거 → 항상 적용 + 유사도 기반 fallback + Hybrid HyDE

Phase 2 (단기, 1주)
  ├── Conversational query rewriting
  │   └── sessionId 있고 이전 대화가 있으면 쿼리 독립화
  └── 복합 의도 감지 → Query Decomposition 트리거

Phase 3 (중기, 2-3주)
  └── 전체 Adaptive Query Pipeline
      ├── 단순 쿼리 → 직접 hybrid search
      ├── 짧고 모호한 쿼리 → HyDE
      ├── 복합/비교 쿼리 → Query Decomposition + Rerank
      └── 대화 연속 쿼리 → Contextual rewriting → HyDE
```

---

## 구현 우선순위 종합

아래 순서로 구현하면 최단 시간 내 최대 효과:

| 순위 | 개선 항목 | 예상 효과 | 구현 난이도 | 소요 기간 |
|------|-----------|----------|------------|---------|
| 1 | HyDE 적용 조건 제거 + Hybrid HyDE | 검색 품질 즉시 향상 | 낮음 | 1일 |
| 2 | chunkSize 1024로 증가 + 비교 테스트 | 컨텍스트 부족 해결 | 낮음 | 1일 |
| 3 | PDF 페이지 경계 보존 | 정확도 0.648 달성 가능 | 중간 | 2일 |
| 4 | Conversational query rewriting | 대화 연속성 향상 | 중간 | 3일 |
| 5 | Parent-Child 청킹 구조 | +15-30% 정확도 | 중간 | 1주 |
| 6 | **Contextual Embeddings** | **검색 실패 -35~67%** | 높음 | 2주 |
| 7 | Query Decomposition | Multi-hop MRR +36.7% | 높음 | 2주 |

---

## 핵심 참고 자료

| 주제 | 논문/자료 | 출처 |
|------|----------|------|
| Page-level Chunking | Finding the Best Chunking Strategy | https://developer.nvidia.com/blog/finding-the-best-chunking-strategy-for-accurate-ai-responses/ |
| Contextual Retrieval | Contextual Retrieval in AI Systems | https://www.anthropic.com/engineering/contextual-retrieval |
| Parent Document Retrieval | Parent Document Retrieval 2025 | https://thread-transfer.com/blog/2025-07-29-parent-document-retrieval/ |
| Adaptive Chunking | arXiv 2603.25333 | https://arxiv.org/pdf/2603.25333 |
| PIC Chunking | ACL 2025 Findings | https://aclanthology.org/2025.findings-acl.422.pdf |
| RSC | ICNLSP 2025 | https://aclanthology.org/2025.icnlsp-1.15.pdf |
| Max-Min Semantic | Springer 2025.06 | https://link.springer.com/article/10.1007/s10791-025-09638-7 |
| Query Decomposition | ACL SRW 2025 | https://doi.org/10.18653/v1/2025.acl-srw.32 |
| UniRAG | EMNLP 2025 Findings | https://aclanthology.org/2025.findings-emnlp.1022.pdf |
| HyDE 적용 가이드 | Milvus AI Reference | https://milvus.io/ai-quick-reference/what-is-hyde-hypothetical-document-embeddings-and-when-should-i-use-it |
| ReDI | arXiv 2509.06544 | https://arxiv.org/html/2509.06544 |

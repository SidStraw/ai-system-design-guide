<a id="reranking-strategies"></a>
# 重排序策略

重排序是檢索的第二階段：使用高精度模型，對一小組候選結果（Top 50-100）重新評分。它是「高效率搜尋」與「完美 grounding」之間的橋樑：第一階段檢索優化的是召回率，重排序優化的是精確率。如今在正式環境中主流的三種 reranker 為 BGE-Reranker-v2-m3、Cohere Rerank 3 與 Voyage rerank-2；實際選擇通常取決於成本模型、延遲尾端表現、語言覆蓋，以及你是否需要可自行託管的權重。

<a id="table-of-contents"></a>
## 目錄

- [為什麼需要重排序](#why-reranking)
- [重排序架構](#reranking-architectures)
- [重排序模型](#reranking-models)
- [實作模式](#implementation-patterns)
- [何時該進行重排序](#when-to-rerank)
- [基於 LLM 的重排序](#llm-based-reranking)
- [SLM 蒸餾](#slm-distillation)
- [正式環境考量](#production-considerations)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="why-reranking"></a>
## 為什麼需要重排序

<a id="the-quality-gap"></a>
### 品質落差

| 階段 | 模型 | 速度 | 品質 |
|-------|-------|-------|---------|
| Embedding 檢索 | Bi-encoder | 快（ms） | 良好 |
| 重排序 | Cross-encoder | 慢（10-100ms） | 更好 |

**為什麼會有這個落差：**
- Bi-encoder 會分別對 query 與 document 獨立編碼
- Cross-encoder 會聯合處理 query 與 document
- 聯合處理可以捕捉 bi-encoder 無法看見的互動關係

<a id="example"></a>
### 範例

```
Query: "How to configure CUDA memory"

Document 1: "Configure GPU memory using CUDA_VISIBLE_DEVICES..."
Document 2: "Memory management in CUDA applications..."
Document 3: "Configure RAM allocation for machine learning..."

Bi-encoder scores (cosine similarity):
- Doc 1: 0.72
- Doc 2: 0.75  <-- Ranked first (wrong)
- Doc 3: 0.71

Cross-encoder scores (relevance):
- Doc 1: 0.91  <-- Ranked first (correct)
- Doc 2: 0.67
- Doc 3: 0.42
```

Cross-encoder 可以看出 query 中的「CUDA memory」與 Doc 1 的「GPU memory...CUDA」是相互對應的。

---

<a id="reranking-architectures"></a>
## 重排序架構

<a id="bi-encoder-vs-cross-encoder"></a>
### Bi-Encoder 與 Cross-Encoder

**Bi-Encoder（第一階段）：**
```
Query --> Encoder --> Query Embedding -+
                                      +-> Similarity
Document --> Encoder --> Doc Embedding +
```
- 每份文件為 O(1)（embedding 可預先計算）
- 無法看到 query-document 之間的互動

**Cross-Encoder（重排序）：**
```
[Query, Document] --> Encoder --> Relevance Score
```
- 每個 query 為 O(n)（需處理每個候選）
- 能看到完整的 query-document 上下文
- 透過 **Attention Mechanism** 比較 query 中特定詞語如何改變 document 中詞語的含義（late interaction）

<a id="two-stage-pipeline"></a>
### 兩階段 Pipeline

正式環境中的檢索通常採用兩階段漏斗：

```
+----------------------------------------------------------------+
|  STAGE 1: Retrieval (Bi-Encoder)                                |
|                                                                 |
|  Query --> Embed --> Top-K candidates (K=100)                   |
|  Scale: Search 1 Billion docs. Cost: Low (ms).                 |
+----------------------------+-----------------------------------+
                             |
                             v
+----------------------------------------------------------------+
|  STAGE 2: Reranking (Cross-Encoder)                             |
|                                                                 |
|  For each candidate:                                            |
|    score = reranker([query, candidate])                         |
|  Scale: Search Top 100 docs. Cost: High (10-100ms).            |
|                                                                 |
|  Return Top-N by reranker score (N=5-10)                        |
+----------------------------------------------------------------+
```

<a id="multi-stage-pipeline"></a>
### 多階段 Pipeline

對超大型語料庫而言：

```
Stage 1: Sparse (BM25)      -> Top 1000
Stage 2: Dense (Bi-encoder) -> Top 100
Stage 3: Cross-encoder      -> Top 10
```

每個階段都以速度換取更高的準確性。

---

<a id="reranking-models"></a>
## 重排序模型

<a id="cross-encoder-models"></a>
### Cross-Encoder 模型

| 模型 | 規模 | 語言 | 品質 |
|-------|------|-----------|---------|
| ms-marco-MiniLM-L-6 | 22M | English | Good |
| bge-reranker-base | 278M | English | Very good |
| **bge-reranker-v2-m3** | 568M | Multilingual | Excellent |
| Cohere Rerank v3 | API | Multilingual | Excellent |
| Jina Reranker v2 | Various | Multilingual (8k+ tokens) | Very good |

**修正「Lost in the Middle」問題**：Reranker 會被訓練成無論相關資訊出現在 chunk 的哪個位置，都能優先辨識與排序，確保位於中段的資訊在送入最終 LLM 前也能被正確評分。

<a id="using-cross-encoders"></a>
### 使用 Cross-Encoder

```python
from sentence_transformers import CrossEncoder

# Load model
reranker = CrossEncoder('BAAI/bge-reranker-base')

def rerank(query: str, documents: list[str], top_k: int = 5) -> list[tuple[str, float]]:
    # Create pairs
    pairs = [[query, doc] for doc in documents]

    # Score all pairs
    scores = reranker.predict(pairs)

    # Sort by score
    scored_docs = sorted(
        zip(documents, scores),
        key=lambda x: x[1],
        reverse=True
    )

    return scored_docs[:top_k]
```

<a id="cohere-rerank"></a>
### Cohere Rerank

```python
import cohere

co = cohere.Client(api_key="...")

def cohere_rerank(
    query: str,
    documents: list[str],
    top_k: int = 5
) -> list[dict]:
    response = co.rerank(
        model="rerank-english-v3.0",
        query=query,
        documents=documents,
        top_n=top_k,
        return_documents=True
    )

    return [
        {
            "text": result.document.text,
            "score": result.relevance_score,
            "index": result.index
        }
        for result in response.results
    ]
```

<a id="model-selection-guide"></a>
### 模型選型指南

| 使用情境 | 建議模型 | 備註 |
|----------|-------------------|-------|
| English、自行託管 | bge-reranker-base | 平衡性佳 |
| 多語言 | bge-reranker-v2-m3 | 最佳開源選擇 |
| 低延遲 | MiniLM-L-6 | 快 4 倍 |
| 最高品質 | Cohere Rerank v3 | API，規模化時成本高 |
| 大批次 | Jina Reranker | 吞吐量佳 |
| 長 query（8k+） | Jina Reranker v2 | 可處理長上下文 |

---

<a id="implementation-patterns"></a>
## 實作模式

<a id="pattern-1-basic-reranking"></a>
### 模式 1：基礎重排序

```python
class RerankedRetriever:
    def __init__(
        self,
        vector_db,
        embedding_model,
        reranker,
        retrieval_k: int = 50,
        rerank_k: int = 5
    ):
        self.vector_db = vector_db
        self.embedding_model = embedding_model
        self.reranker = reranker
        self.retrieval_k = retrieval_k
        self.rerank_k = rerank_k

    def search(self, query: str) -> list[Document]:
        # Stage 1: Retrieve candidates
        query_embedding = self.embedding_model.encode(query)
        candidates = self.vector_db.search(
            query_embedding,
            top_k=self.retrieval_k
        )

        # Stage 2: Rerank
        pairs = [[query, c.text] for c in candidates]
        scores = self.reranker.predict(pairs)

        # Combine and sort
        for candidate, score in zip(candidates, scores):
            candidate.rerank_score = score

        reranked = sorted(candidates, key=lambda x: x.rerank_score, reverse=True)
        return reranked[:self.rerank_k]
```

<a id="pattern-2-batched-reranking"></a>
### 模式 2：批次重排序

```python
def batch_rerank(
    queries: list[str],
    candidates_per_query: list[list[str]],
    reranker,
    batch_size: int = 32
) -> list[list[tuple[str, float]]]:
    # Flatten all pairs
    all_pairs = []
    pair_mapping = []  # (query_idx, doc_idx)

    for q_idx, (query, candidates) in enumerate(zip(queries, candidates_per_query)):
        for d_idx, doc in enumerate(candidates):
            all_pairs.append([query, doc])
            pair_mapping.append((q_idx, d_idx))

    # Batch score
    all_scores = []
    for i in range(0, len(all_pairs), batch_size):
        batch = all_pairs[i:i + batch_size]
        scores = reranker.predict(batch)
        all_scores.extend(scores)

    # Reconstruct per-query results
    results = [[] for _ in queries]
    for (q_idx, d_idx), score in zip(pair_mapping, all_scores):
        results[q_idx].append((candidates_per_query[q_idx][d_idx], score))

    # Sort each query's results
    for i in range(len(results)):
        results[i].sort(key=lambda x: x[1], reverse=True)

    return results
```

<a id="pattern-3-async-reranking"></a>
### 模式 3：非同步重排序

```python
import asyncio

class AsyncReranker:
    def __init__(self, reranker, max_concurrent: int = 5):
        self.reranker = reranker
        self.semaphore = asyncio.Semaphore(max_concurrent)

    async def rerank_async(
        self,
        query: str,
        documents: list[str]
    ) -> list[tuple[str, float]]:
        async with self.semaphore:
            # Run reranking in thread pool
            loop = asyncio.get_event_loop()
            scores = await loop.run_in_executor(
                None,
                lambda: self.reranker.predict([[query, doc] for doc in documents])
            )
            return sorted(zip(documents, scores), key=lambda x: x[1], reverse=True)
```

---

<a id="when-to-rerank"></a>
## 何時該進行重排序

<a id="cost-benefit-analysis"></a>
### 成本效益分析

| 因素 | 不使用重排序 | 使用重排序 |
|--------|-------------------|----------------|
| 延遲 | 50-100ms | 150-300ms |
| 品質（NDCG） | 0.65 | 0.78 |
| 複雜度 | 簡單 | 中等 |
| 成本 | 基準 | +API 成本或 +算力 |

<a id="decision-framework"></a>
### 決策框架

**以下情況應總是重排序：**
- 品質至關重要（面向客戶、高風險場景）
- 取回的候選結果分數相近
- Query 複雜或包含多個子問題
- 預算允許增加延遲

**以下情況可略過重排序：**
- 延遲預算非常緊（總時延 <100ms）
- 取回的候選結果排序已非常明確
- Query 很簡單（單一詞彙查找）
- 在大規模情境下成本受限

<a id="inference-time-tradeoffs"></a>
### 推論時間取捨

| 階段 | 檢索（K） | 重排序（N） | 延遲 | 品質 |
|-------|---------------|------------|---------|---------|
| **Naive** | 5 | 0 | 50ms | Low |
| **Standard** | 50 | 5 | 150ms | High |
| **Enterprise**| 200 | 20 | 500ms | Max |

**關鍵規則**：如果你有 200ms 的預算，建議把 50ms 花在檢索、150ms 花在重排序。對 Top 50 結果做重排序，通常比從 vector DB 多檢索更多 chunks 有更高的投資報酬率。

<a id="optimal-candidate-count"></a>
### 最佳候選數量

在重排序前應先取回多少候選：

```python
def optimize_candidate_count(test_set, retriever, reranker):
    """Find optimal retrieval_k for reranking."""
    results = {}

    for retrieval_k in [10, 20, 50, 100, 200]:
        ndcg_scores = []
        latencies = []

        for query, relevant_docs in test_set:
            start = time.time()

            # Retrieve
            candidates = retriever.search(query, top_k=retrieval_k)

            # Rerank to top 5
            reranked = reranker.rerank(query, candidates, top_k=5)

            latency = time.time() - start
            latencies.append(latency)

            ndcg = compute_ndcg(reranked, relevant_docs)
            ndcg_scores.append(ndcg)

        results[retrieval_k] = {
            "ndcg": mean(ndcg_scores),
            "latency_p99": percentile(latencies, 99)
        }

    return results

# Typical findings:
# K=20:  NDCG 0.72, latency 120ms
# K=50:  NDCG 0.76, latency 180ms  <-- Often sweet spot
# K=100: NDCG 0.77, latency 280ms  <-- Diminishing returns
```

---

<a id="llm-based-reranking"></a>
## 基於 LLM 的重排序

<a id="using-llms-as-rerankers"></a>
### 把 LLM 當作 Reranker

LLM 也能為相關性打分，但成本很高：

```python
def llm_rerank(
    query: str,
    documents: list[str],
    model: str = "gpt-4o-mini"
) -> list[tuple[str, float]]:
    prompt = f"""Rate the relevance of each document to the query.
Query: {query}

Documents:
{format_documents(documents)}

For each document, output a relevance score from 0-10.
Format: DOC_NUM: SCORE
"""

    response = llm.generate(prompt)
    scores = parse_scores(response)

    return sorted(zip(documents, scores), key=lambda x: x[1], reverse=True)
```

**優點：**
- 能處理複雜的相關性判斷
- 理解細微差異與上下文
- 不必額外維護獨立模型

**缺點：**
- 大規模時成本高（是 cross-encoder 的 10-100 倍）
- 更慢（1-3s 對比 100ms）
- 非決定性

<a id="listwise-vs-pointwise-llm-reranking"></a>
### Listwise 與 Pointwise LLM 重排序

**Pointwise：**獨立評分每份文件
```
For document: [doc text]
Query: [query]
Rate relevance 0-10: _
```

**Listwise：**一起排序所有文件
```
Query: [query]
Rank these documents by relevance:
A: [doc1]
B: [doc2]
C: [doc3]
Output order: _
```

**Listwise 通常更好**，因為 LLM 可以直接比較多份文件。Frontier models（如 o1-mini 或 Sonnet 3.7）在這方面尤其強，但也會額外增加 1-2 秒延遲。通常只會用在高風險的企業搜尋場景（法律、醫療）。

<a id="sliding-window-for-many-documents"></a>
### 大量文件的 Sliding Window

```python
def sliding_window_rerank(
    query: str,
    documents: list[str],
    window_size: int = 10,
    step: int = 5
) -> list[str]:
    """Rerank many documents with LLM using sliding window."""
    ranked = list(range(len(documents)))

    for start in range(0, len(documents), step):
        window = ranked[start:start + window_size]

        # LLM ranks this window
        window_docs = [documents[i] for i in window]
        window_order = llm_listwise_rank(query, window_docs)

        # Update rankings
        for new_pos, old_idx in enumerate(window_order):
            ranked[start + new_pos] = window[old_idx]

    return [documents[i] for i in ranked]
```

---

<a id="slm-distillation"></a>
## SLM 蒸餾

為了解決基於 LLM 的重排序延遲問題，現在常見做法是使用 **蒸餾後的小型語言模型（SLM）**。

- **流程**：先用大型模型（例如 GPT-5.2）對 100 萬組 pair 進行重排序，再用這些標註資料去「蒸餾」出一個只有 0.1B 參數的小模型。
- **結果**：你可以用接近標準 CPU 查詢的延遲（<10ms），拿到大型模型約 95% 的重排序品質。
- **正式環境模式：**平時使用 cross-encoder；若重排序分數信心不足，再以 LLM 作為 fallback。

---

<a id="production-considerations"></a>
## 正式環境考量

<a id="latency-optimization"></a>
### 延遲最佳化

```python
class OptimizedReranker:
    def __init__(self, model_name: str, device: str = "cuda"):
        self.model = CrossEncoder(model_name, device=device)
        # Enable optimizations
        self.model.model.half()  # FP16

    def rerank(self, query: str, documents: list[str]) -> list[tuple[str, float]]:
        with torch.inference_mode():
            pairs = [[query, doc] for doc in documents]
            scores = self.model.predict(
                pairs,
                batch_size=32,
                show_progress_bar=False
            )
        return sorted(zip(documents, scores), key=lambda x: x[1], reverse=True)
```

**最佳化技巧：**
- FP16 推論：加速 2 倍
- Batching：分攤 overhead
- ONNX export：加速 1.5-2 倍
- TensorRT：加速 2-3 倍（NVIDIA）
- 模型蒸餾：以品質取捨換取 4 倍加速

<a id="caching-reranker-results"></a>
### 快取 Reranker 結果

```python
class CachedReranker:
    def __init__(self, reranker, cache_ttl: int = 3600):
        self.reranker = reranker
        self.cache = TTLCache(maxsize=10000, ttl=cache_ttl)

    def rerank(self, query: str, documents: list[str]) -> list[tuple[str, float]]:
        # Cache key includes query and doc hashes
        key = self._make_key(query, documents)

        if key in self.cache:
            return self.cache[key]

        result = self.reranker.rerank(query, documents)
        self.cache[key] = result
        return result

    def _make_key(self, query: str, documents: list[str]) -> str:
        doc_hash = hashlib.sha256(
            "".join(sorted(documents)).encode()
        ).hexdigest()[:16]
        query_hash = hashlib.sha256(query.encode()).hexdigest()[:16]
        return f"{query_hash}:{doc_hash}"
```

<a id="fallback-strategy"></a>
### Fallback 策略

```python
def rerank_with_fallback(
    query: str,
    candidates: list[Document],
    primary_reranker,
    timeout: float = 2.0
) -> list[Document]:
    try:
        # Try reranking with timeout
        result = timeout_call(
            primary_reranker.rerank,
            args=(query, candidates),
            timeout=timeout
        )
        return result
    except TimeoutError:
        # Fallback: return original order
        logger.warning("Reranker timeout, using original order")
        return candidates
    except Exception as e:
        logger.error(f"Reranker error: {e}")
        return candidates
```

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-is-a-cross-encoder-fundamentally-more-accurate-than-a-bi-encoder"></a>
### Q：為什麼 Cross-Encoder 在本質上比 Bi-Encoder 更準確？

**強答：**
Bi-Encoder 會在還不知道 query 之前，就先為 document 建立單一、固定的向量表示，因此會失去文字各部分之間的特定關係。Cross-Encoder 則把 query 與 document 當成一組輸入，並透過 **Attention Mechanism** 比較兩者。它能理解 query 中的特定詞語如何改變 document 中詞語的含義（late interaction），因此相關性評分會比兩個固定向量之間的簡單數學相似度更細膩。

**實務上：**第一階段檢索用 bi-encoder（速度），重排序用 cross-encoder（品質）。這樣能同時兼顧兩者優勢。

<a id="q-how-do-you-decide-how-many-candidates-to-rerank"></a>
### Q：你如何決定要對多少個候選結果做重排序？

**強答：**
本質上是在品質與延遲之間取捨：

**因素：**
- 每份文件的 reranker 延遲
- 總延遲預算
- 品質提升曲線（通常會逐漸遞減）
- 第一階段檢索的品質

**流程：**
1. Benchmark 每份文件的 reranker 延遲
2. 根據延遲預算計算可處理的最大候選數
3. 在不同 K 值下測試品質
4. 找出拐點（quality vs latency）

**常見結論：**
- K=20-50 通常最佳
- 超過 K=100 後，品質提升很有限
- 需依第一階段檢索品質調整

如果我的重排序預算是 200ms，而每份文件需 4ms，我會大約重排序 50 個候選結果。

<a id="q-when-would-you-use-llm-based-reranking"></a>
### Q：什麼情況下你會使用基於 LLM 的重排序？

**強答：**
以下情況適合使用 LLM 重排序：

1. **複雜相關性判斷：**Query 需要理解細微語意、上下文或 multi-hop 推理
2. **流量低：**不值得訓練或託管一個 cross-encoder
3. **要求最高品質：**例如法律、醫療、安全關鍵場景
4. **LLM 已在 pipeline 中：**邊際成本較低

**注意事項：**
- 大規模時成本高（是 cross-encoder 的 10-100 倍）
- 更慢（1-3s 對比 100ms）
- 非決定性
- 可能需要仔細做 prompt engineering

**正式環境模式：**平時使用 cross-encoder；在低信心的重排序分數上，再以 LLM 做 fallback。

<a id="q-how-do-you-handle-reranking-for-extremely-long-queries-eg-a-whole-paragraph"></a>
### Q：若 query 非常長（例如整段文字），你會如何處理重排序？

**強答：**
長 query 會帶來 cross-encoder 的「Token Budget」問題，因為它們通常只有 512 或 1024 token 限制。常見解法是 **Sliding Window Reranking** 或 **Query Summarization**。另一種做法是使用像 **Jina-Reranker-v2** 這種可支援 8k+ tokens 的專用模型。也常見先用短上下文、快速模型做一次「First-Pass Rerank」，再對前 5 名候選用高上下文 LLM 做「Second-Pass Rerank」。

---

<a id="references"></a>
## 參考資料

- Nogueira and Cho. "Passage Re-ranking with BERT" (2019)
- Nogueira et al. "Multi-Stage Document Ranking with BERT" (2019/2025 update)
- BAAI BGE Reranker: https://huggingface.co/BAAI/bge-reranker-base
- Cohere Rerank: https://docs.cohere.com/docs/rerank
- Sun et al. "Is ChatGPT Good at Search? Investigating Large Language Models as Re-Ranking Agents" (2023)

---

*上一節：[Hybrid Search](05-hybrid-search.md) | 下一節：[GraphRAG](07-graph-rag.md)*


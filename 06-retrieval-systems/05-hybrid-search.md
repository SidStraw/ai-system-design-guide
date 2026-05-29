<a id="hybrid-search"></a>
# 混合搜尋

Hybrid search 結合 dense（semantic）與 sparse（keyword）檢索，以同時取得兩者優勢。它已是正式環境 RAG 的基準線：Elasticsearch 的 `rrf` retriever、OpenSearch hybrid search、Weaviate、Qdrant 與 Azure AI Search，都原生提供 hybrid pipelines。

<a id="table-of-contents"></a>
## 目錄

- [為什麼要用 Hybrid Search](#why-hybrid-search)
- [Dense vs Sparse 檢索](#dense-vs-sparse-retrieval)
- [Hybrid Search 架構](#hybrid-search-architectures)
- [Fusion 方法](#fusion-methods)
- [Learned Sparse Embeddings（SPLADE）](#learned-sparse-embeddings-splade)
- [實作模式](#implementation-patterns)
- [調校與最佳化](#tuning-and-optimization)
- [正式環境考量](#production-considerations)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="why-hybrid-search"></a>
## 為什麼要用 Hybrid Search

Dense 與 sparse 檢索沒有誰能全面勝出；兩者各自在不同查詢類型上表現較強。

<a id="query-type-analysis"></a>
### 查詢類型分析

| 查詢類型 | 範例 | 較佳檢索方式 |
|------------|---------|------------------|
| 概念型 | "How do transformers learn?" | Dense |
| 關鍵字明確型 | "GPT-4 API rate limits" | Sparse |
| 命名實體 | "John Smith's research on BERT" | Sparse |
| 縮寫／代碼 | "What does HTTP 429 mean?" | Sparse |
| 改寫型 | "How to make AI faster" vs "LLM optimization" | Dense |
| 混合型 | "What is the cost of GPT-4o API?" | Hybrid |

**細節**：在技術文件中，若特定版本號與函式名稱承載了 90% 的資訊價值，dense-only retrieval 往往會失效。

<a id="the-gap-problem"></a>
### Gap 問題

Dense retrieval 可能漏掉精確匹配：

```
Query: "Configure NVIDIA_VISIBLE_DEVICES"
Document: "Set the NVIDIA_VISIBLE_DEVICES environment variable..."

Dense search may miss this because:
- "NVIDIA_VISIBLE_DEVICES" might tokenize poorly
- Semantic embedding does not capture exact string matching
- Training data may not have this specific term
```

Sparse search（BM25）則能因為精確 token match 立刻抓到它。

---

<a id="dense-vs-sparse-retrieval"></a>
## Dense vs Sparse 檢索

<a id="dense-semantic-retrieval"></a>
### Dense（語意）檢索

透過 neural embeddings 比對語意。

```python
def dense_search(query: str, top_k: int = 10) -> list[Result]:
    query_embedding = embedding_model.encode(query)
    results = vector_db.search(query_embedding, top_k=top_k)
    return results
```

**優勢：**
- 能理解改寫與同義詞
- 能捕捉概念相似度
- 支援跨語言（搭配 multilingual models）

**弱點：**
- 可能漏掉精確關鍵字匹配
- 不擅長 entities、codes、acronyms
- 需要 embedding model

<a id="sparse-keyword-retrieval"></a>
### Sparse（關鍵字）檢索

使用詞頻與統計方法（BM25、TF-IDF）。

```python
def sparse_search(query: str, top_k: int = 10) -> list[Result]:
    tokens = tokenize(query)
    results = bm25_index.search(tokens, top_k=top_k)
    return results
```

**優勢：**
- 非常適合精確匹配
- 擅長處理罕見詞、代碼與命名實體
- 快速且可解釋
- 不需要訓練

**弱點：**
- 抓不到語意相似
- 不理解同義詞
- 對 vocabulary mismatch 很敏感

<a id="head-to-head-comparison"></a>
### 正面比較

| 面向 | Dense | Sparse | Hybrid |
|--------|-------|--------|--------|
| 語意匹配 | 最佳 | 差 | 最佳 |
| 精確匹配 | 差 | 最佳 | 最佳 |
| 罕見詞 | 差 | 最佳 | 很好 |
| Zero-shot domain | 很好 | 最佳 | 最佳 |
| 延遲 | 中 | 快 | 中 |
| 實作難度 | 中 | 簡單 | 複雜 |

---

<a id="hybrid-search-architectures"></a>
## Hybrid Search 架構

<a id="architecture-1-parallel-retrieval-with-fusion"></a>
### Architecture 1：平行檢索 + Fusion

```
                    +------------------+
                    |      Query       |
                    +--------+---------+
                             |
              +--------------+--------------+
              v                             v
    +-------------------+         +-------------------+
    |  Dense Retrieval  |         |  Sparse Retrieval |
    |   (Vector DB)     |         |    (BM25/ES)      |
    +---------+---------+         +---------+---------+
              |                             |
              +--------------+--------------+
                             v
                    +-------------------+
                    |      Fusion       |
                    |  (RRF, weighted)  |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    |  Final Results    |
                    +-------------------+
```

**優點：**分工清楚，可對每一邊使用最強方案（例如 Pinecone + Algolia），也能獨立調校
**缺點：**要維護兩套系統，延遲較高（必須等最慢的引擎）

<a id="architecture-2-native-hybrid-single-system"></a>
### Architecture 2：原生 Hybrid（單一系統）

部分向量資料庫原生支援 hybrid：

```python
# Weaviate
results = client.query.get("Document", ["text"]).with_hybrid(
    query="Configure NVIDIA_VISIBLE_DEVICES",
    alpha=0.5  # 0 = sparse only, 1 = dense only
).do()

# Qdrant (with sparse vectors)
results = client.search(
    collection_name="docs",
    query_vector=NamedVector(name="dense", vector=dense_embedding),
    query_sparse_vector=NamedSparseVector(name="sparse", vector=sparse_vector),
)
```

**優點：**單一系統、維運較簡單、延遲較低
**缺點：**Fusion 客製化有限，keyword 與 vector infra 在擴展上彈性較小

<a id="architecture-3-staged-retrieval"></a>
### Architecture 3：分階段檢索

```
Query --> Sparse (fast, broad) --> Top 1000
                    |
                    v
          Dense reranking --> Top 100
                    |
                    v
           Cross-encoder --> Top 10
```

**優點：**效率高，每一層都在精煉結果
**缺點：**更複雜，且早期階段失誤會一路放大

---

<a id="fusion-methods"></a>
## Fusion 方法

<a id="reciprocal-rank-fusion-rrf"></a>
### Reciprocal Rank Fusion（RRF）

RRF 是整合兩種搜尋引擎結果的黃金標準。它不看 *score*（不同引擎的 score 無法直接比較），而是看 **rank**。

```python
def reciprocal_rank_fusion(
    rankings: list[list[str]],  # List of doc_id lists
    k: int = 60
) -> list[tuple[str, float]]:
    scores = defaultdict(float)

    for ranking in rankings:
        for rank, doc_id in enumerate(ranking):
            scores[doc_id] += 1 / (k + rank + 1)

    sorted_docs = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    return sorted_docs
```

**特性：**
- 以排名為基礎，不看原始分數
- 能抵抗 score scale 差異，避免某個引擎單純因分數數值高就「壓過」另一個引擎
- k 參數控制對排名位置的敏感度（k 越高，對名次越不敏感）
- 容易實作，除了 k 以外幾乎不用調參

**常見 k 值：**60（原論文）、實務上常見 10-100

<a id="weighted-score-fusion"></a>
### 加權分數 Fusion

把正規化後的分數加總：

```python
def weighted_fusion(
    dense_results: list[Result],
    sparse_results: list[Result],
    alpha: float = 0.5  # Weight for dense
) -> list[Result]:
    # Normalize scores to [0, 1]
    dense_normalized = normalize_scores(dense_results)
    sparse_normalized = normalize_scores(sparse_results)

    # Combine
    combined = {}
    for r in dense_normalized:
        combined[r.id] = alpha * r.score
    for r in sparse_normalized:
        combined[r.id] = combined.get(r.id, 0) + (1 - alpha) * r.score

    sorted_docs = sorted(combined.items(), key=lambda x: x[1], reverse=True)
    return sorted_docs

def normalize_scores(results: list[Result]) -> list[Result]:
    if not results:
        return []
    min_score = min(r.score for r in results)
    max_score = max(r.score for r in results)
    range_score = max_score - min_score + 1e-6

    return [
        Result(id=r.id, score=(r.score - min_score) / range_score)
        for r in results
    ]
```

**特性：**
- 使用實際分數（比單看排名資訊更多）
- 需要做 score normalization
- Alpha 控制 dense 與 sparse 的平衡

<a id="relative-score-fusion"></a>
### Relative Score Fusion

把分數分布納入考量：

```python
def relative_score_fusion(
    dense_results: list[Result],
    sparse_results: list[Result]
) -> list[Result]:
    # Use z-score normalization
    dense_normalized = z_score_normalize(dense_results)
    sparse_normalized = z_score_normalize(sparse_results)

    # Combine
    combined = {}
    for r in dense_normalized:
        combined[r.id] = r.score
    for r in sparse_normalized:
        combined[r.id] = combined.get(r.id, 0) + r.score

    return sorted(combined.items(), key=lambda x: x[1], reverse=True)

def z_score_normalize(results: list[Result]) -> list[Result]:
    scores = [r.score for r in results]
    mean = sum(scores) / len(scores)
    std = (sum((s - mean) ** 2 for s in scores) / len(scores)) ** 0.5 + 1e-6

    return [Result(id=r.id, score=(r.score - mean) / std) for r in results]
```

<a id="fusion-method-comparison"></a>
### Fusion 方法比較

| 方法 | 使用分數 | Query Adaptive | 複雜度 |
|--------|-------------|----------------|------------|
| RRF | 否（只看 ranks） | 否 | 低 |
| Weighted | 是 | 否 | 低 |
| Relative Score | 是 | 部分 | 中 |
| Learned | 是 | 是 | 高 |

---

<a id="learned-sparse-embeddings-splade"></a>
## Learned Sparse Embeddings（SPLADE）

正式環境的技術堆疊已不再只用 BM25（單純詞頻），而是開始把 **Learned Sparse Embeddings** 用在 hybrid search 的 sparse 支線。

**技術**：像 **SPLADE v3** 這類模型，會為字典中的每個詞預測「importance weights」。

**為什麼重要？**SPLADE 能做 query expansion。若你搜尋「CPU」，它可能會自動對「processor」賦予一點權重，即使 query 中根本沒有這個詞。它把 sparse search 的精確匹配能力，與 dense search 的概念能力，結合在同一種儲存格式中。

<a id="splade-implementation"></a>
### SPLADE 實作

```python
from transformers import AutoModelForMaskedLM, AutoTokenizer

class SpladeEncoder:
    def __init__(self, model_name="naver/splade-cocondenser-ensembledistil"):
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModelForMaskedLM.from_pretrained(model_name)

    def encode(self, text: str) -> dict[str, float]:
        inputs = self.tokenizer(text, return_tensors="pt", truncation=True)
        outputs = self.model(**inputs)

        # Get sparse weights
        weights = torch.max(
            torch.log(1 + torch.relu(outputs.logits)) * inputs["attention_mask"].unsqueeze(-1),
            dim=1
        ).values.squeeze()

        # Convert to sparse dict
        non_zero = weights.nonzero().squeeze().tolist()
        sparse_vec = {
            self.tokenizer.decode([idx]): weights[idx].item()
            for idx in non_zero
            if weights[idx] > 0
        }

        return sparse_vec
```

**什麼時候選 SPLADE 而不是 BM25 + Dense Hybrid：**SPLADE 會產生可儲存在現代向量資料庫（如 Milvus 或 Qdrant）中的 sparse vector，能與 dense vector 一起單次完成 hybrid search，而不需要額外維護 Elasticsearch 或 BM25 索引。不過，如果你的資料集中有極度罕見、非語言型 token（例如唯一序號），而神經模型在訓練中沒見過它們，那就還是用 BM25。

---

<a id="implementation-patterns"></a>
## 實作模式

<a id="pattern-1-elasticsearch-vector-db"></a>
### Pattern 1：Elasticsearch + Vector DB

```python
class HybridSearcher:
    def __init__(self, es_client, vector_db, embedding_model):
        self.es = es_client
        self.vector_db = vector_db
        self.embedding_model = embedding_model

    def search(self, query: str, top_k: int = 10, alpha: float = 0.5) -> list[Result]:
        # Parallel retrieval
        dense_future = self.dense_search(query, top_k * 3)
        sparse_future = self.sparse_search(query, top_k * 3)

        dense_results = dense_future.result()
        sparse_results = sparse_future.result()

        # Fusion
        combined = reciprocal_rank_fusion([
            [r.id for r in dense_results],
            [r.id for r in sparse_results]
        ])

        return combined[:top_k]

    async def dense_search(self, query: str, top_k: int) -> list[Result]:
        embedding = self.embedding_model.encode(query)
        return self.vector_db.search(embedding, top_k=top_k)

    async def sparse_search(self, query: str, top_k: int) -> list[Result]:
        response = self.es.search(
            index="documents",
            body={
                "query": {"match": {"content": query}},
                "size": top_k
            }
        )
        return [
            Result(id=hit["_id"], score=hit["_score"])
            for hit in response["hits"]["hits"]
        ]
```

<a id="pattern-2-native-hybrid-with-weaviate"></a>
### Pattern 2：以 Weaviate 實作原生 Hybrid

```python
import weaviate

def hybrid_search_weaviate(
    client: weaviate.Client,
    query: str,
    alpha: float = 0.5,
    top_k: int = 10
) -> list[dict]:
    result = client.query.get(
        "Document",
        ["text", "title", "source"]
    ).with_hybrid(
        query=query,
        alpha=alpha,  # 0 = BM25 only, 1 = vector only
        fusion_type=weaviate.HybridFusion.RELATIVE_SCORE
    ).with_limit(top_k).do()

    return result["data"]["Get"]["Document"]
```

---

<a id="tuning-and-optimization"></a>
## 調校與最佳化

<a id="alpha-tuning"></a>
### Alpha 調校

Alpha 參數用來平衡 dense 與 sparse：

```python
def find_optimal_alpha(
    test_queries: list[tuple[str, list[str]]],  # (query, relevant_doc_ids)
    alpha_range: list[float] = [0.0, 0.3, 0.5, 0.7, 1.0]
) -> float:
    best_alpha = 0.5
    best_ndcg = 0

    for alpha in alpha_range:
        ndcg_scores = []
        for query, relevant in test_queries:
            results = hybrid_search(query, alpha=alpha)
            ndcg = compute_ndcg(results, relevant)
            ndcg_scores.append(ndcg)

        avg_ndcg = sum(ndcg_scores) / len(ndcg_scores)
        if avg_ndcg > best_ndcg:
            best_ndcg = avg_ndcg
            best_alpha = alpha

    return best_alpha
```

**最佳實務／常見結果：**
- 技術文件與程式碼：alpha 0.3-0.4（關鍵字成分較重）
- 一般文字：alpha 0.5（平衡）
- 聊天與創意探索：alpha 0.7-0.9（語意成分較重）

<a id="query-adaptive-alpha"></a>
### Query-Adaptive Alpha

依每個 query 預測最佳 alpha：

```python
def predict_alpha(query: str) -> float:
    # Heuristics-based
    has_quotes = '"' in query
    has_code = any(c in query for c in ['_', '()', '{}', '[]'])
    has_numbers = any(c.isdigit() for c in query)

    # More sparse for exact match queries
    if has_quotes or has_code:
        return 0.3
    if has_numbers:
        return 0.4

    # More semantic for natural language
    if len(query.split()) > 5:
        return 0.7

    return 0.5  # Default balanced
```

<a id="retrieval-depth"></a>
### 檢索深度

在 fusion 前，應先取回多少結果：

```python
# Rule of thumb: fetch 3-5x more from each source
def hybrid_search(query: str, final_k: int = 10):
    fetch_k = final_k * 4

    dense_results = dense_search(query, top_k=fetch_k)
    sparse_results = sparse_search(query, top_k=fetch_k)

    fused = rrf([dense_results, sparse_results])
    return fused[:final_k]
```

---

<a id="production-considerations"></a>
## 正式環境考量

<a id="latency-budget"></a>
### 延遲預算

```
Typical hybrid search latency breakdown:

Dense embedding:           30-50ms
Dense retrieval:          30-50ms
Sparse retrieval:         20-40ms  (parallel with dense)
Fusion:                    1-5ms
Total:                   60-100ms
```

**最佳化方式：**
- 讓 dense 與 sparse 平行執行
- 為常見 query 預先計算 embeddings
- 兩邊都使用 approximate search
- 對重複 query 快取 fusion 結果

<a id="caching-strategy"></a>
### 快取策略

```python
class HybridSearchCache:
    def __init__(self, ttl_seconds: int = 300):
        self.cache = TTLCache(ttl=ttl_seconds)

    def search(self, query: str, **kwargs) -> list[Result]:
        cache_key = self._make_key(query, kwargs)

        if cache_key in self.cache:
            return self.cache[cache_key]

        results = self._do_search(query, **kwargs)
        self.cache[cache_key] = results
        return results

    def _make_key(self, query: str, kwargs: dict) -> str:
        return hashlib.sha256(
            f"{query}:{sorted(kwargs.items())}".encode()
        ).hexdigest()
```

<a id="fallback-strategy"></a>
### Fallback 策略

```python
def hybrid_search_with_fallback(query: str, top_k: int = 10) -> list[Result]:
    try:
        return hybrid_search(query, top_k=top_k)
    except DenseSearchError:
        # Fallback to sparse only
        return sparse_search(query, top_k=top_k)
    except SparseSearchError:
        # Fallback to dense only
        return dense_search(query, top_k=top_k)
```

---

<a id="interview-questions"></a>
## 面試題

<a id="q-when-would-you-use-hybrid-search-over-pure-dense-search"></a>
### Q：什麼情況下你會用 hybrid search，而不是純 dense search？

**強回答：**
我會在以下情況使用 hybrid search：

1. **查詢包含特定詞彙：**產品代碼、API 名稱、錯誤碼。Dense search 可能漏掉精確匹配。

2. **領域詞彙非常專門：**技術文件、法律、醫療。Sparse 更能抓住精確術語。

3. **Zero-shot retrieval：**新領域尚未有 fine-tuned embeddings。Sparse 能提供穩健基線。

4. **品質非常重要：**Hybrid 的表現幾乎不會比單獨任一方法差，只是複雜度更高。

**我會維持純 dense 的情況：**
- 查詢本質上是概念型／語意型
- 延遲預算非常嚴格
- 更重視架構簡單
- Embedding model 已對該領域充分調校

最後仍要以資料說話。我會用真實查詢分布對 hybrid 與 dense 做 A/B test。

<a id="q-why-is-reciprocal-rank-fusion-rrf-safer-than-simple-score-addition"></a>
### Q：為什麼 Reciprocal Rank Fusion（RRF）比「Simple Score Addition」更安全？

**強回答：**
Simple score addition 很危險，因為向量分數（例如 Cosine Similarity：0.0 到 1.0）與關鍵字分數（例如 BM25：0 到無限大）根本不在同一尺度。一次幸運的 BM25 關鍵字命中，可能用超高分數把 10 個高度相關的語意結果完全淹沒。RRF 忽略絕對分數，只看相對順序（rank），因此在數學上對 outliers 與不同引擎的 score drift 更穩健。

<a id="q-when-would-you-choose-splade-over-the-standard-bm25-dense-hybrid-approach"></a>
### Q：什麼情況下你會選 SPLADE，而不是標準的 BM25 + Dense Hybrid？

**強回答：**
當我想簡化基礎設施時，我會選 SPLADE。SPLADE 會產生 sparse vector，能和 dense vector 一起存進許多現代向量資料庫（如 Milvus 或 Qdrant），因此資料庫可以單次完成「Hybrid search」，不需要額外的 Elasticsearch 或 BM25 索引。不過，如果資料集中有極度罕見、非語言型 token（例如唯一序號），而神經模型可能沒在訓練中見過，我還是會選 BM25。

<a id="q-how-do-you-balance-dense-vs-sparse-in-hybrid-search"></a>
### Q：你如何在 hybrid search 中平衡 dense 與 sparse？

**強回答：**
Alpha 參數負責平衡（通常 alpha 代表 dense 權重）：

**調校方法：**
1. 先從 alpha=0.5（等權）開始
2. 建立含 queries 與 relevance labels 的評估集
3. 對 [0.1, 0.3, 0.5, 0.7, 0.9] 做 grid search
4. 在每個設定上量測 NDCG 或 MRR
5. 選擇使評估指標最大的 alpha

**Query-adaptive tuning：**
- 偵測 query 類型（keyword-heavy、conceptual、mixed）
- 依 query 動態調整 alpha
- 可用簡單 heuristics，也可用 learned classifier

**經驗法則：**
- 技術／程式碼查詢：alpha 0.3-0.4
- 一般文字：alpha 0.5
- 對話式查詢：alpha 0.7-0.8

---

<a id="references"></a>
## 參考資料

- Cormack et al.「Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods」（2009）
- Formal et al.「SPLADE: Sparse Lexical and Expansion Model for First Stage Ranking」（2021/2025）
- Weaviate Hybrid Search: https://weaviate.io/developers/weaviate/search/hybrid
- Qdrant Hybrid Search: https://qdrant.tech/documentation/concepts/hybrid-queries/

---

*上一篇：[向量資料庫](04-vector-databases.md) | 下一篇：[重排序策略](06-reranking-strategies.md)*

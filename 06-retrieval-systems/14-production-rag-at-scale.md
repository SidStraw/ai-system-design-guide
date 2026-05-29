<a id="production-rag-at-scale"></a>
# 大規模正式環境 RAG

Production RAG 已不再是週末專案，而是一個分散式系統：它包含 retrieval pipeline、快取層、routing logic、自我修正迴圈、多租戶隔離與成本控制，且全部都必須在嚴格的延遲 SLA 下運作。當 RAG 在正式環境失敗時，約有 73% 的情況出在 retrieval，而不是 generation；因此真正成功的企業部署，會把知識來源而不是模型本身，視為主要投資標的。

<a id="table-of-contents"></a>
## 目錄

- [RAG 與 Long Context](#rag-vs-long-context)
- [查詢路由與分類](#query-routing)
- [RAG 的語意快取](#semantic-caching)
- [多索引策略](#multi-index)
- [RAG pipeline 最佳化](#pipeline-optimization)
- [Corrective RAG：自我檢查式檢索](#corrective-rag)
- [Adaptive Retrieval](#adaptive-retrieval)
- [成本最佳化模式](#cost-optimization)
- [失敗模式與除錯](#failure-modes)
- [監控與告警](#monitoring)
- [擴展到百萬級文件](#scaling)
- [多租戶 RAG 隔離](#multi-tenant)
- [真實世界架構範例](#architectures)
- [系統設計面試角度](#interview)
- [參考資料](#references)

---

<a id="rag-vs-long-context"></a>
## RAG 與 Long Context

隨著主要 frontier model 家族都已支援 100 萬+ token context window（Claude Opus 4.7、Claude Sonnet 4.6、GPT-5.5、Gemini 3.1 Pro、Qwen 3.6 Plus、Llama 4 Maverick），問題已不再是「RAG 還是 long context？」，而是「什麼情況下各自更好？」

<a id="the-decision-matrix"></a>
### 決策矩陣

```
                    Small Corpus           Large Corpus
                    (<100K tokens)         (>1M tokens)
                 +---------------------+---------------------+
  Static Data    |  Long Context Wins  |  RAG Required       |
  (rarely        |  - Stuff it all in  |  - Can't fit in     |
   changes)      |  - Simpler arch     |    context window   |
                 |  - No index needed  |  - Index + retrieve |
                 +---------------------+---------------------+
  Dynamic Data   |  Hybrid Approach    |  RAG Required       |
  (updates       |  - Cache context    |  - Incremental      |
   frequently)   |  - Invalidate on    |    indexing          |
                 |    change           |  - Real-time updates |
                 +---------------------+---------------------+
  Multi-User     |  RAG Preferred      |  RAG Required       |
  (per-user      |  - Personalized     |  - Tenant isolation  |
   data)         |    retrieval        |  - Access control    |
                 +---------------------+---------------------+
```

<a id="head-to-head-comparison"></a>
### 正面比較

| 維度 | RAG | Long Context（100 萬 tokens） |
|-----------|-----|--------------------------|
| **平均查詢成本** | ~$0.0001 | ~$0.10 |
| **平均延遲（p50）** | ~1s | ~30–45s |
| **特定事實的精準度** | 高（目標式檢索） | 中段資訊會退化 |
| **跨文件綜合能力** | 弱（context 有限） | 強（能看見全部） |
| **語料規模上限** | 幾乎無上限 | ~100 萬 tokens |
| **資料新鮮度** | 分鐘級（增量索引） | 需整體重載 |
| **1000 QPS 成本** | ~$100/天 | ~$100,000/天 |

<a id="the-lost-in-the-middle-problem"></a>
### 「Lost in the Middle」問題

LLM 不會平均地關注整個 context window。放在 long context 中段的資訊，其準確率通常比開頭或結尾低 30% 以上。RAG 完全繞開這個問題，因為它只把最相關的 chunk 放入短而聚焦的 context。

<a id="best-practice-the-hybrid-pattern"></a>
### 最佳實務：Hybrid 模式

最佳架構通常是兩者結合：先用 RAG 從大型語料中找出 top candidate，再把這些候選載入 long context window，做跨文件推理。

```
  User Query
      |
      v
+------------------+     +-------------------+
|  RAG Retrieval   |---->|  Long Context     |
|  (Find top 20    |     |  Synthesis        |
|   from 10M docs) |     |  (Reason across   |
+------------------+     |   20 docs deeply) |
                          +-------------------+
                                  |
                                  v
                          Final Answer with
                          Cross-Doc Citations
```

**經驗法則**：如果語料放得進 context、你負擔得起延遲與成本，就用 long context；否則用 RAG。對大多數受成本與延遲約束的正式環境系統而言，RAG 仍是正確預設值。

---

<a id="query-routing-and-classification"></a>
## 查詢路由與分類

不是每個 query 都需要 retrieval。正式環境系統會先對進來的 query 做分類，再導向最適合的處理路徑。

<a id="the-four-path-router"></a>
### 四路徑 Router

```
                         User Query
                             |
                             v
                    +------------------+
                    |  Query Classifier |
                    |  (LLM or trained  |
                    |   classifier)     |
                    +--------+---------+
                             |
            +--------+-------+-------+--------+
            |        |               |        |
            v        v               v        v
        +------+ +--------+    +--------+ +--------+
        |Direct| |Simple  |    |Complex | |Agentic |
        | LLM  | |  RAG   |    |  RAG   | |  RAG   |
        +------+ +--------+    +--------+ +--------+
        "What    "What is      "Compare   "Analyze
        is 2+2?" our refund    Q3 vs Q4   all legal
                  policy?"     revenue    risks in
                               trends"    these 50
                                          contracts"
```

<a id="classification-signals"></a>
### 分類訊號

| 訊號 | Direct LLM | Simple RAG | Complex RAG | Agentic RAG |
|--------|-----------|------------|-------------|-------------|
| **需要私有資料** | 否 | 是 | 是 | 是 |
| **單跳可答** | 是 | 是 | 否 | 否 |
| **需要多來源** | 否 | 否 | 是 | 是 |
| **需要推理鏈** | 否 | 否 | 可能 | 是 |
| **需要即時資料** | 否 | 可能 | 可能 | 是 |

<a id="implementation-lightweight-router"></a>
### 實作：輕量 Router

```python
class QueryRouter:
    """Routes queries to the optimal retrieval strategy."""

    def __init__(self, classifier_model: str = "gpt-4o-mini"):
        self.classifier = classifier_model
        self.route_counts = Counter()  # for monitoring

    async def classify(self, query: str, user_context: dict) -> str:
        # Step 1: Rule-based fast path
        if self._is_trivial(query):
            return "direct_llm"

        # Step 2: Check if query references private/org data
        if not self._needs_retrieval(query, user_context):
            return "direct_llm"

        # Step 3: LLM-based complexity classification
        complexity = await self._assess_complexity(query)

        if complexity == "simple":
            return "simple_rag"
        elif complexity == "multi_hop":
            return "complex_rag"
        else:
            return "agentic_rag"

    def _is_trivial(self, query: str) -> bool:
        """Fast regex/keyword check for trivial queries."""
        trivial_patterns = [
            r"^(what is|define|explain)\s+\w+$",
            r"^(hi|hello|thanks|bye)",
        ]
        return any(re.match(p, query.lower()) for p in trivial_patterns)

    async def _assess_complexity(self, query: str) -> str:
        """Use a small, fast model to classify complexity."""
        prompt = f"""Classify this query's retrieval complexity:
        - "simple": needs one document lookup
        - "multi_hop": needs 2-3 lookups, comparison, or synthesis
        - "agentic": needs planning, tool use, or iterative search

        Query: {query}
        Classification:"""

        result = await llm_call(self.classifier, prompt, max_tokens=10)
        return result.strip().lower()
```

<a id="domain-specific-routing"></a>
### 領域導向路由

對於擁有多個知識領域的系統，應先將 query 導到正確索引，再進行檢索。

```python
# Rule-based domain routing
DOMAIN_RULES = {
    "revenue|sales|quota|ARR":     "financial_index",
    "policy|handbook|PTO|benefits": "hr_index",
    "API|endpoint|SDK|integration": "engineering_index",
    "compliance|GDPR|SOC2|audit":   "legal_index",
}

# Embedding-based domain routing (for ambiguous queries)
class DomainRouter:
    def __init__(self):
        self.domain_centroids = {}  # pre-computed per domain

    def route(self, query_embedding: list[float]) -> str:
        similarities = {
            domain: cosine_sim(query_embedding, centroid)
            for domain, centroid in self.domain_centroids.items()
        }
        return max(similarities, key=similarities.get)
```

---

<a id="semantic-caching-for-rag"></a>
## RAG 的語意快取

語意快取會辨識新 query 是否與先前 query 有幾乎相同的語意，並直接重用快取結果。正式環境系統若把 semantic cache 調校得當，常可帶來最高 68% 成本下降與 65 倍延遲改善。

<a id="three-layer-caching-architecture"></a>
### 三層快取架構

```
  User Query
      |
      v
+---------------------+
| Layer 1: Exact Cache |  Hash(query) -> response
| (Redis/Memcached)    |  TTL: 1 hour
| Hit rate: ~15-25%    |  Latency: <5ms
+----------+----------+
           | miss
           v
+---------------------+
| Layer 2: Semantic    |  Embed(query) -> nearest neighbor
| Cache (Vector DB)    |  Threshold: cosine > 0.95
| Hit rate: ~20-35%    |  Latency: <50ms
+----------+----------+
           | miss
           v
+---------------------+
| Layer 3: Document    |  Cache retrieved chunks
| Cache               |  Skip re-embedding
| (saves embedding $) |  TTL: until doc changes
+----------+----------+
           | miss
           v
    Full RAG Pipeline
```

<a id="semantic-cache-implementation"></a>
### Semantic Cache 實作

```python
class SemanticCache:
    """Cache RAG responses by query semantic similarity."""

    def __init__(self, vector_store, similarity_threshold: float = 0.95):
        self.vector_store = vector_store
        self.threshold = similarity_threshold
        self.response_store = {}  # query_id -> cached response

    async def get(self, query: str) -> Optional[CachedResponse]:
        # Step 1: Exact match (fast path)
        exact_key = hashlib.sha256(query.encode()).hexdigest()
        if exact_key in self.response_store:
            return self.response_store[exact_key]

        # Step 2: Semantic match
        query_embedding = await embed(query)
        results = self.vector_store.search(
            query_embedding, top_k=1
        )

        if results and results[0].score >= self.threshold:
            cached_id = results[0].metadata["response_id"]
            cached = self.response_store.get(cached_id)
            if cached and not cached.is_expired():
                return cached

        return None

    async def put(
        self, query: str, response: str,
        sources: list[str], ttl_seconds: int = 3600
    ):
        query_embedding = await embed(query)
        response_id = str(uuid4())

        # Store the embedding for future similarity lookups
        self.vector_store.upsert(
            id=response_id,
            embedding=query_embedding,
            metadata={"response_id": response_id}
        )

        # Store the actual response
        self.response_store[response_id] = CachedResponse(
            response=response,
            sources=sources,
            created_at=time.time(),
            ttl=ttl_seconds,
        )
```

<a id="cache-invalidation-strategies"></a>
### 快取失效策略

| 策略 | 觸發條件 | 使用情境 |
|----------|---------|----------|
| **基於 TTL** | 固定時間過期 | 一般查詢、新聞 |
| **事件驅動** | 文件更新 webhook | 知識庫 |
| **版本標記** | 文件版本不符 | 高法規要求場景 |
| **信心門檻** | 檢索分數低 | 高波動領域 |

**關鍵規則**：一定要把來源文件 ID 與回應一併快取。只要任一來源文件更新，就要讓所有引用該文件的快取項目失效。

```python
# Webhook-based cache invalidation
@app.post("/webhook/document-updated")
async def on_document_updated(doc_id: str):
    # Find all cache entries that used this document
    affected = cache_index.find_by_source(doc_id)
    for entry in affected:
        semantic_cache.invalidate(entry.response_id)
    logger.info(f"Invalidated {len(affected)} cache entries for doc {doc_id}")
```

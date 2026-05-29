<a id="vector-databases"></a>
# 向量資料庫

向量資料庫是專門為儲存、索引與搜尋高維 embeddings 而設計的系統。市場已分化成 **Managed Serverless** 與 **Specialized High-Performance** 兩大類。我們不再問「它支援 vector search 嗎？」（Postgres、Redis、Mongo 都支援），而是問 **「它能在完整 metadata filtering 下，於 sub-100ms P99 規模化到 1 億以上向量嗎？」**

<a id="table-of-contents"></a>
## 目錄

- [什麼是向量資料庫](#what-is-a-vector-database)
- [向量搜尋基礎](#vector-search-fundamentals)
- [索引演算法](#indexing-algorithms)
- [競爭版圖](#competitive-landscape)
- [資料庫詳細比較](#detailed-database-comparison)
- [Metadata Filtering](#metadata-filtering)
- [查詢模式](#query-patterns)
- [正式環境營運](#production-operations)
- [託管 vs 自架（TCO 分析）](#managed-vs-self-hosted-tco-analysis)
- [選型框架](#selection-framework)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="what-is-a-vector-database"></a>
## 什麼是向量資料庫

向量資料庫用來儲存 embeddings（dense vectors），並支援對它們進行高速相似度搜尋。

```
Traditional DB:      SELECT * FROM docs WHERE category = 'tech'
Vector DB:           SELECT * FROM docs ORDER BY similarity(embedding, query_embedding) LIMIT 10
```

<a id="core-capabilities"></a>
### 核心能力

| 能力 | 用途 |
|------------|---------|
| 向量儲存 | 持久化高維 embeddings |
| 相似度搜尋 | 快速找出最近鄰 |
| Metadata filtering | 將向量搜尋與屬性篩選結合 |
| CRUD 操作 | 隨資料變動更新 embeddings |
| 擴充能力 | 處理數百萬到數十億向量 |

<a id="why-not-general-databases"></a>
### 為什麼不直接用通用資料庫？

傳統資料庫可以儲存向量，但缺乏最佳化搜尋能力：

| 方法 | 搜尋複雜度 | 實務上可擴展 |
|----------|-------------------|-------------------|
| Brute force（PostgreSQL pgvector） | O(n * d) | 約到 ~1M vectors 還可以 |
| ANN index（專用 vector DB） | O(log n) 或 O(1) | 可以，到數十億 |

---

<a id="vector-search-fundamentals"></a>
## 向量搜尋基礎

<a id="exact-vs-approximate-search"></a>
### 精確搜尋 vs Approximate Search

**Exact（brute force）：**
- 將 query 與每個已儲存向量逐一比較
- 每次查詢複雜度為 O(n * d)
- 準確率完美

**Approximate Nearest Neighbor (ANN)：**
- 使用索引結構剪枝搜尋空間
- 次線性複雜度
- Recall 會略低一些（通常 95-99%）

<a id="distance-metrics"></a>
### 距離度量

| Metric | 公式 | 範圍 | 最適用 |
|--------|---------|-------|----------|
| Cosine | 1 - (a . b) / (norm(a) * norm(b)) | [0, 2] | 文字 embeddings |
| Euclidean (L2) | sqrt(sum((a - b)^2)) | [0, inf) | 圖像 embeddings |
| Dot product | a . b | (-inf, inf) | 已正規化向量 |

**對文字 embeddings 而言：**使用 cosine similarity（若向量已預先 normalize，則可用 dot product）。

<a id="recall-vs-latency-tradeoff"></a>
### Recall vs Latency 的取捨

```
                    ^ Recall
                    |
               100% | ------------------ Brute force
                    |         *          Well-tuned ANN
                    |      *
                    |   *
                95% |*                   Fast ANN
                    |
                    +-----+-------+------> Latency
                       1ms      10ms
```

ANN 索引會用部分準確度換取速度。要依你的需求做 tuning。

---

<a id="indexing-algorithms"></a>
## 索引演算法

<a id="hnsw-hierarchical-navigable-small-world"></a>
### HNSW（Hierarchical Navigable Small World）

這是正式環境中最常見的 **in-memory** 向量搜尋演算法。

**運作方式：**
1. 建立一個節點為向量的圖
2. 連接到附近鄰居
3. 建立多層抽象（hierarchical）
4. 搜尋時：從上層往下導航，採用 greedy nearest neighbor

```
Layer 2:   *--------*--------*
           |        |        |
Layer 1:   *--*--*--*--*--*--*
           |  |  |  |  |  |  |
Layer 0:   ********************  (all vectors)
```

**優點：**
- 卓越的 recall/latency 平衡
- 不需要訓練
- 原生支援更新

**缺點：**
- 很吃記憶體（圖結構）
- 索引大小約為向量資料的 ~1.5-2 倍
- 1536 維、1000 萬向量約需 ~80GB RAM

**關鍵參數：**
- `M`：每個節點最大連線數（16-64）
- `ef_construction`：建索引時的探索深度（100-500）
- `ef_search`：查詢時的探索深度（50-200）

<a id="diskann-ssd-based"></a>
### DiskANN（SSD-based）

這是 **PB 級搜尋** 的業界標準。

**運作方式：**
- 將圖主要放在 SSD（NVMe）上，RAM 中只保留很小的索引
- 使用 Vamana 演算法高效遍歷磁碟上的圖結構

**優點：**
- 在十億級資料集上，相較 HNSW 成本便宜 10 倍，延遲只多不到 5ms
- 與 HNSW 相比，RAM 需求可降低 90-95%

**缺點：**
- 延遲仍略高於純 in-memory HNSW
- 更適合非即時搜尋場景

**例子：**1536 維的 1 億向量索引若用 HNSW，幾乎需要 1TB RAM。使用 DiskANN，可在維持 sub-10ms query time 的同時，把 RAM 需求壓低 90-95%。

<a id="ivf-inverted-file-index"></a>
### IVF（Inverted File Index）

先將向量分群，只搜尋相關群集。

**運作方式：**
1. 使用 k-means 建立 centroids
2. 將每個向量指派到最近 centroid
3. 查詢時先找最近 centroids，再搜尋那些群集

**優點：**
- 記憶體需求低於 HNSW
- 可搭配 quantization（IVF-PQ）

**缺點：**
- 需要訓練
- 更新通常要重分群，或採用混合策略

**關鍵參數：**
- `nlist`：群集數（經驗法則為 sqrt(n)）
- `nprobe`：查詢時要搜尋的群集數

<a id="product-quantization-pq"></a>
### Product Quantization (PQ)

透過壓縮向量來降低記憶體占用並加速比較。

**運作方式：**
1. 將向量切成多個 subvector
2. 各自量化到對應 codebook
3. 儲存 codes 而非完整向量

**記憶體縮減：**典型可達 4-32 倍

**取捨：**因量化損失而降低準確率

<a id="flat-index-brute-force"></a>
### Flat Index（Brute Force）

不做近似，直接精確搜尋。

**適用時機：**
- 少於 100K 向量
- 準確率極度重要
- 可接受較寬鬆的延遲預算

<a id="algorithm-comparison"></a>
### 演算法比較

| 演算法 | 記憶體 | 建置時間 | 查詢速度 | Recall | 更新 |
|-----------|--------|------------|-------------|--------|---------|
| HNSW | 高 | 中 | 非常快 | 95-99% | 好 |
| DiskANN | 低（SSD） | 中 | 快 | 95-99% | 普通 |
| IVF | 中 | 快 | 快 | 90-98% | 普通 |
| IVF-PQ | 低 | 快 | 快 | 85-95% | 普通 |
| Flat | 低 | 無 | 慢 | 100% | 即時 |

---

<a id="competitive-landscape"></a>
## 競爭版圖

<a id="vector-native-dedicated"></a>
### Vector-Native（專用）

| 資料庫 | 類型 | 最適合 | 計價模式 |
|----------|------|----------|---------------|
| **Pinecone** | 託管雲端（serverless standard） | 容易起步、易擴展、託管 SLA | Per vector-hour |
| **Qdrant** | Open source / Cloud（Rust，高效能） | 自架控制、常見工作負載下最快的開源方案之一（10M vectors 約 12ms p99） | 雲端按 GB 或免費 |
| **Weaviate** | Open source / Cloud | 單次查詢內建 hybrid（BM25 + dense + metadata）、multimodal | Per dimension-hour |
| **Milvus** | Open source / Cloud（Zilliz） | 分散式擴展（50M+ vectors）、異質節點、分層儲存 | 自架免費或 Zilliz Cloud |
| **Chroma** | Open source | 原型開發、本機開發、embedded 使用 | 免費 |

<a id="general-purpose-pluginextension"></a>
### General-Purpose（Plugin／Extension）

| 資料庫 | 類型 | 最適合 | 計價模式 |
|----------|------|----------|---------------|
| **pgvector (v0.8+)** | PostgreSQL extension | 小規模、已在用 PG（現已支援 HNSW + IVFFlat） | 只算 compute |
| **Elasticsearch (v9.0)** | Search engine | 搭配 cross-entropy fusion 的 Hybrid Search | 授權制 |

---

<a id="detailed-database-comparison"></a>
## 資料庫詳細比較

<a id="feature-matrix"></a>
### 功能矩陣

| 功能 | Pinecone | Qdrant | Weaviate | Milvus | pgvector |
|---------|----------|--------|----------|--------|----------|
| **語言** | Proprietary | Rust | Go | Go/C++ | C |
| Hosted option | Yes | Yes | Yes | Yes (Zilliz) | Via cloud PG |
| Self-hosted | No | Yes | Yes | Yes | Yes |
| **Serverless** | Yes (Best) | Yes | Yes | Yes (Zilliz) | No |
| **Cloud-Native** | Any | Any | Any | K8s Only | Any |
| Metadata filtering | Good | Excellent | Good | Good | Via SQL |
| **Hybrid search** | Native | Native | Native | Native | Multi-stage (limited) |
| Max vectors | Billions | Billions | Billions | Billions | ~10M |
| HNSW index | Yes | Yes | Yes | Yes | Yes |

---

<a id="metadata-filtering"></a>
## Metadata Filtering

對 multi-tenant 與篩選型使用情境來說，這是關鍵能力。

```python
# Pinecone
results = index.query(
    vector=query_embedding,
    top_k=10,
    filter={"tenant_id": "123", "category": {"$in": ["tech", "science"]}}
)

# Qdrant
results = client.search(
    collection_name="documents",
    query_vector=query_embedding,
    limit=10,
    query_filter=Filter(
        must=[
            FieldCondition(key="tenant_id", match=MatchValue(value="123")),
            FieldCondition(key="category", match=MatchAny(any=["tech", "science"]))
        ]
    )
)
```

**效能影響：**Filtering 發生在搜尋過程中，而不是搜尋之後。預先過濾的索引較快，但彈性較低。

**為什麼 metadata filtering 常是瓶頸：**在樸素向量搜尋中，我們會先找出「Top K」最近鄰，**然後**再依 metadata 過濾。若 filter 很嚴格，最後可能變成 0 筆結果。專用資料庫現在會使用 **Pre-Filtering with HNSW**：在走訪圖的同時，只考慮符合布林 metadata 條件的節點。這需要專用 bitmasks 或硬體加速（SIMD），才能維持低延遲。

**Disk-Native Metadata：**像 **Qdrant** 這樣的現代 DB 會把 metadata 下放到 disk-mapped segments，讓複雜過濾（例如 full-text + geo + vector）不會把 RAM 壓爆。

---

<a id="query-patterns"></a>
## 查詢模式

<a id="pattern-1-simple-semantic-search"></a>
### Pattern 1：簡單語意搜尋

```python
def semantic_search(query: str, top_k: int = 5) -> list[Document]:
    query_embedding = embed(query)
    results = vector_db.search(query_embedding, top_k=top_k)
    return [Document(id=r.id, text=r.payload["text"], score=r.score) for r in results]
```

<a id="pattern-2-filtered-search"></a>
### Pattern 2：條件過濾搜尋

```python
def filtered_search(query: str, filters: dict, top_k: int = 5) -> list[Document]:
    query_embedding = embed(query)
    results = vector_db.search(
        query_embedding,
        top_k=top_k,
        filter=filters  # {"tenant_id": "abc", "created_after": "2025-01-01"}
    )
    return results
```

<a id="pattern-3-hybrid-search-dense-sparse"></a>
### Pattern 3：Hybrid Search（Dense + Sparse）

```python
def hybrid_search(query: str, alpha: float = 0.5, top_k: int = 5) -> list[Document]:
    # Dense (semantic)
    dense_embedding = embed(query)
    dense_results = vector_db.search(dense_embedding, top_k=top_k * 2)

    # Sparse (keyword)
    sparse_results = bm25_search(query, top_k=top_k * 2)

    # Combine with reciprocal rank fusion
    combined = reciprocal_rank_fusion(
        [dense_results, sparse_results],
        weights=[alpha, 1 - alpha]
    )

    return combined[:top_k]
```

有些資料庫（Weaviate、Qdrant、Pinecone）原生支援 hybrid search：

```python
# Weaviate native hybrid
results = client.query.get("Document", ["text"]).with_hybrid(
    query=query,
    alpha=0.5  # 0 = BM25 only, 1 = vector only
).with_limit(5).do()
```

<a id="pattern-4-multi-vector-query"></a>
### Pattern 4：Multi-Vector Query

適用於 parent-child 或 multi-aspect retrieval：

```python
def multi_vector_search(queries: list[str], top_k: int = 5) -> list[Document]:
    all_results = []

    for query in queries:
        embedding = embed(query)
        results = vector_db.search(embedding, top_k=top_k)
        all_results.extend(results)

    # Dedupe and rerank
    unique = dedupe_by_id(all_results)
    reranked = rerank(queries[0], unique)  # Use primary query for reranking

    return reranked[:top_k]
```

---

<a id="production-operations"></a>
## 正式環境營運

<a id="capacity-planning"></a>
### 容量規劃

```python
def estimate_resources(
    num_vectors: int,
    dimensions: int,
    metadata_size_bytes: int = 500
) -> dict:
    # Vector storage
    vector_size = dimensions * 4  # float32
    total_vector_storage = num_vectors * vector_size

    # Index overhead (HNSW ~1.5x)
    index_overhead = total_vector_storage * 1.5

    # Metadata
    metadata_storage = num_vectors * metadata_size_bytes

    # Total
    total_gb = (total_vector_storage + index_overhead + metadata_storage) / 1e9

    # QPS estimate (rough)
    qps_per_gb = 50  # depends heavily on config
    estimated_qps = total_gb * qps_per_gb

    return {
        "storage_gb": total_gb,
        "estimated_qps": estimated_qps,
        "recommended_replicas": max(1, int(total_gb / 50))  # ~50GB per replica
    }
```

<a id="index-maintenance"></a>
### 索引維護

```python
class VectorDBMaintenance:
    def __init__(self, client):
        self.client = client

    def add_documents(self, documents: list[Document]):
        """Upsert documents with batching."""
        batch_size = 100
        for i in range(0, len(documents), batch_size):
            batch = documents[i:i + batch_size]
            embeddings = embed_batch([d.text for d in batch])

            self.client.upsert([
                {
                    "id": doc.id,
                    "vector": embedding,
                    "payload": doc.metadata
                }
                for doc, embedding in zip(batch, embeddings)
            ])

    def delete_documents(self, doc_ids: list[str]):
        """Delete by document ID."""
        self.client.delete(ids=doc_ids)

    def update_metadata(self, doc_id: str, metadata: dict):
        """Update metadata without re-embedding."""
        self.client.set_payload(
            collection_name="documents",
            payload=metadata,
            points=[doc_id]
        )
```

<a id="high-availability"></a>
### 高可用性

```
+-------------------------------------------------------------+
|                    Load Balancer                              |
+----------------------------+--------------------------------+
                             |
            +----------------+----------------+
            v                v                v
     +--------------+ +--------------+ +--------------+
     |  Replica 1   | |  Replica 2   | |  Replica 3   |
     |   (Read)     | |   (Read)     | |   (Primary)  |
     +--------------+ +--------------+ +--------------+
                                             |
                                       (Replication)
                                             |
                                       +-----v-----+
                                       |  Storage   |
                                       +-----------+
```

**關鍵模式：**
- 寫入採 leader-follower
- 以 read replicas 擴展查詢能力
- 以 async replication 達成 HA

<a id="monitoring"></a>
### 監控

```python
VECTOR_DB_METRICS = [
    "query_latency_p50",
    "query_latency_p99",
    "queries_per_second",
    "index_size_gb",
    "vector_count",
    "filter_latency",
    "upsert_latency",
    "cache_hit_rate"
]

def alert_rules():
    return {
        "query_latency_p99_high": {
            "condition": "query_latency_p99 > 500ms",
            "severity": "warning"
        },
        "query_latency_p99_critical": {
            "condition": "query_latency_p99 > 2000ms",
            "severity": "critical"
        },
        "low_recall": {
            "condition": "bench_recall < 0.90",
            "severity": "warning"
        }
    }
```

---

<a id="managed-vs-self-hosted-tco-analysis"></a>
## 託管 vs 自架（TCO 分析）

<a id="cost-comparison"></a>
### 成本比較

| 面向 | Pinecone（Serverless） | 自架（Qdrant/Milvus） |
|--------|-----------------------|-----------------------------|
| **維運負擔** | 幾乎為零 | 高（需要 K8s + SRE） |
| **擴展能力** | 即時（可 scale to zero） | 手動（節點供應） |
| **成本（小規模）** | $0 - $100/月 | $50/月（最低實例） |
| **成本（大規模）** | 每 token/vector 單價高 | 單位成本低 |

<a id="managed-service-pricing-indicative-always-verify-on-provider-pages"></a>
### 託管服務定價（僅供參考，請務必以供應商頁面為準）

| 供應商 | 模式 | 範例：1000 萬向量、1536 維 |
|----------|-------|--------------------------------|
| Pinecone | Pod-based or Serverless | 約 ~$70-150/月 serverless |
| Qdrant Cloud | Per GB | 約 ~$50/月（20GB） |
| Weaviate Cloud | Per dimensions | 約 ~$100/月 |
| Zilliz（Milvus） | Per CU | 約 ~$75/月 |

<a id="self-hosted-costs"></a>
### 自架成本

```python
def estimate_self_hosted_cost(
    vectors: int,
    dimensions: int,
    cloud: str = "aws"
) -> dict:
    storage_gb = (vectors * dimensions * 4 * 2.5) / 1e9  # 2.5x for index

    # Instance sizing
    if storage_gb < 50:
        instance = "r6g.large"  # 16 GB RAM, ~$60/month
    elif storage_gb < 200:
        instance = "r6g.xlarge"  # 32 GB RAM, ~$120/month
    else:
        instance = "r6g.2xlarge"  # 64 GB RAM, ~$240/month

    return {
        "storage_gb": storage_gb,
        "instance": instance,
        "monthly_compute": instance_pricing[instance],
        "monthly_storage": storage_gb * 0.10,  # EBS
        "total_monthly": instance_pricing[instance] + storage_gb * 0.10
    }
```

<a id="decision-managed-vs-self-hosted"></a>
### 決策：託管 vs 自架

| 因素 | 託管 | 自架 |
|--------|---------|-------------|
| 維運負擔 | 低 | 高 |
| 小規模成本 | 較高 | 較低 |
| 大規模成本 | 浮動 | 通常較低 |
| 控制權 | 較少 | 完整 |
| 合規 | 視供應商而定 | 完全掌控 |
| Vendor lock-in | 有 | 無（若用開源） |

**結論**：先從 Serverless 開始。只有在你有超過 5 億向量，或有嚴格的 **On-Prem/GPU-Local** 需求時，才值得自架。

---

<a id="selection-framework"></a>
## 選型框架

<a id="decision-tree"></a>
### 決策樹

```
Need < 100K vectors?
+-- Yes -> pgvector (if already using PostgreSQL)
|          +-- Chroma (for prototyping)
|
+-- No -> Need managed service?
          +-- Yes -> Cloud-first?
          |          +-- Yes -> Pinecone (easiest)
          |          +-- No -> Qdrant Cloud or Zilliz
          |
          +-- No -> Need enterprise features?
                    +-- Yes -> Milvus on Kubernetes
                    +-- No -> Qdrant or Weaviate self-hosted
```

<a id="evaluation-criteria"></a>
### 評估標準

| 標準 | 權重 | 要問的問題 |
|-----------|--------|------------------|
| 規模 | 高 | 現在多少向量？一年後多少？ |
| 延遲 | 高 | p99 要求是多少？ |
| 維運能力 | 高 | 團隊能運營它嗎？ |
| 成本 | 中 | 預算限制？ |
| 功能 | 中 | 是否需要 hybrid search？multimodal？ |
| Lock-in 風險 | 中低 | 是否偏好開源？ |

<a id="proof-of-concept-checklist"></a>
### Proof of Concept 檢查清單

在選定向量資料庫前：

- [ ] 載入具代表性的資料量
- [ ] 在目標 QPS 下 benchmark 查詢延遲
- [ ] 測試 metadata filtering 效能
- [ ] 驗證 update/delete 效能
- [ ] 測試故障復原
- [ ] 評估監控與 observability
- [ ] 計算 total cost of ownership

---

<a id="interview-questions"></a>
## 面試題

<a id="q-how-would-you-choose-between-pinecone-and-a-self-hosted-solution"></a>
### Q：你會如何在 Pinecone 與自架方案之間做選擇？

**強回答：**
決策取決於幾個因素：

**適合選 Pinecone 的情況：**
- 團隊缺乏營運 stateful infrastructure 的能力
- 需要快速上線（幾天而不是幾週）
- 規模中等（低於 1 億向量）
- 預算可接受託管服務溢價
- 合規允許依賴雲端供應商

**適合選自架（Qdrant、Milvus）的情況：**
- 具備 Kubernetes 與維運能力
- 在大規模下對成本很敏感
- 需要完整掌控資料
- 有特定合規要求
- 想避免 vendor lock-in

對大多數新創來說，我會先用 Pinecone 或 Qdrant Cloud 追求速度，若未來在規模上成本變得難以接受，再評估遷移。切換成本屬中等，因為多數 vector DB 的 API 都相似。

<a id="q-explain-how-hnsw-works-and-when-you-would-not-use-it"></a>
### Q：請解釋 HNSW 的運作方式，以及你什麼情況下不會使用它。

**強回答：**
HNSW 會建立一個多層次的向量圖：

**運作方式：**
1. 將向量作為節點插入多層圖中
2. 越高層節點越少，但跳躍距離越大
3. 搜尋時先從最高層開始，greedily 找最近鄰
4. 一路往下走到最底層（包含所有向量）

**它的優點：**
- O(log n) 查詢複雜度
- 不需訓練
- 支援即時更新
- recall/latency 表現非常均衡

**不適合使用的情況：**
- 資料集很小（<10K）：brute force 就夠了
- 記憶體非常受限：HNSW 的圖結構需要 1.5-2 倍向量大小
- 需要 exact search：HNSW 是近似搜尋
- 更新量很大且延遲要求嚴格：更新可能造成暫時性退化

替代方案：
- 記憶體受限時用 IVF-PQ
- 十億級且追求成本效率時用 DiskANN
- 要 exact search 時用 Flat index
- 超高維 sparse vectors 可考慮 LSH

<a id="q-when-would-you-use-a-disk-based-index-like-diskann-over-a-ram-based-index-hnsw"></a>
### Q：什麼情況下你會用磁碟型索引（如 DiskANN），而不是記憶體型索引（HNSW）？

**強回答：**
當索引的記憶體成本超出預算，或超出單一高記憶體節點容量時，我就會選擇磁碟型索引。例如，1536 維的 1 億向量若採 HNSW，幾乎需要 1TB RAM。使用 DiskANN 後，我可以把那 1TB 的大部分資料放在 NVMe SSD 上，將 RAM 需求降低 90-95%，同時維持 sub-10ms query time。對非即時搜尋場景而言，這代表極大的 TCO（Total Cost of Ownership）下降。

<a id="q-why-is-metadata-filtering-often-the-bottleneck-in-vector-databases"></a>
### Q：為什麼 metadata filtering 常常是向量資料庫的瓶頸？

**強回答：**
在樸素向量搜尋中，我們會先找出「Top K」最近鄰，**然後**再依 metadata 過濾（例如「只要 2024 年的文件」）。若 filter 很嚴格，結果可能在過濾後變成 0。專用資料庫現在會用 **Pre-Filtering with HNSW**，在走訪圖時就只考慮符合布林 metadata 條件的節點。這在計算上很昂貴，因為它破壞了 HNSW 原有的「short-circuit」邏輯，需要仰賴專門的 bitmasks 或硬體加速（SIMD）才能維持低延遲。

<a id="q-how-do-you-handle-multi-tenancy-in-a-vector-database"></a>
### Q：你如何在向量資料庫中處理 multi-tenancy？

**強回答：**
主要有三種方法：

**1. Metadata filtering（最常見）：**
```python
results = db.search(
    vector=query,
    filter={"tenant_id": current_tenant}
)
```
- 優點：簡單、單一索引
- 缺點：所有 tenant 共享資源，若有 bug 可能暴露資料

**2. 每個 tenant 一個 collection：**
```python
results = db.collection(f"tenant_{tenant_id}").search(vector=query)
```
- 優點：隔離性強，可按 tenant 擴展
- 缺點：collection 太多，維運負擔高

**3. 每個 tenant 一個 namespace（Pinecone）：**
```python
results = index.query(vector=query, namespace=tenant_id)
```
- 優點：在單一索引內提供隔離
- 缺點：供應商特定

**我會這樣選：**
- 大多數情況用 metadata filtering（簡單、成本效益高）
- 高安全需求用獨立 collections
- 絕不做 post-filter（先取全部再過濾），避免資料洩漏風險

---

<a id="references"></a>
## 參考資料

- Malkov and Yashunin.「Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs」（HNSW, 2018）
- Microsoft Research.「Vamana/DiskANN: A Disk-based Index for ANN Search」（2019/2023）
- Pinecone Documentation: https://docs.pinecone.io/
- Pinecone.「The Managed Architecture of Serverless Vector DBs」（2024）
- Qdrant Documentation: https://qdrant.tech/documentation/
- Weaviate Documentation: https://weaviate.io/developers/weaviate
- Milvus Documentation: https://milvus.io/docs
- pgvector: https://github.com/pgvector/pgvector

---

*上一篇：[嵌入模型](03-embedding-models.md) | 下一篇：[混合搜尋](05-hybrid-search.md)*

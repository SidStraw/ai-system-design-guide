<a id="late-interaction--colbert"></a>
# Late Interaction 與 ColBERT

Late Interaction 是一種介於快速但不夠精準的 **bi-encoder** 與精準但較慢的 **cross-encoder** 之間的檢索範式。ColBERT（Contextualized Late Interaction over BERT）是這個領域的代表模型，能以接近 bi-encoder 的速度，提供接近 cross-encoder 的準確度。Late-interaction 家族如今已成熟為可用於正式環境的高精度搜尋方案，而其多模態延伸（ColPali、ColQwen2.5、ColNomic，以及 Wholembed v3 這類統一式 retriever）也已成為同一套工具鏈的一部分。

<a id="table-of-contents"></a>
## 目錄

- [檢索架構光譜](#spectrum)
- [ColBERT 架構](#colbert-architecture)
- [MaxSim：核心評分機制](#maxsim)
- [ColBERTv2 與 PLAID 索引](#colbertv2)
- [Late Interaction 與其他方案比較](#comparison)
- [使用 RAGatouille 實作](#ragatouille)
- [正式環境部署模式](#production)
- [何時該選擇 ColBERT](#when-to-choose)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-retrieval-architecture-spectrum"></a>
## 檢索架構光譜

神經檢索主要有三種基本架構。理解 late interaction 位於哪個位置，是掌握本章內容的關鍵。

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   SPEED ◄──────────────────────────────────────────────► ACCURACY   │
│                                                                     │
│   Bi-Encoder          Late Interaction          Cross-Encoder       │
│   (Single Vector)     (Multi-Vector)            (Full Attention)    │
│                                                                     │
│   ● Fast (< 10ms)     ● Balanced (10-50ms)      ● Slow (100ms+)   │
│   ● Low accuracy       ● High accuracy           ● Highest accuracy│
│   ● Scales to 1B+     ● Scales to 100M+         ● Scales to 10K   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

<a id="how-each-architecture-processes-a-query-document-pair"></a>
### 各種架構如何處理查詢—文件配對

```
BI-ENCODER (e.g., E5, BGE):
  Query  ──► Encoder ──► [1 vector]  ─┐
                                      ├──► dot product ──► score
  Doc    ──► Encoder ──► [1 vector]  ─┘

  Total interaction: 1 comparison

─────────────────────────────────────────────

LATE INTERACTION (ColBERT):
  Query  ──► Encoder ──► [N vectors] ─┐
                (one per token)       ├──► MaxSim ──► score
  Doc    ──► Encoder ──► [M vectors] ─┘
                (one per token)

  Total interaction: N x M comparisons (but decomposable)

─────────────────────────────────────────────

CROSS-ENCODER (e.g., ms-marco-MiniLM):
  [Query + Doc] ──► Encoder ──► score

  Total interaction: Full self-attention across
                     all query AND document tokens
```

**洞見**：關鍵差異在於查詢與文件「何時」互動。Bi-encoder 完全不互動（獨立編碼）；cross-encoder 完全互動（聯合編碼）；late interaction 則是折衷方案：先獨立編碼，再以 token 層級做低成本互動。

---

<a id="colbert-architecture"></a>
## ColBERT 架構

ColBERT 會把查詢與文件編碼成 **token 層級嵌入的矩陣**（而不是單一向量），並透過細粒度的 token 互動來評分。

<a id="encoding-phase"></a>
### 編碼階段

```
Query: "What is the price of Widget-X?"

Token Embeddings (each 128-dim):
  q1 = Embed("What")     = [0.12, -0.34, ..., 0.08]
  q2 = Embed("is")       = [0.05, -0.11, ..., 0.22]
  q3 = Embed("the")      = [0.01, -0.02, ..., 0.15]
  q4 = Embed("price")    = [0.45,  0.67, ..., 0.91]  ◄── high signal
  q5 = Embed("of")       = [0.03, -0.05, ..., 0.11]
  q6 = Embed("Widget-X") = [0.88,  0.21, ..., 0.73]  ◄── high signal

Document: "Widget-X costs $200 per month for the Standard plan"

Token Embeddings:
  d1 = Embed("Widget-X")  = [0.85,  0.19, ..., 0.71]
  d2 = Embed("costs")     = [0.42,  0.63, ..., 0.88]
  d3 = Embed("$200")      = [0.31,  0.55, ..., 0.79]
  d4 = Embed("per")       = [0.02, -0.01, ..., 0.09]
  d5 = Embed("month")     = [0.11,  0.08, ..., 0.14]
  d6 = Embed("Standard")  = [0.38,  0.44, ..., 0.62]
  d7 = Embed("plan")      = [0.29,  0.37, ..., 0.51]
```

**關鍵設計選擇**：ColBERT 使用 **128 維** token embedding（相較於標準 bi-encoder 常見的 768–1024 維）。由於我們儲存的是每份文件的 N 個向量而不是 1 個，這個較小的維度對儲存效率至關重要。

<a id="offline-vs-online-computation"></a>
### 離線與線上計算

| 元件 | 時機 | 成本 |
|-----------|------|------|
| 文件編碼 | 離線（建索引時） | 一次性，可平行化 |
| 查詢編碼 | 線上（每次查詢） | 快速（GPU 約 5–10ms） |
| MaxSim 評分 | 線上（每次查詢） | token 層級運算，由 PLAID 最佳化 |

**這種拆解正是 ColBERT 快的原因**：文件只需預先編碼一次。查詢時只要編碼 query，而評分只是對預先計算好的向量做簡單算術運算。

---

<a id="maxsim-the-core-scoring-mechanism"></a>
## MaxSim：核心評分機制

MaxSim（Maximum Similarity）是讓 late interaction 得以運作的運算子。概念上很簡單，但威力出乎意料地強大。

<a id="how-maxsim-works"></a>
### MaxSim 如何運作

```
For each query token qi:
  1. Compute dot product with EVERY document token dj
  2. Keep only the MAXIMUM score

Score(Q, D) = SUM over all qi of MAX over all dj of (qi . dj)
```

<a id="worked-example"></a>
### 範例演算

```
            d1        d2       d3       d4       d5
          Widget-X   costs    $200     per     month
  q4       0.41      0.89*    0.73     0.01     0.05
  price
  q6       0.95*     0.38     0.27     0.01     0.03
  Widget-X

  * = maximum for that query token

  MaxSim contribution from q4 ("price"): 0.89 (matched "costs")
  MaxSim contribution from q6 ("Widget-X"): 0.95 (matched "Widget-X")

  Total Score = sum of all max values across all query tokens
```

<a id="why-maxsim-outperforms-single-vector-similarity"></a>
### 為什麼 MaxSim 優於單向量相似度

| 屬性 | 單向量（Dot Product） | MaxSim（Late Interaction） |
|----------|----------------------------|--------------------------|
| **粒度** | 文件層級 | token 層級 |
| **部分匹配** | 全有或全無 | 各 token 可獨立匹配 |
| **詞項重要性** | 壓縮進 1 個向量 | 每個 token 各自貢獻 |
| **罕見詞** | 會被平均稀釋 | 以獨立向量保留 |

**直覺**：在 bi-encoder 中，「Widget-X」的語意會和「costs」、「$200」以及其他所有 token 平均進同一個向量。若「Widget-X」很罕見，它的訊號就會被稀釋。而在 ColBERT 中，「Widget-X」保有自己的專屬向量，因此 MaxSim 可以獨立為它找到強匹配。

---

<a id="colbertv2-and-plaid-indexing"></a>
## ColBERTv2 與 PLAID 索引

原始 ColBERT（2020）有一個關鍵限制：**儲存成本**。為每一份文件中的每個 token 都儲存 128 維向量非常昂貴。若語料庫有 1000 萬份文件、每份 200 個 token，向量儲存需求大約會是 ~256 GB。

<a id="colbertv2-improvements-2021"></a>
### ColBERTv2 的改進（2021）

ColBERTv2 帶來兩項重要創新：

**1. Residual Compression**：

```
Original ColBERT:
  Each token vector: 128 dims x 32-bit float = 512 bytes

ColBERTv2 Residual Compression:
  1. Cluster all token vectors into centroids (k-means)
  2. Store only the centroid ID + residual (difference)
  3. Quantize the residual to 1-2 bits per dimension

  Each token vector: ~16-32 bytes (16-32x compression)
```

**2. Denoised Supervision**：
- 以 cross-encoder teacher 挖出的 hard negatives 進行訓練
- 用 cross-encoder 標籤「清理」雜訊訓練資料
- 結果：即使經過壓縮，embedding 品質仍更好

**ColBERTv2 儲存比較**：

| 系統 | 每個 token 的儲存量 | 1000 萬文件（每份 200 token） |
|--------|------------------|---------------------------|
| ColBERT v1 | 512 bytes | ~1 TB |
| ColBERTv2（壓縮） | 32 bytes | ~64 GB |
| Bi-encoder（每文件 1 向量） | 3 KB | ~30 GB |

<a id="plaid-the-indexing-engine"></a>
### PLAID：索引引擎

PLAID（Performance-optimized Late Interaction Driver）是讓 ColBERT 能在大規模場景中實用化的索引與檢索引擎。

```
┌─────────────────────────────────────────────────────────────────┐
│                    PLAID RETRIEVAL PIPELINE                     │
│                                                                 │
│  Stage 1: CENTROID PRUNING                                      │
│  ─────────────────────────                                      │
│  For each query token, find nearest centroids                   │
│  Collect candidate passages that contain those centroids        │
│  Result: ~10,000 candidates from millions                       │
│                                                                 │
│  Stage 2: CENTROID INTERACTION                                  │
│  ─────────────────────────────                                  │
│  Approximate MaxSim using centroid-level scores only            │
│  Filter candidates to top ~1,000                                │
│                                                                 │
│  Stage 3: CENTROID PRUNING (Fine)                               │
│  ──────────────────────────────                                 │
│  Decompress residuals for remaining candidates                  │
│  Compute approximate MaxSim with residual vectors               │
│  Filter to top ~100                                             │
│                                                                 │
│  Stage 4: FULL DECOMPRESSION                                    │
│  ────────────────────────────                                   │
│  Fully decompress token vectors for top candidates              │
│  Compute exact MaxSim                                           │
│  Return final ranked results                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**關鍵洞見**：PLAID 不會為所有文件解壓所有向量。每一個階段都以低成本縮小候選集，因此昂貴的精確評分只會發生在整個語料中極小的一部分上。

**PLAID 效能**：
- 在單張 GPU 上，能於 **50–100ms** 內自 **1000 萬+ 文件** 檢索
- 維持 **精確 MaxSim** 的準確度（不是近似結果）
- 透過 centroid pruning，在完整評分前就略過 99%+ 的語料

---

<a id="late-interaction-vs-alternatives"></a>
## Late Interaction 與其他方案比較

<a id="comprehensive-comparison"></a>
### 全面比較

| 維度 | BM25 | Bi-Encoder | ColBERT（Late） | Cross-Encoder |
|-----------|------|-----------|----------------|---------------|
| **編碼方式** | 詞頻 | 1 向量／文件 | N 向量／文件 | 聯合編碼（不可預先計算） |
| **查詢延遲** | ~5ms | ~10ms | ~30–50ms | 每對 ~500ms+ |
| **可擴展性** | 十億級 | 十億級 | 100M+ | ~10K（僅 reranking） |
| **儲存量（100 萬文件）** | ~2 GB | ~3 GB | ~6–12 GB | 0（無索引） |
| **準確率（NDCG@10）** | 0.30–0.35 | 0.35–0.40 | 0.39–0.44 | 0.42–0.46 |
| **跨領域轉移** | 強（詞彙式） | 弱（需微調） | 強（token 層級） | 最強 |
| **建置複雜度** | 低 | 中 | 高 | 低（無索引） |

<a id="when-colbert-wins"></a>
### ColBERT 何時勝出

```
                  ▲ Accuracy
                  │
             0.45 ┤                     ● Cross-Encoder
                  │                   ●
             0.40 ┤              ● ColBERT
                  │         ●
             0.35 ┤    ● Bi-Encoder
                  │ ●
             0.30 ┤ BM25
                  │
                  └────┬────┬────┬────┬────┬──► Throughput (QPS)
                      10   100  1K   10K  100K
```

**ColBERT 佔據了最佳甜蜜點**：在特定領域基準測試上，它比 bi-encoder 準確 **3–5 倍**（在專門資料集上 mAP 最多可提升 +13.8%），同時又比 cross-encoder 快 **10–50 倍**。

---

<a id="implementation-with-ragatouille"></a>
## 使用 RAGatouille 實作

RAGatouille（由 Answer.AI 推出）是將 ColBERT 用於 RAG pipeline 的標準 Python 函式庫。它以簡單、高階的 API 包裝 Stanford ColBERT 程式庫。

<a id="basic-usage"></a>
### 基本用法

```python
from ragatouille import RAGPretrainedModel

# Load a pretrained ColBERT model
RAG = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")

# Index documents (one-time, creates PLAID index on disk)
documents = [
    "Widget-X costs $200 per month for the Standard plan.",
    "The Enterprise plan includes SSO and audit logs for $800/month.",
    "All plans include 99.9% uptime SLA and 24/7 email support.",
    "Widget-X was launched in 2023 and serves 10,000+ customers.",
]

index_path = RAG.index(
    index_name="products",
    collection=documents,
    split_documents=True  # auto-chunk long docs
)

# Search the index
results = RAG.search(
    query="How much does Widget-X cost?",
    k=3
)

for result in results:
    print(f"Score: {result['score']:.4f}")
    print(f"Text:  {result['content']}\n")
```

<a id="integration-with-langchain"></a>
### 與 LangChain 整合

```python
from ragatouille import RAGPretrainedModel
from langchain_core.runnables import RunnablePassthrough
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

# Create ColBERT retriever
RAG = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")
retriever = RAG.as_langchain_retriever(k=5)

# Build RAG chain
template = """Answer based on the following context:
{context}

Question: {question}"""

prompt = ChatPromptTemplate.from_template(template)
llm = ChatOpenAI(model="gpt-4o")

chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | llm
)

response = chain.invoke("What features does the Enterprise plan include?")
```

<a id="other-colbert-libraries-and-integrations"></a>
### 其他 ColBERT 函式庫與整合方案

| 函式庫 | 使用情境 | 說明 |
|---------|----------|-------|
| **RAGatouille** | Python 優先、API 簡單 | 最適合原型與中小規模場景 |
| **colbert-ai**（Stanford） | 研究、完整控制 | 較底層、可設定項較多 |
| **Vespa** | 正式環境大規模部署 | 託管式基礎設施，原生支援 ColBERT |
| **PyLate** | 彈性訓練／微調 | 建於 Sentence Transformers 之上，適合自訂模型 |
| **Jina ColBERT v2** | 多語言（89 種語言） | 輸出維度彈性、可直接上正式環境 |

---

<a id="production-deployment-patterns"></a>
## 正式環境部署模式

<a id="pattern-1-colbert-as-primary-retriever"></a>
### 模式 1：以 ColBERT 作為主檢索器

```
Query ──► ColBERT (PLAID) ──► Top 20 ──► LLM
```

最適合：中型規模語料（100 萬–5000 萬文件），而且準確度是首要目標，並且能接受額外的儲存成本。

<a id="pattern-2-colbert-as-reranker-most-common"></a>
### 模式 2：以 ColBERT 作為 reranker（最常見）

```
Query ──► BM25 or Bi-Encoder ──► Top 1000 ──► ColBERT Rerank ──► Top 20 ──► LLM
```

最適合：大規模系統中第一階段檢索必須很便宜，但你又需要高品質 reranking，且不想承擔 cross-encoder 的成本。

```
┌─────────────────────────────────────────────────────────────────┐
│              COLBERT-AS-RERANKER ARCHITECTURE                   │
│                                                                 │
│  User Query                                                     │
│      │                                                          │
│      ▼                                                          │
│  First Stage: BM25 / Bi-Encoder                                 │
│  (cheap, high recall, Top 1000)                                 │
│      │                                                          │
│      ▼                                                          │
│  Second Stage: ColBERT MaxSim Reranking                         │
│  (pre-computed doc tokens, score Top 1000)                      │
│  Cost: only query encoding + MaxSim arithmetic                  │
│      │                                                          │
│      ▼                                                          │
│  Top 20 Passages ──► LLM Generation                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

<a id="pattern-3-hybrid-colbert--bm25--dense"></a>
### 模式 3：混合式（ColBERT + BM25 + Dense）

```
Query ──┬──► BM25 (Top 50) ────────┐
        ├──► Dense Bi-Encoder (50) ─┼──► RRF ──► ColBERT Rerank ──► Top 10
        └──► ColBERT (Top 50) ─────┘
```

最適合：在中等規模下追求最高準確度。成本較高，但能覆蓋所有檢索模態。

<a id="storage-and-infrastructure-considerations"></a>
### 儲存與基礎設施考量

| 語料規模 | Bi-Encoder 儲存量 | ColBERT 儲存量 | GPU 需求 |
|------------|-------------------|-----------------|-----------------|
| 100K 文件 | ~300 MB | ~600 MB - 1.2 GB | 純 CPU 亦可 |
| 1M 文件 | ~3 GB | ~6–12 GB | 建議 1 張 GPU |
| 10M 文件 | ~30 GB | ~60–120 GB | 需要 1–2 張 GPU |
| 100M 文件 | ~300 GB | ~600 GB - 1.2 TB | 多 GPU／分散式 |

**現實檢查**：ColBERT 的儲存量通常是 bi-encoder 的 2–4 倍。對大多數 RAG 使用情境（1000 萬文件以下）而言仍可接受；但對網頁級搜尋（數十億頁）來說，bi-encoder 或 learned sparse methods 在第一階段檢索仍更實際。

---

<a id="when-to-choose-colbert"></a>
## 何時該選擇 ColBERT

<a id="decision-framework"></a>
### 決策框架

```
Is your corpus < 100M documents?
├── No  ──► Use Bi-Encoder for retrieval + ColBERT for reranking
└── Yes
    │
    Is accuracy more important than infrastructure simplicity?
    ├── No  ──► Use Bi-Encoder (simpler, cheaper)
    └── Yes
        │
        Can you afford 2-4x storage vs. bi-encoder?
        ├── No  ──► Use Bi-Encoder + Cross-Encoder reranker
        └── Yes ──► Use ColBERT (PLAID) as primary retriever
```

<a id="colbert-vs-dense-retrieval-vs-hybrid-search"></a>
### ColBERT vs. Dense Retrieval vs. Hybrid Search

| 情境 | 最佳選擇 | 原因 |
|----------|-------------|-----|
| 通用型 RAG（< 100 萬文件） | 混合式（Dense + BM25） | 最簡單，準確度也夠用 |
| 特定領域搜尋（法律、醫療） | ColBERT | token 層級匹配可保留專業術語 |
| 多語言語料 | Jina ColBERT v2 | 原生支援 89 種語言 |
| 成本敏感、高流量 | Bi-Encoder + BM25 | 儲存與計算成本最低 |
| 中等規模下追求最高準確度 | ColBERT + Reranker | 不用承擔 cross-encoder 延遲即可達到最佳品質 |
| 網頁級（10 億+ 文件） | 第一階段用 Bi-Encoder，之後 ColBERT rerank | ColBERT 索引太大，不適合做主檢索 |

---

<a id="interview-questions"></a>
## 面試題

<a id="q-explain-the-difference-between-bi-encoders-cross-encoders-and-late-interaction-models-when-would-you-choose-each"></a>
### Q：說明 bi-encoder、cross-encoder 與 late interaction model 的差異。你會在什麼情況下選擇各自方案？

**強答範例：**
這三種架構的差異在於查詢與文件「何時」互動：

**Bi-encoder** 會把 query 與 document 各自獨立編碼為單一向量。互動只在最後以 dot product 發生。這種方法很快（可預先計算所有文件向量、在毫秒級搜尋），但會失去細粒度匹配能力——整份文件的語意被壓縮成向量空間中的一個點。

**Cross-encoder** 會把串接後的 query + document 一起送進單一 transformer。完整 self-attention 代表每個 query token 都能關注每個 document token。這能提供最高準確度，但無法預先計算任何內容——每個 query-document pair 都需要一次完整 forward pass，因此不適合做第一階段檢索。Cross-encoder 通常用於前 10–100 個候選結果的 reranking。

**Late interaction（ColBERT）** 則像 bi-encoder 一樣獨立編碼 query 與 document，但產出的是 *每個 token 一個向量* 的矩陣，而不是單一向量。評分採用 MaxSim——對每個 query token，找出最匹配的 document token。這保留了 token 層級粒度，同時仍允許文件預先計算。因此它能以接近 bi-encoder 的速度，達到接近 cross-encoder 的準確度。

如果是重視簡潔、需大規模第一階段檢索，我會選 bi-encoder；若是高風險場景下對小候選集合做高精度 reranking，我會選 cross-encoder；若我需要 cross-encoder 的準確度但負擔不起其延遲，尤其是在法律、醫療、技術文件這類詞項級匹配很重要的領域搜尋，我會選 ColBERT。

<a id="q-colbert-stores-one-vector-per-token-how-does-it-scale-and-what-are-the-storage-tradeoffs"></a>
### Q：ColBERT 為每個 token 儲存一個向量。它如何擴展？儲存上的取捨是什麼？

**強答範例：**
ColBERT 的原始儲存成本確實不低。一份 200-token 的文件，需要 200 個 128 維向量；相較之下，bi-encoder 只需 1 個 768–1024 維向量。因此每份文件大約會多出 3–5 倍的儲存成本。

ColBERTv2 透過 **residual compression** 改善這點：先把 token 向量分群成 centroids，然後只儲存 centroid ID 與量化後的 residual。這能讓每個 token 向量得到 16–32 倍壓縮，將實際儲存量降到約 bi-encoder 的 2–4 倍。

PLAID 索引引擎則進一步提升查詢時效率，透過多階段 pipeline 運作。它先用快速且粗略的 centroid pruning 去除 99% 候選，再只為有希望的候選逐步解壓 residual。最後只在少於 100 份文件上計算精確 MaxSim，因此即便是 1000 萬+ 文件語料，延遲仍可控制在 50–100ms。

若規模超過 1 億份文件，我會把 ColBERT 用作 reranker，而不是主檢索器——先讓 bi-encoder 或 BM25 做第一階段檢索，把候選縮到 1000 份文件，再用 ColBERT 的 MaxSim 做高品質 reranking。

<a id="q-you-are-designing-a-legal-document-search-system-with-5m-documents-the-team-is-debating-between-dense-bi-encoder-search-with-a-cross-encoder-reranker-vs-colbert-what-do-you-recommend"></a>
### Q：你正在設計一個有 500 萬份文件的法律文件搜尋系統。團隊正在猶豫要用 dense bi-encoder search 搭配 cross-encoder reranker，還是使用 ColBERT。你會怎麼建議？

**強答範例：**
我會推薦這個場景使用 ColBERT，原因有三：

第一，**法律文本對詞項非常敏感**。合約條款會引用特定章節編號、定義術語（例如「Force Majeure」）與精確片語。ColBERT 的 token 層級 MaxSim 匹配能保留這些稀少但關鍵的詞，而這些詞在單向量的 bi-encoder embedding 中容易被稀釋。

第二，**500 萬份文件正好落在 ColBERT 的甜蜜點**。使用 ColBERTv2 壓縮後，索引大約會落在 30–60 GB——單張 GPU 即可容納。這已小到足以作為主檢索器，無需再建立額外的第一階段檢索器。

第三，**cross-encoder reranking 會帶來額外延遲**。每一組 query-document pair 都需要一次完整 transformer forward pass。若對 100 個候選做 reranking，可能要 500ms–2s。ColBERT 能維持相近準確度，同時把總延遲控制在 100ms 內，因為 document token 都已預先計算。

我唯一會補強 ColBERT 的地方，是並行加上一個 BM25 索引用來處理精確比對查詢（如法條編號、判例引用），因為這類查詢需要關鍵字精準度。我會在送給 LLM 前用 RRF 合併 ColBERT 與 BM25 的結果。

---

<a id="references"></a>
## 參考資料
- Khattab & Zaharia. "ColBERT: Efficient and Effective Passage Search" (SIGIR 2020)
- Santhanam et al. "ColBERTv2: Effective and Efficient Retrieval via Lightweight Late Interaction" (NAACL 2022)
- Santhanam et al. "PLAID: An Efficient Engine for Late Interaction Retrieval" (CIKM 2022)
- Answer.AI. "RAGatouille: State-of-the-art Late Interaction Retrieval" (GitHub, 2024)
- Jina AI. "Jina-ColBERT-v2: General-Purpose Multilingual Late Interaction Retriever" (2024)
- Weaviate. "An Overview of Late Interaction Retrieval Models" (2025)
- ECIR 2026. "Late Interaction Workshop" (2026)

---

*上一篇：[Contextual Retrieval](10-contextual-retrieval.md)*

<a id="embeddings-and-vector-spaces"></a>
# 嵌入與向量空間

嵌入（Embeddings）是文字的稠密向量表示，能捕捉語意含義。它們是 RAG 系統、semantic search，以及許多 AI 應用的基礎。

<a id="table-of-contents"></a>
## 目錄

- [什麼是嵌入](#what-are-embeddings)
- [嵌入模型架構](#embedding-model-architectures)
- [訓練目標](#training-objectives)
- [距離度量](#distance-metrics)
- [嵌入模型比較](#embedding-model-comparison)
- [Matryoshka 與自適應維度](#matryoshka-and-adaptive-dimensions)
- [Late Interaction 與 Late Chunking](#late-chunking-and-interaction)
- [Binary 與 Scalar Quantization](#quantization-for-scale)
- [實務考量（Batching、Caching）](#practical-considerations)
- [嵌入漂移與版本管理](#embedding-drift-and-versioning)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="what-are-embeddings"></a>
## 什麼是嵌入

嵌入會將離散文字（單字、句子、文件）映射到連續向量空間中，讓語意相似度對應到幾何上的接近程度。

**關鍵特性：**
- 相似含義會彼此靠近
- 關係可用向量運算編碼（king - man + woman = queen）
- 可透過 approximate nearest neighbor 演算法高效率地做相似度搜尋

**心智模型：**
可以把嵌入想成位於極高維空間中的座標。維度數（512 到 4096）提供表達能力。每個維度都捕捉某種意義面向，但單一維度本身通常不可解釋。

---

<a id="embedding-model-architectures"></a>
## 嵌入模型架構

<a id="word-embeddings-historical"></a>
### 詞嵌入（歷史）

早期方法會為單一詞彙建立嵌入：

| 模型 | 年份 | 方法 | 限制 |
|-------|------|----------|------------|
| Word2Vec | 2013 | Skip-gram, CBOW | 靜態：「bank」在所有語境下都相同 |
| GloVe | 2014 | Co-occurrence matrix | 靜態 |
| FastText | 2017 | Subword embeddings | 靜態，但能處理 OOV |

**主要限制：** 無論語境如何，同一個詞都會得到相同的嵌入。

<a id="contextual-embeddings"></a>
### 上下文嵌入

以 Transformer 為基礎的模型會產生依語境而變的嵌入：

```python
# Static embedding (Word2Vec)
embed("bank") = [0.1, 0.3, ...]  # Same vector always

# Contextual embedding (BERT)
embed("river bank") = [0.1, 0.3, ...]   # Geography sense
embed("bank account") = [0.5, 0.2, ...]  # Finance sense
```

<a id="sentencedocument-embeddings"></a>
### 句子／文件嵌入

對檢索而言，我們需要為整段文字建立嵌入：

| 方法 | 作法 | 優點 | 缺點 |
|----------|--------|------|------|
| Mean pooling | 對 token embeddings 取平均 | 簡單 | 會遺失資訊 |
| CLS token | 使用 [CLS] token embedding | BERT 的標準做法 | 可能無法捕捉全文 |
| Last token | 使用最後一個 token | 適合 decoder 模型 | 位置偏差 |
| Trained pooling | 學習 pooling 權重 | 品質較好 | 需要訓練 |

現代嵌入模型通常會專門為句子／文件嵌入而訓練，而不是只是從 language model 改造而來。

<a id="bi-encoder-architecture"></a>
### Bi-Encoder 架構

標準的檢索嵌入架構：

```
Document -> Encoder -> Document Embedding
Query    -> Encoder -> Query Embedding

Similarity = cosine(doc_embedding, query_embedding)
```

**特性：**
- 文件嵌入可預先計算並建立索引
- Query embedding 在查詢時才計算
- 每份文件的相似度計算是 O(1)（搭配 ANN）

<a id="cross-encoder-architecture"></a>
### Cross-Encoder 架構

另一種做法是將 query 與文件一起送入模型處理：

```
[Query, Document] -> Encoder -> Relevance Score
```

**特性：**
- 更準確（可同時看到兩者）
- 無法預先計算：對 n 份文件推論成本為 O(n)
- 用於 reranking，而非 retrieval

---

<a id="training-objectives"></a>
## 訓練目標

<a id="contrastive-learning"></a>
### 對比式學習

多數現代嵌入模型都使用對比式學習：

```python
# Simplified contrastive loss
def contrastive_loss(anchor, positive, negatives):
    pos_sim = cosine_similarity(anchor, positive)
    neg_sims = [cosine_similarity(anchor, neg) for neg in negatives]
    
    # Push positive close, negatives far
    loss = -log(exp(pos_sim / tau) / 
                (exp(pos_sim / tau) + sum(exp(neg_sim / tau) for neg_sim in neg_sims)))
    return loss
```

**關鍵因素：**
- **正樣本對：** 語意相近的文字（平行句、query-document 配對）
- **困難負樣本：** 相似但不匹配的文字（由 BM25 找回但不相關）
- **Batch 內負樣本：** 將同一個 batch 內其他項目當成負樣本（高效率）

<a id="training-data-sources"></a>
### 訓練資料來源

| 來源 | 正樣本對 | 品質 | 規模 |
|--------|---------------|---------|-------|
| 平行句 | 翻譯配對 | 高 | 中 |
| Query-document | 搜尋日誌 | 高 | 中 |
| Title-body | 文件結構 | 中 | 大 |
| Paraphrase | NLI 資料集 | 高 | 小 |
| Generated | LLM 產生配對 | 不一定 | 大 |

<a id="instruction-tuned-embeddings"></a>
### 指令微調嵌入

近期模型可以接受任務指令：

```python
# Instruction-tuned (e.g., E5, BGE)
query_embedding = embed("Represent this query for retrieval: What is RAG?")
doc_embedding = embed("Represent this document for retrieval: RAG combines...")
```

這能透過明確指定用途來提升效果。

---

<a id="distance-metrics"></a>
## 距離度量

<a id="cosine-similarity"></a>
### Cosine Similarity

最常見於文字嵌入：

```python
def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```

**特性：**
- 範圍：[-1, 1]（對正規化向量而言，若皆為正則是 [0, 1]）
- 衡量的是夾角，不是大小
- 不受向量長度影響

**適用時機：** 文字嵌入的預設選擇。

<a id="dot-product"></a>
### Dot Product

```python
def dot_product(a, b):
    return np.dot(a, b)
```

**特性：**
- 大小會影響結果
- 範圍無界
- 對已正規化向量來說，等同於 cosine

**適用時機：** 當嵌入已正規化，或大小本身具有意義時。

<a id="euclidean-distance"></a>
### Euclidean Distance

```python
def euclidean_distance(a, b):
    return np.linalg.norm(a - b)
```

**特性：**
- 衡量絕對差異
- 會受到大小影響
- 對正規化向量而言：sqrt(2 - 2 * cosine)

**適用時機：** 很少用於文字；更常見於影像嵌入。

<a id="metric-selection"></a>
### 度量選擇

| 度量 | 向量資料庫 | 常見用途 |
|--------|------------------|------------|
| Cosine | Pinecone, Qdrant, Weaviate | 文字嵌入 |
| Dot Product | 所有主流 DB | 正規化嵌入 |
| Euclidean | 所有主流 DB | 影像、多模態 |

---

<a id="embedding-model-comparison"></a>
## 嵌入模型比較

<a id="current-top-models-december-2025"></a>
### 目前頂尖模型（2025 年 12 月）

| 模型 | 維度 | 最大 Tokens | MTEB Retrieval | 成本 / 100 萬 tokens |
|-------|------------|------------|----------------|------------------|
| OpenAI text-embedding-4 | 3072 | 16k | 68.2 | $0.10 |
| Voyage-4 | 1024 | 128k | 70.1 | $0.05 |
| Cohere embed-v3.5 | 1024 | 512 | 67.5 | $0.10 |
| Google text-embedding-005 | 768 | 8k | 67.2 | $0.02 |

*MTEB 分數代表 2025 年底前沿模型的大致水準。*

*MTEB 分數為近似值，且會隨 benchmark 子集而變動。請務必查證最新數值。*

<a id="open-source-models"></a>
### 開源模型

| 模型 | 維度 | 最大 Tokens | MTEB Retrieval | 備註 |
|-------|------------|------------|----------------|-------|
| BGE-large-en-v1.5 | 1024 | 512 | 63.9 | 強勢開源模型 |
| E5-large-v2 | 1024 | 512 | 62.4 | 指令微調 |
| GTE-large | 1024 | 512 | 63.1 | Alibaba |
| Nomic-embed-text-v1.5 | 768 | 8192 | 62.3 | 長上下文、開源 |

<a id="selection-criteria"></a>
### 選擇標準

| 因素 | 考量 |
|--------|----------------|
| 品質（MTEB） | 越高越好，但更重要的是針對任務做評估 |
| 維度 | 越高表達力越強，但儲存與計算成本也越高 |
| 最大 tokens | 必須容納你的文件大小 |
| 成本 | API 與 self-hosting 的取捨 |
| 延遲 | 產生嵌入所需時間 |
| 多語言 | 若要服務非英文內容 |

---

<a id="matryoshka-and-adaptive-dimensions"></a>
## Matryoshka 與自適應維度

<a id="the-idea"></a>
### 核心概念

Matryoshka Representation Learning（MRL）會訓練嵌入，使完整嵌入的前綴片段也具備意義：

```python
full_embedding = model.encode(text)  # 1024 dimensions

# All these are valid embeddings with decreasing quality
dim_512 = full_embedding[:512]  
dim_256 = full_embedding[:256]
dim_128 = full_embedding[:128]
dim_64 = full_embedding[:64]
```

<a id="why-it-matters"></a>
### 為何重要

| 使用情境 | 維度 | 取捨 |
|----------|-----------|----------|
| 完整檢索 | 1024-3072 | 最高準確度 |
| **雙階段檢索**| 128 -> 1024 | **2025 年標準**：先用 128 維找回 1000 筆，再用 1024 維精修前 100 筆。 |
| 成本敏感 | 256 | 儲存節省 12 倍，MRR 損失 <2% |
| Edge / Mobile | 64 | 速度最快，可處理簡單意圖 |

<a id="models-with-matryoshka-support"></a>
### 支援 Matryoshka 的模型

- OpenAI text-embedding-3-*（原生支援）
- Nomic-embed-text-v1.5
- 多個 fine-tuned 模型

<a id="using-matryoshka-embeddings"></a>
### 使用 Matryoshka 嵌入

```python
from openai import OpenAI
client = OpenAI()

# Request smaller dimensions
response = client.embeddings.create(
    model="text-embedding-3-large",
    input="Your text here",
    dimensions=256  # Request 256 instead of full 3072
)
```

---

<a id="late-chunking-and-interaction"></a>
<a id="late-chunking-the-2025-shift"></a>
### Late Chunking（2025 年的轉變）

**傳統 Chunking：**
`Document -> Split into chunks -> Embed chunks individually`
- **問題**：Chunk 2 會失去來自 Chunk 1 的上下文。

**Late Chunking（由 Jina AI/Voyage 提出）：**
`Full Document -> Model Encoder -> Token-level Embeddings -> Pool into chunk boundaries`
- **優點**：每個 chunk 的 embedding 都包含**整份文件**的資訊，因為在 pooling 之前，Transformer 的 self-attention 已套用到完整序列。
- **需求**：模型必須支援長上下文（至少 8k+ tokens）。

---

<a id="quantization-for-scale"></a>
## 大規模量化

為了處理數十億個向量，**Binary** 與 **Scalar（Int8）** quantization 現在已成為標準。

| 類型 | 資料大小 | 記憶體節省 | 品質損失 | 支援者 |
|------|-----------|----------------|--------------|--------------|
| Float32 | 4 bytes/dim | 基準 | 0% | 全部 |
| Int8 | 1 byte/dim | 4x | <1% | Cohere, BGE |
| **Binary** | **1 bit/dim** | **32x** | ~5-10% | Cohere v3, v4 |

**Binary Quantization 模式：**
1. 先用 Binary embeddings 取回前 1000 筆（極致速度）。
2. 再用 Float32 或 Cross-Encoder 對前 50 筆 rerank（最高準確度）。

<a id="when-to-use-colbert"></a>
### 何時使用 ColBERT

- 檢索精度極為重要
- 能負擔額外的儲存成本
- Query latency 預算 > 50ms

<a id="implementation"></a>
### 實作

```python
# Using RAGatouille
from ragatouille import RAGPretrainedModel

model = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")

# Index documents
model.index(
    collection=documents,
    index_name="my_index"
)

# Search
results = model.search(query="What is RAG?", k=10)
```

---

<a id="practical-considerations"></a>
## 實務考量

<a id="batch-processing"></a>
### 批次處理

```python
# Inefficient: one API call per document
embeddings = [embed(doc) for doc in documents]

# Efficient: batch API calls
batch_size = 100
embeddings = []
for i in range(0, len(documents), batch_size):
    batch = documents[i:i + batch_size]
    batch_embeddings = embed_batch(batch)
    embeddings.extend(batch_embeddings)
```

<a id="chunking-for-embeddings"></a>
### 用於嵌入的 Chunking

長文件在建立嵌入前必須先切塊：

```python
def embed_document(document: str, max_tokens: int = 512) -> list[np.array]:
    chunks = chunk_document(document, max_tokens=max_tokens)
    embeddings = []
    for chunk in chunks:
        embedding = embed(chunk)
        embeddings.append(embedding)
    return embeddings
```

**考量事項：**
- Chunk 大小應小於模型的最大 token 限制
- Overlap 有助於保留 chunk 邊界間的上下文
- 必須儲存 chunk 與文件的對應關係，以利檢索

<a id="normalization"></a>
### 正規化

許多系統都預期嵌入已被正規化：

```python
def normalize(embedding):
    norm = np.linalg.norm(embedding)
    return embedding / norm

# Cosine similarity of normalized vectors = dot product
similarity = np.dot(normalize(a), normalize(b))
```

多數向量資料庫與 embedding API 會處理正規化，但仍要確認。

<a id="caching"></a>
### 快取

嵌入計算成本很高，因此要積極快取：

```python
import hashlib

def get_embedding(text: str, cache: dict) -> np.array:
    key = hashlib.sha256(text.encode()).hexdigest()
    
    if key in cache:
        return cache[key]
    
    embedding = compute_embedding(text)
    cache[key] = embedding
    return embedding
```

---

<a id="embedding-drift-and-versioning"></a>
## 嵌入漂移與版本管理

<a id="the-problem"></a>
### 問題

嵌入在以下情況之間不可直接比較：
- 不同模型
- 同一模型的不同版本
- 有時甚至是不同 API 呼叫（某些 API 具有非決定性）

<a id="consequences"></a>
### 後果

如果你更新了嵌入模型：
- 既有嵌入都會變得不相容
- 必須重新為整個語料庫建立嵌入
- 遷移期間的搜尋結果會不一致

<a id="mitigation-strategies"></a>
### 緩解策略

**1. 為嵌入加上版本：**
```python
embedding_metadata = {
    "model": "text-embedding-3-large",
    "model_version": "2024-01",
    "dimensions": 3072,
    "created_at": "2025-12-16"
}
```

**2. 預先規劃重新嵌入：**
- 估算完整 re-embed 的成本與時間
- 建立可在背景執行的 pipeline
- 切換前先測試新嵌入

**3. Blue-green deployment：**
```
Index A: Current embeddings
Index B: New embeddings (building)

Query -> Both indexes -> Merge or switch
```

**4. 追蹤嵌入品質：**
- 持續監控檢索指標
- 偵測嵌入分布漂移
- 在品質下降時發出警示

---

<a id="interview-questions"></a>
## 面試題

<a id="q-how-do-embedding-models-learn-semantic-similarity"></a>
### 問：嵌入模型如何學會語意相似度？

**理想答案：**
嵌入模型會透過對比式學習來訓練。其目標是讓語意相近文字的嵌入彼此接近，語意不同的嵌入彼此遠離。

訓練流程：
1. 正樣本對：應該相似的文字（query-document 配對、改寫句、翻譯）
2. 負樣本對：應該不相似的文字（常來自同一個 batch，或來自 BM25 的困難負樣本）
3. Loss function：把正樣本拉近、把負樣本推遠

模型會學會把文字放進高維空間中，讓距離與語意相似度對應。這讓檢索成為可能：先嵌入 query，再在文件嵌入空間中尋找最近鄰。

像 E5 與 BGE 這類現代模型也會做 instruction tuning，透過加上任務指令前綴來讓嵌入更貼合特定用途。

<a id="q-when-would-you-use-colbert-over-a-bi-encoder"></a>
### 問：什麼情況下你會選擇 ColBERT 而不是 bi-encoder？

**理想答案：**
ColBERT 使用 late interaction：不是每份文件只產生一個 embedding，而是保留每個 token 的 embedding。查詢時，它會計算 token 層級的相似度。

適合選擇 ColBERT 的情況：
- 檢索精度極為重要（法律、醫療、高風險場景）
- 你能負擔每份文件高出 10-100 倍的儲存成本
- Query latency 預算為 50ms 以上（比 bi-encoder 稍慢）
- 查詢會受益於 lexical matching（技術術語）

適合選擇 bi-encoder 的情況：
- 儲存受限
- 需要低於 20ms 的延遲
- bi-encoder 的檢索精度已足夠
- 經常需要重新建立索引（ColBERT reindex 成本高）

在實務上，常見模式是：用 bi-encoder 做第一階段檢索（前 100 筆），再用 cross-encoder 或 ColBERT 做 reranking。

<a id="q-how-do-you-handle-embedding-drift-when-updating-models"></a>
### 問：更新模型時，如何處理嵌入漂移？

**理想答案：**
嵌入模型產生的向量只在相同模型的前提下才有意義。如果更新模型，所有舊嵌入都會變得不相容。

我的做法：
1. **絕不原地更新。** 建立一個使用新嵌入的平行索引。
2. **切換前先測試。** 用測試集比較舊嵌入與新嵌入的檢索品質。
3. **背景重建。** 在背景用新模型重新嵌入整個語料庫。
4. **原子切換。** 新索引完整且驗證通過後，再原子性地切換流量。
5. **回滾計畫。** 保留舊索引，以便快速回滾。

成本估算範例：如果你有 1000 萬份文件，平均 500 tokens，而 text-embedding-3-large 的價格是 $0.13/100 萬 tokens，重新嵌入大約需要 $650。評估是否更新模型時，必須事先把這筆成本算進去。

<a id="q-how-do-you-choose-dimensions-for-embeddings"></a>
### 問：你如何選擇嵌入維度？

**理想答案：**
更高的維度能捕捉更多資訊，但也會增加儲存與計算成本。

考量因素：
- **儲存：** 1024 維 float32 = 每個 embedding 4 KB。若有 1000 萬份文件，光 embedding 就要 40 GB。
- **搜尋速度：** 維度越高，nearest neighbor search 越慢。
- **品質：** 對多數任務而言，超過某個維度後報酬會遞減。

實務做法：
1. 先從模型建議的維度開始。
2. 如果使用 Matryoshka 模型（例如 text-embedding-3），就在你的任務上實驗較低維度。
3. Benchmark 不同維度的品質：很多時候 256-512 維就有完整品質的 95%。
4. 若採用雙階段檢索：第一階段用低維度，reranking 時再用完整維度。

對大多數應用而言，768-1024 維能提供良好平衡。例外是對極高精度有要求的情境，此時 2048-4096 維可能有幫助。

---

<a id="references"></a>
## 參考資料

- Reimers and Gurevych. "Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks" (2019)
- Khattab and Zaharia. "ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction over BERT" (2020)
- Wang et al. "Text Embeddings by Weakly-Supervised Contrastive Pre-training" (E5, 2022)
- Xiao et al. "C-Pack: Packaged Resources To Advance General Chinese Embedding" (BGE, 2023)
- Kusupati et al. "Matryoshka Representation Learning" (MRL, 2022)
- MTEB Leaderboard: https://huggingface.co/spaces/mteb/leaderboard
- OpenAI Embeddings Guide: https://platform.openai.com/docs/guides/embeddings

---

*上一章：[Transformer Architecture](04-transformer-architecture.md) | 下一章：[Inference Pipeline](06-inference-pipeline.md)*

<a id="embedding-models"></a>
# 嵌入模型

Embedding models 會把文字轉成高維向量。技術前沿已經從靜態的單一向量表示，推進到 **multi-resolution、late-interaction，以及 multimodal** embeddings。

<a id="table-of-contents"></a>
## 目錄

- [Embedding 前沿（Matryoshka）]( #matryoshka)
- [Late Interaction（ColBERT v2）]( #late-interaction)
- [Binary 與 Int8 量化]( #quantization)
- [模型選型標準]( #selection)
- [多模態 Embeddings（Vision + Text）]( #multimodal)
- [面試題]( #interview-questions)
- [參考資料]( #references)

---

<a id="the-embedding-frontier-matryoshka-embeddings"></a><a id="matryoshka"></a>
## Embedding 前沿：Matryoshka Embeddings

傳統上，如果你把文字 embedding 成 1,536 維，就得在搜尋時使用完整的 1,536 維。

**Matryoshka Representation Learning (MRL)**
- 模型會被訓練成把最重要的資訊「存放」在前面幾個維度。
- **優勢**：你可以先用 1,536 維產生 embedding，但只索引前 **64 維** 來做快速搜尋，再用完整 1,536 維精煉前幾名結果。
- **效率**：記憶體／索引大小減少 20 倍，而準確率下降不到 2%。

---

<a id="late-interaction-colbert-v2"></a><a id="late-interaction"></a>
## Late Interaction：ColBERT v2

標準 embeddings 屬於「Bi-Encoders」（每個 chunk 一個向量）。**ColBERT**（Contextualized Late Interaction over BERT）採用的是「token-level」方法。

- **做法**：ColBERT 不再為每個 chunk 只存 1 個向量，而是為**每個 token** 各存 1 個向量。
- **互動方式**：在 query time，模型會將查詢中的每個 token 與文件中的每個 token 逐一比較（也就是「MaxSim」操作）。
- **現況**：ColBERT v2（以及後續的 ColPali、ColQwen2.5、ColNomic，適用於文件與 page-as-image）透過 PLAID indexing 大幅壓縮，已能在正式環境落地。它對於「大海撈針」式的技術查詢，精準度明顯更高。

---

<a id="binary-and-int8-quantization"></a><a id="quantization"></a>
## Binary 與 Int8 量化

儲存 `float32` 向量成本很高。正式環境的索引高度依賴 **in-model quantization**。

- **Binary Embeddings**：把向量轉成 1 和 0。
  - **記憶體**：縮小 32 倍。
  - **速度**：在現代 CPU 上，Hamming distance（XOR 運算）比 cosine similarity 快 10 倍。
- **Int8/Int4**：`text-embedding-3-small` 等模型原生支援。

---

<a id="model-selection-criteria"></a><a id="selection"></a>
## 模型選型標準

| 模型 | 供應商 | 特性 | Context |
|-------|----------|----------|---------|
| **Gemini Embedding 001** | Google | Multimodal（text、image、video、audio、PDF）、共用 3072 維空間、MTEB-English leader | 8k |
| **Qwen3-Embedding-8B** | Open Source | MTEB-Multilingual leader、instruction-tuned、擅長長文件 | 32k |
| **Llama-Embed-Nemotron-8B** | NVIDIA | 多語表現頂尖、open weights | 8k |
| **Cohere Embed v4** | Cohere | Multimodal（text + image）、Matryoshka、binary quantization | 128k |
| **Voyage-Multimodal-3.5** | Voyage AI | 統一文字／圖片、retrieval-tuned | 32k |
| **OpenAI text-embedding-3-large** | OpenAI | Matryoshka、Native Int8、支援廣泛 | 8k |
| **BGE-M3** | Open Source | 多語、多粒度（dense + sparse + late-interaction） | 8k |
| **Jina-Embeddings-v3** | Jina AI | 支援 late-interaction、長上下文 | 128k |

開源權重模型（Qwen3、Llama-Embed-Nemotron、BGE）現在在純 MTEB 分數上已能追平甚至超越商業 API。若你要的是託管基礎設施與 SLA，就選商業方案；若在高流量下更在意每次查詢成本，而不是最低延遲，就選 open weights。

---

<a id="multimodal-embeddings-vision-text"></a><a id="multimodal"></a>
## 多模態 Embeddings

純文字 RAG 會悄悄丟掉圖表、表格、示意圖與版面配置等訊號，而答案往往就藏在這些資訊裡。現代技術堆疊把頁面、截圖與圖像視為第一級檢索物件：

- **統一的 vision-text embeddings**：Cohere Embed v4、Voyage-Multimodal-3.5、Gemini Embedding 001 都使用單一向量空間，因此你可以拿「where is the emergency shutoff valve?」去查示意圖。
- **Page-as-image with late interaction**：ColPali、ColQwen2.5 與 ColNomic 會直接對頁面 render 結果做 embedding，跳過脆弱的 OCR，並保留視覺階層。
- **CLIP 家族模型**：對以圖片為主的型錄（如 e-commerce、media）仍然很好用，因為 text-image alignment 才是核心訊號。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-what-is-the-vocabulary-mismatch-problem-in-embeddings"></a>
### Q：embeddings 中的「Vocabulary Mismatch」問題是什麼？

**強回答：**
Embeddings 依賴訓練期間學到的語意空間。若使用者查詢用了較新的術語（例如 embedding model 截止日期之後才出現的模型名稱），而這個詞不在訓練資料中，模型可能只會把它映射成泛化的「AI」向量，錯失其特定語意。標準解法是 **Hybrid Search**（用 BM25 補捉精確關鍵字），再加上 **Cross-Encoder Reranking**，因為它同時看 query 與 document tokens，對 out-of-distribution 詞彙更穩健。

<a id="q-why-would-you-choose-a-matryoshka-model-for-a-1-billion-vector-index"></a>
### Q：為什麼你會為 10 億向量索引選擇 Matryoshka 模型？

**強回答：**
若使用標準 `float32` 的 1536 維 embeddings，要把索引擴展到 10 億向量，HNSW 約需要 6TB 高速 RAM，成本高得難以接受。使用 Matryoshka 模型後，我可以先用前 128 維（再搭配 Binary quantization）做初次檢索。這能把記憶體占用降低 90% 以上，讓「Top 1,000」候選可以在便宜得多的硬體上找出來，再只對這 1,000 筆取回完整解析度向量做最後 reranking。

---

<a id="references"></a>
## 參考資料
- Kusupati et al.「Matryoshka Representation Learning」（2022/2024 update）
- Khattab et al.「ColBERT v1 & v2: Efficient Late Interaction」（2021/2023）
- OpenAI.「Introducing New Embedding Models with Matryoshka Support」（2024）

---

*下一篇：[向量資料庫](04-vector-databases.md)*

<a id="multi-modal-rag"></a>
# 多模態 RAG

多模態 RAG 將 retrieval-augmented generation 從純文字擴展到可處理影像、表格、圖表、音訊，以及混合版面文件。如今正式環境系統經常需要匯入包含圖解的 PDF、簡報、掃描發票，以及研究論文，而這些文件中的視覺版面本身 *就是* 意義所在。當前主流有三種架構：caption-and-index、統一式 vision-text embedding（Cohere Embed v4、Voyage-Multimodal-3.5、Gemini Embedding 001），以及以 page-as-image 搭配 late interaction 的做法（ColPali、ColQwen2.5、ColNomic）。

<a id="table-of-contents"></a>
## 目錄

- [為什麼純文字 RAG 會失敗](#why-text-only-rag-fails)
- [架構模式](#architecture-patterns)
- [多模態嵌入策略](#multi-modal-embedding-strategies)
- [用於文件理解的 Vision-Language Models](#vision-language-models)
- [ColPali 與視覺式檢索](#colpali)
- [表格擷取與結構化資料檢索](#table-extraction)
- [圖表與示意圖理解](#chart-understanding)
- [正式環境架構](#production-architecture)
- [實作範例](#implementation-example)
- [系統設計面試角度](#system-design-interview-angle)
- [參考資料](#references)

---

<a id="why-text-only-rag-fails"></a>
## 為什麼純文字 RAG 會失敗

傳統 RAG pipeline 會把文件解析成文字 chunk、做 embedding，然後根據文字查詢來檢索。但這種做法在真實世界文件上會失效：

| 文件元素 | 純文字 RAG 的表現 | 實際遺失的資訊 |
|-----------------|----------------------|------------------------|
| **長條圖** | 只抽出座標軸標籤 | 趨勢、比較、數值量級 |
| **架構圖** | 完全漏掉 | 元件關係、資料流 |
| **表格** | 壓平的列失去結構 | 列欄關聯、表頭 |
| **資訊圖表** | 只抓到零散文字片段 | 視覺層級、空間分組 |
| **附圖說的照片** | 只拿到圖說、失去影像 | 視覺證據、空間脈絡 |

**現實**：企業文件中有 40–60% 屬於非文字內容。財報的價值在圖表；醫學論文的關鍵發現往往在 figures。忽略視覺內容，就等於忽略了大部分知識。

---

<a id="architecture-patterns"></a>
## 架構模式

多模態 RAG 主要有三種模式，各自有明確的取捨：

<a id="pattern-1-unified-embedding-space"></a>
### 模式 1：統一嵌入空間

```
                     Shared Vector Space
                    +-------------------+
  Text  --> Encoder |  [0.2, 0.8, ...] |
  Image --> Encoder |  [0.3, 0.7, ...] |  --> Single Index --> Retrieve
  Table --> Encoder |  [0.1, 0.9, ...] |
                    +-------------------+

  Query "show revenue trends" --> encode --> nearest neighbors across ALL modalities
```

- **作法**：使用像 CLIP 或 SigLIP 這類模型，將文字與影像投影到同一個向量空間。
- **優點**：單一索引、單一查詢、檢索邏輯簡單。
- **缺點**：不同模態的 embedding 品質不一致；表格仍需序列化。

<a id="pattern-2-modality-specific-retrieval-with-fusion"></a>
### 模式 2：依模態分開檢索，再做融合

```
  Query --> +----> Text Index    --> Top-K text chunks
            |
            +----> Image Index   --> Top-K images
            |
            +----> Table Index   --> Top-K tables
            |
            v
        Fusion / Reranking Layer --> Combined Top-K --> VLM Generator
```

- **作法**：每種模態使用獨立的 embedding 與索引，最後由 reranker 或 reciprocal rank fusion（RRF）合併結果。
- **優點**：每種模態都可用最佳 embedding；每個 retriever 都能獨立調校。
- **缺點**：基礎設施更複雜；融合邏輯不容易。

<a id="pattern-3-vision-first-page-as-image"></a>
### 模式 3：Vision-First（Page-as-Image）

```
  Document Page --> Screenshot/Render --> Vision Encoder --> Multi-vector Index
                                              |
  Query ---------> Text Encoder --------------+---> Late Interaction Score
                                                    --> Retrieve top pages
```

- **作法**：把每一頁文件都當作影像處理。使用 vision-language model（例如 ColPali）建立 patch-level embedding，再透過 late interaction（MaxSim）評分。
- **優點**：不需要 OCR、不需要版面解析、也不需要表格擷取 pipeline，可端到端訓練。
- **缺點**：索引時計算量較高；細粒度文字搜尋能力較弱。

**建議**：在文件密集場景中，模式 3（vision-first）正在快速成為主流；若你同時需要精準文字搜尋與視覺檢索，模式 2 仍是目前最穩定的正式環境主力。

---

<a id="multi-modal-embedding-strategies"></a>
## 多模態嵌入策略

<a id="clip-contrastive-language-image-pretraining"></a>
### CLIP（Contrastive Language-Image Pretraining）

這是最早將文字與影像映射到共享 512/768 維空間的 dual-encoder。

- **優勢**：生態系龐大、理解成熟、已有許多微調版本。
- **弱點**：對文件型影像（圖表、表格）不如對自然照片表現好；contrastive loss 需要較大的 batch size。

<a id="siglip--siglip-2"></a>
### SigLIP / SigLIP 2

它以 sigmoid loss 取代 CLIP 的 softmax cross-entropy，讓每組 image-text pair 都能獨立評估。

- **SigLIP 2（2025）**：加入 captioning decoder、self-distillation 與 masked prediction，並以 109 種語言、100 億+ 影像進行訓練。
- **關鍵優勢**：在較小 batch size（4–8k）下仍優於 CLIP，並提供更密集、更穩健的特徵。
- **正式環境應用**：挪威國家圖書館、電商視覺搜尋、AI 藝術策展。

<a id="comparison-for-rag"></a>
### 用於 RAG 的比較

| 模型 | 最適合 | 嵌入維度 | 文件品質 | 自然影像品質 |
|-------|----------|--------------|-----------------|----------------------|
| CLIP ViT-L/14 | 通用用途 | 768 | 中 | 高 |
| SigLIP 2 So400m | 多語文件 | 1152 | 高 | 高 |
| Nomic Embed Vision | 文字密集文件 | 768 | 高 | 中 |
| Voyage Multimodal 3 | 混合型文件 | 1024 | 高 | 高 |

<a id="embedding-strategy-decision"></a>
### 嵌入策略決策

```
Is your content mostly natural images (photos, products)?
  YES --> CLIP or SigLIP fine-tuned on your domain
  NO
    |
    v
Is your content document pages (PDFs, slides, reports)?
  YES --> ColPali / ColQwen (vision-first, no OCR needed)
  NO
    |
    v
Is it a mix of text, images, and structured data?
  YES --> Modality-specific encoders + fusion (Pattern 2)
```

---

<a id="vision-language-models-for-document-understanding"></a>
## 用於文件理解的 Vision-Language Models

VLM 在多模態 RAG 中扮演兩種角色：（1）作為 **generator**，從檢索到的多模態脈絡綜合生成答案；（2）作為 **indexing engine**，在匯入階段萃取結構化資訊。

<a id="vlm-capabilities-comparison"></a>
### VLM 能力比較

| 能力 | Claude Opus 4.7 / Sonnet 4.6 | GPT-5.5 | Gemini 3.1 Pro |
|-----------|------------------------------|---------|----------------|
| **圖表閱讀** | 極佳 | 極佳 | 極佳 |
| **表格擷取** | 極佳 | 良好 | 極佳 |
| **示意圖理解** | 極佳 | 良好 | 極佳 |
| **手寫 OCR** | 良好 | 良好 | 良好 |
| **多頁推理** | 極佳（Sonnet 4.6 可到 1M ctx） | 極佳（1M ctx） | 極佳（1M ctx） |
| **結構化輸出** | 原生 JSON mode | 原生 JSON mode | 原生 JSON mode |

<a id="vlm-augmented-ingestion-pipeline"></a>
### VLM 增強式匯入 pipeline

```
  Raw PDF
    |
    v
  Page Renderer (pdf2image, 300 DPI)
    |
    v
  VLM Extraction Pass:
    +-- "Extract all tables as markdown"
    +-- "Describe this chart: axes, trends, key data points"
    +-- "Summarize the diagram: components and relationships"
    |
    v
  Structured Output (JSON)
    |
    +---> Text chunks     --> Text embedding index
    +---> Table markdown  --> Text embedding index (with metadata: "type=table")
    +---> Chart summaries --> Text embedding index (with metadata: "type=chart")
    +---> Page images     --> Image embedding index (CLIP/SigLIP)
```

這種「先描述、再嵌入」的方法，能把視覺內容轉成可搜尋文字，同時保留原始影像供生成階段使用。

---

<a id="colpali-and-vision-based-retrieval"></a>
## ColPali 與視覺式檢索

ColPali 代表一種典範轉移：與其建立複雜的 OCR + 版面 + 表格擷取 pipeline，不如把每一頁文件都當成單一影像，交給 vision-language model 全權處理。

<a id="how-colpali-works"></a>
### ColPali 如何運作

```
  Document Page Image
        |
        v
  SigLIP Vision Encoder (So400m)
        |
  Splits image into patches (e.g., 32x32 grid = 1024 patches)
        |
        v
  Gemma 2B Language Model (contextualizes patch embeddings)
        |
        v
  Linear Projection --> 128-dim patch embeddings
        |
  Result: 1024 vectors of dim 128 per page
        |
        v
  Stored in Multi-Vector Index

  At query time:
  Query --> Tokenize --> Embed --> 128-dim token embeddings
        |
        v
  Late Interaction (MaxSim):
    Score = Sum over query tokens of Max similarity to any patch
```

<a id="colpali-vs-traditional-pipeline"></a>
### ColPali 與傳統 pipeline 比較

| 面向 | 傳統 pipeline | ColPali |
|--------|---------------------|---------|
| **OCR** | 必要（Tesseract、Azure OCR） | 不需要 |
| **版面偵測** | 必要（Detectron2、LayoutLM） | 不需要 |
| **表格解析器** | 必要（Camelot、Tabula） | 不需要 |
| **圖表擷取器** | 必要（ChartOCR） | 不需要 |
| **索引速度** | 慢（多階段） | 快（單次 forward pass） |
| **檢索品質** | 文字高、視覺差 | 各模態都高 |
| **儲存量** | 文字索引（較小） | 多向量索引（較大） |

<a id="colpali-family"></a>
### ColPali 家族

- **ColPali（v1）**：以 PaliGemma-3B 為 backbone，最原始版本。
- **ColQwen 2.5**：以 Qwen2-VL 為 backbone，多語言支援更好，對亞洲語言文件表現更佳。
- **ColSmol**：較小型變體，適合 edge deployment，約 10 億參數。

<a id="vidore-benchmark-results"></a>
### ViDoRe 基準結果

ColPali 在 InfographicVQA、ArxivQA、TabFQuAD 等視覺複雜基準上表現出色，這些資料集分別測試資訊圖表、圖形與表格。即使在文字為主的文件上，它也優於傳統文字型 pipeline。

---

<a id="table-extraction-and-structured-data-retrieval"></a>
## 表格擷取與結構化資料檢索

表格是傳統 RAG 最難處理的模態。若逐列將表格壓平成文字，就會破壞每個儲存格賴以成立的欄位—表頭關係。

<a id="strategy-1-vlm-based-extraction"></a>
### 策略 1：基於 VLM 的擷取

```python
# Pseudocode: Extract tables using a VLM
def extract_tables_from_page(page_image: bytes) -> list[dict]:
    prompt = """
    Extract ALL tables from this document page.
    For each table, return:
    {
      "title": "table title or caption",
      "headers": ["col1", "col2", ...],
      "rows": [["val1", "val2", ...], ...],
      "markdown": "| col1 | col2 |\n|---|---|\n| val1 | val2 |"
    }
    Return JSON array. If no tables, return [].
    """
    response = vlm.generate(image=page_image, prompt=prompt)
    return json.loads(response)
```

<a id="strategy-2-specialized-table-parsers"></a>
### 策略 2：專用表格解析器

- **Tabula / Camelot**：基於規則的 PDF 表格擷取，速度快，但遇到複雜版面很脆弱。
- **Table Transformer（基於 DETR）**：從影像中偵測表格邊界與儲存格結構。
- **Unstructured.io**：結合 heuristics 與 ML model，做版面感知解析。

<a id="strategy-3-table-aware-chunking"></a>
### 策略 3：具表格感知的 chunking

```
  Original Table (20 rows x 8 columns)
        |
        v
  Chunk as complete unit (do NOT split tables across chunks)
        |
        v
  Embed the full markdown table as a single chunk
        |
        v
  Add metadata: {"type": "table", "page": 14, "caption": "Q3 Revenue by Region"}
        |
        v
  At generation time: pass the FULL table to the LLM, not a fragment
```

**關鍵原則**：表格必須是原子化的檢索單位。絕對不要把表格切開跨越 chunk 邊界。

---

<a id="chart-and-diagram-understanding"></a>
## 圖表與示意圖理解

<a id="chart-types-and-extraction-approaches"></a>
### 圖表類型與擷取方法

| 圖表類型 | 需要擷取什麼 | 最佳方法 |
|-----------|----------------|---------------|
| **長條／折線／圓餅圖** | 數值、趨勢、比較 | VLM 描述 + 資料表擷取 |
| **流程圖** | 步驟、決策點、連線 | VLM 結構化擷取（nodes + edges） |
| **架構圖** | 元件、關係、資料流 | VLM 描述 + 實體擷取 |
| **散佈圖** | 關聯、離群值、群聚 | VLM 趨勢描述 + 可用時附原始資料 |
| **甘特圖** | 時程、依賴、里程碑 | VLM 結構化擷取 |

<a id="dual-representation-strategy"></a>
### 雙重表示策略

對每一張圖表或示意圖，儲存 **兩種** 表示：

```
  Chart Image
    |
    +---> (1) Text Description (for text-based retrieval)
    |         "This bar chart shows Q3 revenue by region.
    |          North America: $4.2M, Europe: $3.1M, APAC: $2.8M.
    |          NA grew 15% QoQ while APAC declined 3%."
    |
    +---> (2) Original Image (for visual retrieval + generation context)
              Stored with CLIP/SigLIP embedding for image-based queries
```

這能確保圖表既可透過文字查詢檢索（如「APAC 營收是多少？」），也可透過視覺查詢檢索（如「把營收圖表給我看」）。

---

<a id="production-architecture"></a>
## 正式環境架構

<a id="full-multi-modal-rag-pipeline"></a>
### 完整多模態 RAG pipeline

```
  INGESTION:
  Raw Docs --> Doc Classifier --+--> Text-Heavy  --> chunking + text embeddings
                                +--> Visual-Heavy --> page render + ColPali
                                +--> Mixed        --> VLM extraction + hybrid
                                         |
                                         v
                          [Text Index] [Image Index] [Table Index]

  RETRIEVAL:
  Query --> Query Analyzer --+--> Text:  BM25 + dense search
                             +--> Image: CLIP/ColPali search
                             +--> Table: metadata-filtered dense
                                    |
                                    v
                             Cross-Modal Reranker --> Context Assembly --> VLM --> Response
```

<a id="scaling-considerations"></a>
### 擴展考量

| 顧慮 | 解法 |
|---------|----------|
| **索引大小** | ColPali 每頁約存 ~1024 個向量。100 萬頁約等於 ~10 億向量，需使用量化（binary、PQ）。 |
| **匯入延遲** | VLM 擷取很慢（約 2–5 秒／頁），需用非同步 worker 搭配 GPU 加速。 |
| **查詢延遲** | 多索引 fan-out 會增加延遲，需採平行檢索 + 積極的 top-k pruning。 |
| **成本** | 匯入時的 VLM 呼叫是一次性成本，可攤提至查詢量；預算可抓每頁 $0.01–0.05。 |
| **儲存** | 將頁面影像放 object storage（S3），embedding 存向量 DB，文字存 search index。 |

---

<a id="implementation-example"></a>
## 實作範例

<a id="end-to-end-multi-modal-rag-with-colpali--vlm"></a>
### 以 ColPali + VLM 建立端到端多模態 RAG

```python
# Pseudocode: Production multi-modal RAG pipeline

from colpali_engine import ColPali, ColPaliProcessor
from qdrant_client import QdrantClient
import anthropic

# --- INDEXING ---

def index_document(pdf_path: str, collection: str):
    """Index a PDF document using ColPali for visual retrieval
    and VLM extraction for text-based retrieval."""

    pages = render_pdf_to_images(pdf_path, dpi=300)

    colpali_model = ColPali.from_pretrained("vidore/colpali-v1.3")
    processor = ColPaliProcessor.from_pretrained("vidore/colpali-v1.3")
    vlm_client = anthropic.Anthropic()

    for page_num, page_image in enumerate(pages):
        # 1. Generate ColPali multi-vector embeddings
        inputs = processor(images=[page_image])
        patch_embeddings = colpali_model(**inputs)  # shape: [1, 1024, 128]

        # 2. Extract structured content via VLM
        extraction = vlm_client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            messages=[{
                "role": "user",
                "content": [
                    {"type": "image", "source": encode_image(page_image)},
                    {"type": "text", "text": """Extract from this page:
                    1. All text content (preserve structure)
                    2. Tables as markdown
                    3. Chart descriptions with data points
                    Return as JSON with keys: text, tables, charts"""}
                ]
            }]
        )

        structured = json.loads(extraction.content[0].text)

        # 3. Store in vector DB
        qdrant.upsert(collection, points=[
            # ColPali multi-vector for visual retrieval
            PointStruct(
                id=f"{pdf_path}:page:{page_num}:colpali",
                vector={"colpali": patch_embeddings[0].tolist()},
                payload={
                    "source": pdf_path,
                    "page": page_num,
                    "type": "page_image",
                    "text_preview": structured["text"][:500]
                }
            ),
            # Text embeddings for each extracted element
            *create_text_chunks(structured, pdf_path, page_num)
        ])


# --- RETRIEVAL ---

def retrieve(query: str, collection: str, top_k: int = 5):
    """Hybrid retrieval: ColPali visual + text semantic search."""

    # Visual retrieval via ColPali
    query_inputs = processor(text=[query])
    query_embeddings = colpali_model(**query_inputs)

    visual_results = qdrant.query(
        collection,
        query_vector=("colpali", query_embeddings[0].tolist()),
        limit=top_k,
        query_filter=Filter(must=[FieldCondition(key="type", match="page_image")])
    )

    # Text retrieval via dense embeddings
    text_embedding = text_encoder.encode(query)
    text_results = qdrant.search(
        collection,
        query_vector=("text", text_embedding.tolist()),
        limit=top_k
    )

    # Fuse results using reciprocal rank fusion
    fused = reciprocal_rank_fusion(visual_results, text_results, k=60)
    return fused[:top_k]


# --- GENERATION ---

def generate_answer(query: str, retrieved_context: list) -> str:
    """Generate answer using VLM with multi-modal context."""

    content_blocks = [{"type": "text", "text": f"Question: {query}\n\nContext:"}]

    for ctx in retrieved_context:
        if ctx.payload["type"] == "page_image":
            # Include the actual page image
            content_blocks.append({
                "type": "image",
                "source": load_page_image(ctx.payload["source"], ctx.payload["page"])
            })
        else:
            # Include text/table content
            content_blocks.append({
                "type": "text",
                "text": f"[{ctx.payload['type']}] {ctx.payload['content']}"
            })

    content_blocks.append({
        "type": "text",
        "text": "Answer the question using ONLY the provided context. Cite sources."
    })

    response = vlm_client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        messages=[{"role": "user", "content": content_blocks}]
    )
    return response.content[0].text
```

---

<a id="system-design-interview-angle"></a>
## 系統設計面試角度

<a id="q-design-a-rag-system-for-a-financial-research-platform-that-needs-to-answer-questions-about-earnings-reports-containing-text-tables-and-charts"></a>
### Q：設計一個給金融研究平台使用的 RAG 系統，需回答包含文字、表格與圖表的財報問題。

**強答範例：**

核心挑戰在於，財報中 60% 以上的重要資訊存在於表格與圖表，而非敘述文字。若只用文字型 RAG pipeline，就會錯過營收拆解、趨勢線與比較性資料。

**架構**：我會採用混合方案（模式 2，加上一部分模式 3）：

1. **匯入**：每一頁 PDF 以 300 DPI render。先跑一輪 VLM 擷取，把表格轉為 markdown、把圖表轉成結構化描述；同時為每一頁影像產生 ColPali 多向量 embedding。

2. **儲存**：建立三個索引——（a）文字 chunk 搭配 dense embedding（金融文字）、（b）表格 markdown 搭配 dense embedding，並附加表格類型等 metadata filter、（c）頁面層級視覺檢索用的 ColPali 多向量索引。

3. **檢索**：由 query analyzer 判定查詢類型。「What was Q3 revenue?」會觸發文字 + 表格搜尋；「Show me the revenue trend」則觸發視覺（ColPali）搜尋。結果再用 RRF 融合，並交給 cross-encoder rerank。

4. **生成**：VLM（Claude 或 Gemini）接收融合後的脈絡——文字 chunk、表格 markdown 與相關頁面影像，生成帶有頁碼與表格引用的 grounded answer。

**關鍵取捨**：ColPali 對視覺內容有很好的 recall，但每頁要儲存約 ~1024 個向量，因此若有 10 萬份文件（50 萬頁），就會是 ~5 億向量。我會使用 binary quantization，把儲存量降 32 倍，接受一點 recall 損失。文字路徑則由 BM25 + dense hybrid search 來處理金融術語。

<a id="q-how-would-you-handle-a-query-that-requires-information-from-both-a-chart-and-a-table-on-different-pages"></a>
### Q：如果查詢需要同時從不同頁面的圖表與表格取得資訊，你會怎麼處理？

**強答範例：**

這是 cross-modal、cross-page 的檢索問題。解法有三個部分：

1. **檢索多樣性**：確保 retriever 會回傳多種模態的結果。可設定最小配額——每次檢索結果中，至少包含 2 個文字結果、2 個表格結果與 1 個視覺結果，不論哪一種模態分數最高。

2. **脈絡組裝**：在組裝 VLM prompt 時，把所有檢索內容連同明確來源一起放入，例如「[第 14 頁表格：Q3 Revenue by Region]」與「[第 22 頁圖表：Revenue Trend 2024-2026]」。如此 VLM 才能跨模態推理。

3. **Agentic fallback**：若第一次檢索沒有找出足夠的跨模態內容，agentic layer 可以再發出後續檢索：「表格有營收數字，但使用者問的是趨勢——我還需要再搜尋與營收相關的圖表。」

關鍵洞見是：跨模態問題本質上就是 multi-hop。系統必須先從一種模態檢索、辨識缺口，再去另一種模態補足。

---

<a id="references"></a>
## 參考資料

- Faysse et al. "ColPali: Efficient Document Retrieval with Vision Language Models" (ICLR 2025)
- Google. "SigLIP 2: Multilingual Vision-Language Encoders" (2025)
- NVIDIA. "An Easy Introduction to Multimodal Retrieval-Augmented Generation" (2025)
- HKUDS. "RAG-Anything: All-in-One Multimodal RAG Framework" (2025)
- Vespa Blog. "PDF Retrieval with Vision Language Models" (2024)

---

*上一篇：[Advanced Retrieval Patterns](09-advanced-retrieval-patterns.md) | 下一篇：[RAG Evaluation Patterns](13-rag-evaluation-patterns.md)*

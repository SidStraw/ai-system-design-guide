<a id="ocr-and-layout-analysis"></a>
# OCR 與版面分析

傳統 OCR（Tesseract、專用引擎）如今大多已被 **原生多模態 LLMs**（Gemini 3.1 Pro、GPT-5.5、Claude Sonnet 4.6、Claude Opus 4.7）取代。我們不再只是「讀取字元」，而是「理解版面」。

<a id="table-of-contents"></a>
## 目錄

- [轉變：傳統 OCR 與 Vision-LLMs](#shift)
- [Vision-LLM 版面擷取](#layout-extraction)
- [閱讀順序與邏輯結構](#reading-order)
- [處理低品質掃描與手寫內容](#quality)
- [成本與延遲取捨](#tradeoffs)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="the-shift-traditional-ocr-vs-vision-llms"></a>
## 轉變：傳統 OCR 與 Vision-LLMs

| 特性 | 傳統 OCR（Tesseract/AWS Textract） | Vision-LLMs（Gemini 3.1 Pro、GPT-5.5、Claude Opus 4.7） |
|---------|-------------------------------------------|--------------------------------------------------------|
| **主要機制** | 字元辨識 | 視覺 token 理解 |
| **邏輯** | 點與線分析 | 語意脈絡 |
| **閱讀順序** | 簡單由上到下 | 可理解多欄與複雜版面 |
| **手寫** | 差 | 優秀（接近人類水準） |
| **輸出** | 文字區塊 + Bounding boxes | 結構化 Markdown/JSON |

---

<a id="vision-llm-layout-extraction"></a>
## Vision-LLM 版面擷取

標準工作流程是 **Screenshot-to-Markdown**。
1. **Rasterize**：將 PDF 頁面轉成影像。
2. **Visual Prompting**：要求 vision model「將以下頁面轉寫為 GitHub-flavored Markdown，並保留表格與標題。」
3. **Structured Recovery**：利用模型的空間感知能力重建邏輯階層。

---

<a id="reading-order-and-logical-structure"></a>
## 閱讀順序與邏輯結構

> [!IMPORTANT]
> naive RAG 的常見失敗之一，是把同一段文字切跨到不同欄位。 
> Vision-LLMs 能透過「看見」欄間留白來正確排列文字順序；不同於 rule-based parser 可能會直接橫向讀過兩欄內容。

---

<a id="handling-low-quality-scans"></a>
## 處理低品質掃描與手寫內容

現代多模態模型對以下情況具有良好韌性：
- **Skew/Rotation**：會在視覺注意力層中自動校正。
- **Bleed-through**：模型會利用語意脈絡來「忽略」來自紙張背面的文字。
- **Handwritten Annotations**：可擷取到獨立的 `annotations` JSON 欄位中。

---

<a id="cost-and-latency-tradeoffs"></a>
## 成本與延遲取捨

| Model Tier | 使用情境 | 延遲 | 成本（1K 頁） |
|------------|----------|---------|-----------------|
| **Gemini 3.1 Flash** | 高量批次 | 1-2s / page | $1-3 |
| **GPT-5.5 / Claude Sonnet 4.6** | 高精度 / 法務 | 3-5s / page | $8-18 |
| **Local (Llama 4 Vision)** | 對 PII 敏感 / On-prem | <1s / page | 僅基礎設施成本 |

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-why-would-you-still-use-aws-textract-or-azure-ai-search-ocr-when-vision-llms-exist"></a>
### 問：當 Vision LLMs 已存在時，為什麼你仍會使用 AWS Textract 或 Azure AI Search (OCR)？

**理想回答：**
**嚴格的空間中繼資料與合規需求**。如果我的應用需要每個單字的精確像素級 bounding boxes（例如法務遮罩工具），專用 OCR 引擎通常會更精準且更便宜。此外，OCR 引擎具備 **Deterministic** 特性：它們不會對不存在的文字產生「Hallucinate」。對於高風險的文件處理場景，若相較於「版面理解」更要求 100% 字元準確率，傳統引擎在混合式 pipeline 中仍有其位置。

<a id="q-how-do-you-handle-a-500-page-pdf-with-vision-llms-efficiently"></a>
### 問：你會如何高效率地用 Vision LLMs 處理一份 500 頁 PDF？

**理想回答：**
我們會使用 **Parallel Map-Reduce** 模式。
1. **Map**：啟動 50 個平行 worker（使用 AWS Lambda 或 Modal），每個 worker 處理 10 頁。每個 worker 呼叫快速 Vision model（例如 Gemini 3 Flash）取得 Markdown。
2. **Consolidate**：中央代理審查各段 Markdown，確保標題銜接一致。
3. **Cache**：將產生的 Markdown 儲存在 vector DB 中。
這能把處理時間從 30 分鐘（循序處理）縮短到 20 秒以內。

---

<a id="references"></a>
## 參考資料
- Google DeepMind. "Gemini 2.0: Understanding Multi-column Documents" (2025)
- OpenAI. "Vision Models for Document Understanding" (2025)
- Tesseract v6. "The Integration of Hybrid Transformer OCR" (2025)

---

*下一步：[Multimodal Parsing and Markdown Conversion](02-multimodal-parsing.md)*

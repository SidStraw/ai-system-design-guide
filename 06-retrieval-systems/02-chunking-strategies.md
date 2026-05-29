<a id="chunking-strategies"></a>
# 分塊策略

Chunking 是將文件切分成離散片段以供檢索的過程。正式環境的 pipeline 已不再停留在盲目的固定大小切分，而是轉向 **structure-aware 與 semantic segments**；較新的技術如 late chunking 與 contextual prepending，也已成為主流工具箱的一部分。

<a id="table-of-contents"></a>
## 目錄

- [檢索與上下文的張力](#tension)
- [遞迴式結構切分](#recursive)
- [語意分塊](#semantic)
- [階層式（Parent-Child）分塊](#hierarchical)
- [內容特化策略（Code、PDF、Tables）](#content-specific)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-retrieval-context-tension"></a><a id="tension"></a>
## 檢索與上下文的張力

| 面向 | 小分塊（100t） | 大分塊（1000t） |
|--------|---------------------|----------------------|
| **精準度** | 高（精確匹配） | 低（被稀釋） |
| **上下文** | 差（句子被切斷） | 豐富（周邊資訊完整） |
| **儲存成本** | 高（更多向量） | 低（較少向量） |
| **延遲** | 低（搜尋快） | 高（檢索負擔重） |

**規則**：小 chunk 比較適合**找**資料，但大 chunk 更適合**理解與推理**。使用 **Hierarchical Chunking** 才能兩者兼得。

---

<a id="recursive-structure-splitting"></a><a id="recursive"></a>
## 遞迴式結構切分

與其每 500 個字元硬切一次，我們更應該依照邏輯邊界切分：
`[Double Newline] > [Single Newline] > [Period] > [Space]`。

**最佳實務**：使用 **Markdown-Aware Splitting**。若文件有 `#` 標題，務必把標題 prepend 到**每個**子 chunk 前面，以保留上下文（Contextual Chunking）。

---

<a id="semantic-chunking"></a><a id="semantic"></a>
## 語意分塊

Semantic chunking 會用 embedding model 偵測「topic shifts」。

1. 先把文字切成單一句子。
2. 只要句子間 embedding 相似度高於某個門檻（例如 0.82），就將它們分在同一組。
3. 一旦相似度下降，就開始新的 chunk。

**細節**：正式環境的 pipeline 越來越常使用 **Cross-Encoder Segmenters**。一個小型模型會掃描文字，並在每個語意斷點預測「Separator token」。這比單純用 cosine similarity 門檻切分，準確度高出約 10 倍。

---

<a id="hierarchical-parent-child-chunking"></a><a id="hierarchical"></a>
## 階層式（Parent-Child）分塊

這是正式環境 RAG 的業界標準。

- **流程**：
  1. 建立 1,500 tokens 的「Parent」chunks。
  2. 將每個 parent 再切成 5 個 300 tokens 的「Child」chunks。
  3. **只索引 children**。
  4. 檢索時，只要 child 命中，就把**完整的 parent context** 回傳給 LLM。
- **原因**：child 體積小，vector DB 容易比對；parent 則提供足夠上下文，讓 LLM 能正確推理，避免因「Broken Context」產生幻覺。

---

<a id="content-specific-strategies-code-pdf-tables"></a><a id="content-specific"></a>
## 內容特化策略

<a id="1-code-chunking"></a>
### 1. Code 分塊
- **策略**：使用 AST（Abstract Syntax Tree）解析。
- **規則**：絕對不要在函式主體中間切開。保留 imports 與 class declarations 和其 methods 在一起。

<a id="2-table-chunking"></a>
### 2. Table 分塊
- **策略**：表格使用 Markdown 格式保存。
- **2025 模式**：「Summarized Tables」。在 vector DB 中儲存表格的自然語言摘要，但回傳給 LLM 的仍是完整 Markdown 表格。

<a id="3-pdflayout-chunking"></a>
### 3. PDF／版面分塊
- **策略**：使用 **Vision-Language Model (VLM)** 預處理（例如 ColPali）。
- **細節**：不只儲存文字，而是儲存能表達頁面**位置版面**的 embeddings，確保圖表與側欄不會混進正文。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-is-fixed-size-chunking-with-overlap-problematic-for-production-systems"></a>
### Q：為什麼固定大小加 overlap 的 chunking，對正式環境來說有問題？

**強回答：**
固定大小分塊是「content-blind」的。它經常在句子思路尚未結束時就切開、破壞數學公式，也會把標題和對應說明拆散。雖然「Overlap」（例如 10%）能靠跨 chunk 重複 10% 文字來稍微緩解，但它沒有解決核心問題：模型的注意力仍被迫從碎片化字串中重建語意。現代 pipeline 更偏好 **Semantic 或 Logical Chunking**，因為它能保證每個向量代表一個「完整語意單元」，大幅提升檢索精準度。

<a id="q-what-is-contextual-retrieval-the-anthropic-pattern"></a>
### Q：什麼是「Contextual Retrieval」（Anthropic 的模式）？

**強回答：**
Contextual Retrieval 指的是在 embedding 前，先為每個 chunk prepend 一句全域上下文。例如，若某個 chunk 談的是「battery life」，但它出自「2025 Model X Drone」手冊，就可以在 chunk 前加上 `[Drone_Model_X_Manual]:`。如此一來，「battery life」的向量就會受到「Drone」語境影響，不會被誤抓去回答「phone battery」相關查詢。

---

<a id="references"></a>
## 參考資料
- Anthropic.「Contextual Retrieval: Improving RAG Accuracy」（2024）
- LlamaIndex.「Advanced Chunking Strategies for RAG」（2025）
- LangChain.「RecursiveCharacterTextSplitter Benchmarks」（2024）

---

*下一篇：[嵌入模型](03-embedding-models.md)*

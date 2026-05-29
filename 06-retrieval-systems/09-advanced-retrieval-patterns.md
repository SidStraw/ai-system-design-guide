<a id="advanced-retrieval-patterns"></a>
# 進階檢索模式

除了基礎做法外，正式環境中的 RAG 系統還會使用各種專門模式，來處理複雜的 query-document 落差。這些模式正是高精度搜尋的「秘密醬料」，也越來越常被打包進受管式 RAG 服務中。

<a id="table-of-contents"></a>
## 目錄

- [Query Decomposition（Multi-Query）](#query-decomposition-multi-query)
- [Hypothetical Document Embeddings（HyDE）](#hypothetical-document-embeddings-hyde)
- [Contextual Retrieval（Anthropic 模式）](#contextual-retrieval-the-anthropic-pattern)
- [迭代式文件增豐](#iterative-document-enrichment)
- [In-Context 重排序](#in-context-reranking)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="query-decomposition-multi-query"></a>
## Query Decomposition（Multi-Query）

複雜的使用者 query 常常是「複合型 query」。
- **使用者**：「比較我們 Q3 與 Q4 的營收，並解釋下滑原因。」
- **分解後**：
  1. 「Q3 Revenue」
  2. 「Q4 Revenue」
  3. 「Q4 營收差異的原因」
- **實作方式**：使用 LLM 產生這 3 個子查詢，對資料庫把它們全部查一次，再整合 context。

---

<a id="hypothetical-document-embeddings-hyde"></a>
## Hypothetical Document Embeddings（HyDE）

Query 很短，文件很長。這種「不對稱（Asymmetry）」很容易導致檢索失敗。
- **模式**：
  1. 取得使用者 query。
  2. 問 LLM：「請先寫一段 1 段落的假想答案。」
  3. **不要嵌入原始 query，而是嵌入這段假想答案。**
- **為什麼有效？**：假想答案會落在與真實文件更接近的「向量鄰域」中，因此能大幅提升召回率。

---

<a id="contextual-retrieval-the-anthropic-pattern"></a>
## Contextual Retrieval（Anthropic 模式）

此模式由 Anthropic 在 2024 年底標準化，用來解決 **Context Dilution**。

- **問題**：某個 chunk 可能只寫著「它的價格是 200 美元」，但若沒有標題，我們就不知道這個「它」是「Widget-X」。
- **模式**：在 ingestion 階段，對每個 300-token chunk，都讓 LLM 產生一段 50-token 的上下文字串（例如「這個 chunk 在說明北美市場中 Widget-X 的定價」）。
- **效益**：對碎片化資料可提升 30-50% 的檢索精度。

---

<a id="iterative-document-enrichment"></a>
## 迭代式文件增豐

我們不只儲存原始文件，也會儲存「增豐後（Enriched）」的 metadata。
- **摘要**：儲存一段 1 段落的文件摘要。
- **Q&A 生成**：產生這份文件能回答的 5 個問題，並把這些問題 *與* 文件一起做 embedding。
- **現況**：現在許多高階 RAG 系統嵌入的是 **「問題」**，而不是 **「答案」**，以更貼近使用者 query 意圖。

---

<a id="in-context-reranking"></a>
## In-Context 重排序

現在 1M-2M context windows 已逐漸成為標準（Claude Sonnet 4.6、Gemini 3.1 Pro），因此 **Rank-by-Context** 已是可行模式。
1. 取回 Top 100 文件。
2. 將全部 100 份文件放進 context window。
3. 要求模型：「讀完這 100 份文件，找出最相關的 5 份，然後用那 5 份來回答。」
- **優勢**：這能利用模型的 **Long Context Reasoning**，在不需要額外 Cross-Encoder 模型的情況下完成重排序。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-is-hyde-hypothetical-document-embedding-risky-for-some-applications"></a>
### Q：為什麼 HyDE（Hypothetical Document Embedding）在某些應用中有風險？

**強答：**
HyDE 依賴先「幻覺」出一個基準答案，再用它來找真實資料。如果使用者 query 描述的是不存在或邏輯上不可能的事，LLM 還是會生成一段假想答案。這可能把資料庫中「錯誤但語意相近」的資料拉進來，反而強化模型最初的幻覺。標準緩解方式是 **Hybrid approach**：先用真實 query（Keyword）檢索一次，再用 HyDE query 檢索一次，最後用 **RRF** 把兩者合併。

<a id="q-what-is-the-asymmetric-retrieval-problem"></a>
### Q：「Asymmetric Retrieval」問題是什麼？

**強答：**
Asymmetric retrieval 指的是：使用者 query 通常很短（3-10 個字），而文件 chunk 很長（300-500 個字）。它們在向量空間中落在不同的統計分布上，因此會造成「Distance Bias」。高效能系統通常用 **Asymmetric Encoders**（query 與 doc 各用一個模型）或 **Query Expansion**（HyDE）來把 query「膨脹」成更像文件的分布。

---

<a id="references"></a>
## 參考資料
- Gao et al. "Precise Zero-Shot Dense Retrieval without Relevance Labels" (HyDE, 2023/2024)
- Anthropic. "The Contextual Retrieval Playbook" (2024)
- LlamaIndex. "Query Transformation Cookbook" (2025)

---

*下一節：[Agentic Systems](../07-agentic-systems/01-agents-fundamentals.md)*


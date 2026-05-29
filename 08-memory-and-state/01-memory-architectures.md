
<a id="memory-architectures"></a>
# 記憶體架構

LLM 記憶體已從「歷史緩衝區（history buffers）」演進為**三層式認知架構**。這個階層結構模仿人類認知系統（L1-L3），以平衡速度、成本與回憶容量。現在的生產級代理堆疊，傾向於在向量儲存之上依賴專用的記憶層（Mem0、Zep、Letta、Cognee），而不是自行打造。

<a id="table-of-contents"></a>
## 目錄

- [三層式階層](#the-three-tiered-hierarchy)
- [第 1 層：工作記憶（Context）](#tier-1-working-memory-context)
- [第 2 層：情節記憶（Events）](#tier-2-episodic-memory-events)
- [第 3 層：語意記憶（Knowledge）](#tier-3-semantic-memory-knowledge)
- [記憶整合模式](#memory-consolidation-patterns)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="the-three-tiered-hierarchy"></a>
## 三層式階層

| 層級 | 類型 | 人類類比 | 技術 | 延遲 |
|------|------|---------------|------------|---------|
| **L1** | 工作記憶 | 即時焦點 | Context Window / KV Cache | <50ms |
| **L2** | 情節記憶 | 過往經驗 | Vector DB / Local Graph | 100-300ms |
| **L3** | 語意記憶 | 一般知識 | Global Graph / SQL / Mem0 | >500ms |

---

<a id="tier-1-working-memory-context"></a>
## 第 1 層：工作記憶（Context）

L1 是模型的**主動焦點**。
- **Context Window**：目前前沿模型（Claude Opus 4.7、Claude Sonnet 4.6、GPT-5.5、Gemini 3.1 Pro）可達 128K - 2M tokens。
- **KV Cache**：儲存預先計算 key 與 value 的 GPU「RAM」。
- **管理策略**：**Sliding Windows** 與 **Prefix Caching**（vLLM/PagedAttention）。
- **冗餘說明**：我們只在 L1 保留最近的幾輪對話與關鍵系統指令。

---

<a id="tier-2-episodic-memory-events"></a>
## 第 2 層：情節記憶（Events）

L2 儲存本次工作階段，或此使用者過去工作階段中「先前發生了什麼」。
- **儲存**：向量資料庫（Pinecone、Weaviate、Qdrant）。
- **檢索**：語意搜尋。如果使用者問「上週二我們談了什麼？」，L2 會提供答案。
- **模式**：**Experience Replay**。Agent 會檢索過去成功的決策軌跡，以引導當前決策。

---

<a id="tier-3-semantic-memory-knowledge"></a>
## 第 3 層：語意記憶（Knowledge）

L3 儲存**不可變事實**與**習得規則**。
- **Knowledge Graphs**：儲存關係（例如：`User` -- `WORKS_FOR` --> `Company_X`）。
- **Mem0**：一種受管服務，會抽取事實（例如：「User likes Dark Mode」），並讓它們能在全域使用。
- **真實性錨定**：當 L1 與 L2 提供互相衝突的資訊時，L3 會作為「事實基準（Ground Truth）」。

---

<a id="memory-consolidation-patterns"></a>
## 記憶整合模式

記憶會透過**整合（Consolidation）**在各層之間移動：
1. **抽取**：在 session 結束時，由 LLM「Reviewer」從 L1 抽取事實。
2. **索引**：事實會儲存在 L2（作為向量）與 L3（作為圖節點）。
3. **衰退**：未被再次強化的舊記憶會從 L2 移至冷儲存（L3）或被刪除。

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-why-not-just-use-a-2m-token-context-window-for-all-memory-l1-l3"></a>
### 問：為什麼不直接用 2M token 的 context window 來承載所有記憶（L1-L3）？

**強而有力的回答：**
雖然可行，但這在**經濟上與認知上都沒有效率**。 
1. **成本**：每一輪都呼叫帶有 1M+ tokens 的模型，成本遠高於使用 RAG 召回內容的 10K token 呼叫。
2. **注意力稀釋**：即使是「長上下文（Long Context）」模型，「Lost in the Middle」仍然存在。如果 context 被不相關的歷史對話塞滿，模型對*當前*任務的推理能力就會下降。
3. **延遲**：由於 KV cache 載入，TTFT（Time to First Token）會隨 context 大小增加而擴大。 
資深等級的架構會使用**策略式檢索（Strategic Retrieval）**，讓 context window 保持精簡且聚焦。

<a id="q-how-do-you-handle-privacy-leakage-in-tier-3-global-semantic-memory"></a>
### 問：你如何處理第 3 層（Global Semantic Memory）中的「隱私外洩（Privacy Leakage）」？

**強而有力的回答：**
第 3 層（Semantic Memory）必須**依 Namespace 分片（Sharded by Namespace）**。每位使用者或每個組織，都會在 vector DB 與 Knowledge Graph 中擁有唯一的 `namespace_id`。我們在資料庫層實作**RLS（Row Level Security）**。此外，我們會在整合步驟中加入**PII-Scrubbing Layer**，以確保敏感資料（密碼、PII）絕不會從暫時性的 L1 context 進入持久化的 L3 知識儲存。

---

<a id="references"></a>
## 參考資料
- Park 等人，"Generative Agents"（2023/2025 Context）
- OpenAI，"Context Window Optimization"（2025）
- Mem0 Documentation，"Dynamic Memory Management"（2025）

---

*下一篇：[短期 Context 管理](02-short-term-context.md)*

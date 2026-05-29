
<a id="long-term-memory"></a>
# 長期記憶

長期記憶（L2 與 L3）提供跨工作階段的持久性。生產環境堆疊已從簡單的「History RAG」演進為**多重表徵儲存**（Multi-Representation Stores），結合 Vector、Graph 與 Relational 資料。專用記憶服務（Zep、Mem0、Letta、Cognee）如今也能直接封裝這些儲存層，並內建對話摘要、實體擷取與時間感知能力。

<a id="table-of-contents"></a>
## 目錄

- [情節記憶（敘事）](#episodic-memory-the-personal-log)
- [語意記憶（知識）](#semantic-memory-the-fact-store)
- [混合式 Vector-Graph 儲存](#hybrid-vector-graph-storage)
- [記憶修剪與衰退](#memory-pruning-and-decay)
- [隱私與多租戶](#privacy-and-multi-tenancy)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="episodic-memory-the-personal-log"></a>
## 情節記憶：個人日誌

情節記憶會儲存**軌跡**（Trajectories）：事件序列及其結果。
- **資料結構**：`(Timestamp, Interaction_ID, Trajectory_Summary, Embedding)`。
- **背後原因**：如果一個 agent 上個月曾透過特定工具序列成功建立 React 元件，那麼今天再次被要求建立類似元件時，它就應該能「回想起」那次成功經驗。
- **實作說明**：我們儲存*摘要*以供檢索，並將*原始日誌*存放於冷儲存（S3/GCS）中，以利鑑識分析。

---

<a id="semantic-memory-the-fact-store"></a>
## 語意記憶：事實儲存庫

語意記憶會儲存關於實體的**已發現事實**。
- **實體識別**：使用「Fact Extraction Agent」來解析每一次使用者回合。
- **三元組範例**：
  - `(User_1, HAS_PREFERENCE, Dark_Mode)`
  - `(Company_X, USES_SDK, Stripe)`
- **技術**：Knowledge Graph（Neo4j、AWS Neptune）結合 relational tagging。

---

<a id="hybrid-vector-graph-storage"></a>
## 混合式 Vector-Graph 儲存

資深工程師會使用**GraphRAG 風格的記憶**。
- **Vector Search** 會找出「相關」節點。
- **Graph Traversal** 會找出「連接」節點。
- **優勢**：如果我搜尋「Project Alpha」，vector search 會找到這個名稱，但 graph traversal 還能找出 10 位開發者、截止日期，以及相關聯的程式碼儲存庫。

---

<a id="memory-pruning-and-decay"></a>
## 記憶修剪與衰退

如果記憶無限制成長，它就會成為負擔。
- **時間衰退**：除非經常被存取，較舊的記憶會逐漸失去其「相關性分數」。
- **整合**：將 10 次關於「billing」的獨立互動，合併成一個高品質的摘要節點。
- **明確遺忘**：遵循 GDPR 的「被遺忘權」，刪除與某個使用者 ID 關聯的所有情節與語意叢集。

---

<a id="privacy-and-multi-tenancy"></a>
## 隱私與多租戶

> [!CAUTION]
> **跨工作階段洩漏**是全域記憶中的頭號安全風險。
> 請確保 `user_id` 是你 vector DB metadata 中的強制分區鍵。絕不要依賴 LLM 依使用者來過濾結果。

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-how-do-you-choose-between-a-vector-db-and-a-knowledge-graph-for-long-term-memory"></a>
### 問：對於長期記憶，你會如何在 Vector DB 與 Knowledge Graph 之間做選擇？

**強而有力的回答：**
我會將**Vector DB**用於**情節脈絡**（非結構化日誌、過往對話），因為我需要基於語意的「模糊」匹配。我會將**Knowledge Graph**用於**結構化語意知識**（關係、屬性、階層），因為我需要可「確定性」遍歷的能力。生產系統會採用**混合式**方法：vector index 指向 graph ID，讓系統先找到正確的「起始節點」，再透過遍歷取得高精度脈絡。

<a id="q-what-is-catastrophic-forgetting-in-the-context-of-learned-agentic-memory"></a>
### 問：在習得式 agentic memory 的脈絡中，什麼是「災難性遺忘」？

**強而有力的回答：**
在微調過的 agents 中，災難性遺忘指的是新的訓練資料覆蓋掉舊知識。在**Agentic Memory（基於 RAG）**中，它指的是**索引過載**。如果一個 agent 將 1,000 條低品質的新「事實」加入記憶，檢索精度就會下降，實際上等同於它「忘記」了較舊、品質更高的事實，因為那些事實被噪音掩埋了。我們會用**品質加權檢索**（Quality-Weighted Retrieval）來緩解這個問題：來自 supervisor 且具有高「驗證分數」的記憶，會比原始日誌獲得更高權重。

---

<a id="references"></a>
## 參考資料
- Neo4j. "Knowledge Graphs for Generative AI" (2025)
- Pinecone. "The Managed Memory Layer" (2025)
- GraphRAG. "Reasoning over Relationships" (2024/2025)

---

*下一篇：[使用 Mem0 的 Agentic Memory](04-agentic-memory-mem0.md)*

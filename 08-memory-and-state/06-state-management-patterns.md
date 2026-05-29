
<a id="state-management-patterns"></a>
# 狀態管理模式

AI 系統中的狀態管理，已從簡單的「session」演進為**具狀態代理圖（Stateful Agent Graphs）**。管理代理「心智」的流程與持久化，與 LLM 本身一樣關鍵：這也是 LangGraph 成為以 LangChain 建構之代理的預設控制流程執行階段（runtime）的主要原因之一。

<a id="table-of-contents"></a>
## 目錄

- [狀態物件](#the-state-object)
- [狀態機（LangGraph）](#state-machines-langgraph)
- [檢查點（Checkpointing）與續跑（Resume）](#checkpointing-and-resume)
- [平行狀態（Fork/Join）](#parallel-state-forkjoin)
- [時間旅行（狀態重寫）](#time-travel-state-rewriting)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="the-state-object"></a>
## 狀態物件

「State」是代理 session 的**單一真實來源（Single Source of Truth）**。
```python
class AgentState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    plan: list[str]
    current_task: str
    tool_results: dict[str, Any]
    user_context: dict[str, Any]
    iteration_count: int
```
**最佳實務**：State 應盡可能保持**嚴格型別化（Strictly Typed）**與**僅附加（Append-Only）**，以避免在長時間執行迴圈中發生資料遺失。

---

<a id="state-machines-langgraph"></a>
## 狀態機（LangGraph）

業界已逐漸收斂到使用**循環圖（Cyclic Graphs）**（狀態機）。
- **節點（Nodes）**：接收 state 並回傳更新的函式。
- **邊（Edges）**：根據 state 值決定下一個節點的條件邏輯（例如：`if state['error'] -> goto 'recovery_node'`）。

---

<a id="checkpointing-and-resume"></a>
## 檢查點（Checkpointing）與續跑（Resume）

在正式環境中，代理可能會執行數分鐘甚至數小時。
- **持久化層（Persistence Layer）**：每次 state 更新都會儲存到 DB（Postgres/Redis）。
- **韌性（Resiliancy）**：如果伺服器當機，orchestrator 會取回最後的 `checkpoint_id`，並從中斷處精準恢復執行。
- **使用者體驗（UX）**：這讓**非同步代理（Asynchronous Agents）**成為可能，使用者會先收到一則「我正在處理」訊息，並在 10 分鐘後於 state 變成「完成（Complete）」時收到通知。

---

<a id="parallel-state-forkjoin"></a>
## 平行狀態（Fork/Join）

面對複雜任務時，我們會對 state 進行**Fork**。
1. **Fan-out**：將 state 傳送給 3 個子代理（例如 Researcher A、B、C）。
2. **Fan-in（Join）**：由一個「Manager」代理接收三者輸出，並將它們合併回主 state 物件。

---

<a id="time-travel-state-rewriting"></a>
## 時間旅行（狀態重寫）

如同在 HITL 章節中提到的，狀態管理允許**人為介入（Human Intervention）**。
- 開發者可以瀏覽 session 歷史、找到某個「bad turn」、編輯該特定時間點的 state 物件，並從那個位置**重新執行（Re-run）**圖流程。

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-why-use-a-graph-based-state-machine-langgraph-instead-of-a-simple-while-loop-for-agents"></a>
### 問：為什麼代理要使用「圖式（Graph-based）」狀態機（LangGraph），而不是簡單的「while 迴圈（While loop）」？

**強力回答：**
while 迴圈（While loop）是**不透明且脆弱（Opaque and Brittle）**的。你無法輕鬆視覺化其邏輯，而錯誤處理也會變成一團巢狀 if-statements。圖式（Graph-based）方法則是**可觀測且模組化（Observable and Modular）**的。你可以將整體流程完整視覺化（例如用 Mermaid 圖）、對個別節點做單元測試，並且只要新增邊，就能實作像是「回溯（Backtracking）」或「平行執行（Parallel execution）」這類複雜能力。它也讓**狀態持久化（State Persistence）**變得非常簡單，因為 framework 會處理節點之間的儲存／載入。

<a id="q-how-do-you-prevent-state-bloat-in-long-running-agent-sessions"></a>
### 問：你如何防止長時間執行的代理 session 出現「狀態膨脹（State Bloat）」？

**強力回答：**
我們會使用**狀態修剪（State Pruning）**與**訊息摘要（Message Summarization）**。我們不會讓整個 `tool_results` 字典一路貫穿整張圖，而是在子任務完成後就加以裁剪。對於 `messages` 清單，我們會使用一個專門的「Summarizer Node」，每 10 個 turn 執行一次，將歷史壓縮成精簡的上下文區塊，確保在保持 state 物件反應靈敏的同時，不會觸及 token 限制。

---

<a id="references"></a>
## 參考資料
- LangChain. 「LangGraph: Multi-Agent Workflows」(2024/2025)
- Temporal.io. 「Stateful AI Agents at Scale」(2025)
- AWS Bedrock. 「Managing Long-Running Agent Sessions」(2025)

---

*下一節：[第 09 節：框架與工具](../09-frameworks-and-tools/01-langchain-deep-dive.md)*

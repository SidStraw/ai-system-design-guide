<a id="langgraph-orchestration"></a>
# LangGraph 協調編排

LangGraph 是建立有狀態、多代理系統的 **事實標準**。它在 2025 年底到達 v1.0，並因企業對其 graph-based runtime 的採用，在 2026 年初於 GitHub stars 上超越 CrewAI。不同於單純的 chains，LangGraph 允許 **Cycles**、**State Persistence**，以及 **Human-in-the-Loop** 介入。

<a id="table-of-contents"></a>
## 目錄

- [Graph 哲學](#philosophy)
- [循環式 vs. 非循環式工作流](#cyclic)
- [LangGraph 中的狀態管理](#state)
- [持久化與 Checkpointing](#persistence)
- [多代理協調模式](#multi-agent)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="the-graph-philosophy"></a>
## Graph 哲學

在 2023 年，agents 是「黑盒子」。
如今，agents 是 **Graphs**。
Graph 由以下元素組成：
- **Nodes**：Python functions（LLM、工具，或資料處理步驟）。
- **Edges**：節點之間的路徑。
- **Conditional Edges**：根據 **State** 決定路徑的邏輯。

---

<a id="cyclic-vs-acyclic"></a>
## 循環式 vs. 非循環式

標準 LangChain 是 **非循環式**（Sequential）。
LangGraph 是 **循環式**。
- **迴圈的力量**：agent 可以先嘗試某個工具、看到錯誤後，再 **循環返回**「Thinking」節點重新嘗試。這正是 **ReAct** 模式的基礎。

---

<a id="state-management"></a>
## 狀態管理

**State Schema** 是 graph 的「心智」。
```python
class GraphState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    plan: list[str]
    is_secure: bool
```
**細節重點**：將 `Annotated` 與 `add_messages` 搭配使用，可讓 graph 對歷史記錄採用 **Append** 而不是覆寫，因而保留完整的推理軌跡。

---

<a id="persistence-and-checkpointing"></a>
## 持久化與 Checkpointing

2025 年底的 LangGraph 使用 **以 thread 為基礎的持久化**。
- **概念**：每個 session 都有一個 `thread_id`。
- **優勢**：如果使用者兩天後回來，agent 仍記得自己在多步 workflow 中停在哪一個確切位置。
- **Time-Travel**：開發者可以從先前狀態「重新執行」某個特定 thread，以除錯失敗原因。

---

<a id="multi-agent-patterns"></a>
## 多代理模式

| 模式 | 說明 | 案例 |
|---------|-------------|------------|
| **Supervisor** | 一個「Manager」指揮各個專精 worker。 | Research Team |
| **Peer-to-Peer**| agents 直接彼此交接任務。 | Customer Support |
| **Hierarchical**| Graphs 之中再包含 Graphs（巢狀 graphs）。 | Enterprise Engineering |

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-why-use-langgraph-instead-of-openais-assistant-api"></a>
### 問：為什麼使用 LangGraph，而不是 OpenAI 的「Assistant API」？

**理想回答：**
因為它具備 **控制力與可攜性**。Assistant API 是黑盒子：你看不到精確的 prompts，也無法控制邏輯閘點。LangGraph 則是 **White Box framework**。我可以使用任何模型（OpenAI、Claude、Llama 3.3），精準控制工具何時被呼叫，並在步驟之間插入我自己的驗證邏輯。更重要的是，LangGraph 是 **Open Source**，可在本地或 on-prem 執行，這對許多企業安全需求至關重要。

<a id="q-how-do-you-handle-state-overload-in-a-graph-with-20-nodes"></a>
### 問：你會如何處理擁有 20+ 個節點的 graph 中的「State Overload」？

**理想回答：**
我們會使用 **State Narrowing**。我們不會把完整的全域 state 傳給每個節點，而是為各個 sub-graphs 定義專門的子狀態。我們也會使用 **Trim Runnables**，在訊息歷史送進 LLM 前先裁剪，確保不浪費 tokens，同時又把「Truth」保留在 persistence layer 中。

---

<a id="references"></a>
## 參考資料
- LangChain Team. "LangGraph: Multi-Agent Workflows at Scale" (2025)
- Anthropic. "Building Resilient Agents with State Machines" (2025)
- OpenSource AI. "Cycles and the Future of Agency" (2024 Tech Report)

---

*下一步：[LangSmith Observability](03-langsmith-observability.md)*

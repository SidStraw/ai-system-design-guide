<a id="multi-agent-orchestration"></a>
# 多代理協作編排

複雜系統很少只由單一代理構成，而是由一組各司其職的專門代理協同運作。編排模式已從「盲目管理者」演進為 **Hierarchical Supervisors**、**Dynamic Swarms**，以及由 A2A 等互通協定支援的 **Cross-Vendor Agent Networks**。Gartner 預測，到 2026 年底，40% 的企業應用程式將具備任務導向的 AI agents，而 2025 年初時此比例仍低於 5%。

<a id="table-of-contents"></a>
## 目錄

- [為什麼要使用多代理？](#why)
- [Supervisor 模式](#supervisor)
- [管線模式](#pipeline)
- [Swarms 與點對點（P2P）](#swarms)
- [圖式編排（2026 主流模式）](#graph-orchestration)
- [透過 A2A 進行跨廠商代理編排](#cross-vendor)
- [2026 多代理框架版圖](#framework-landscape)
- [代理團隊中的狀態管理](#state)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="why"></a>
<a id="why-multi-agent"></a>
## 為什麼要使用多代理？

單一代理若配有 50 個工具，會面臨 **Cognitive Load**。
1. **專精化**：例如「Code Agent」可以使用針對 Python 最佳化的模型，而「Search Agent」則使用針對 RAG 最佳化的模型。
2. **平行化**：多個代理可以同時處理彼此獨立的子任務。
3. **解耦評估**：你可以將「Writer Agent」與「Researcher Agent」分開評估。

---

<a id="supervisor"></a>
<a id="the-supervisor-pattern-hierarchical"></a>
## Supervisor 模式（階層式）

這是截至 2026 年最常見的企業模式。

- **Supervisor**：高推理能力模型（Claude Opus 4.7、GPT-5.5 reasoning、Gemini 3.1 Pro Deep Think），負責拆解使用者提示並委派給 worker。
- **Workers**：快速、具成本效益的模型（Claude Haiku 4.5、Gemini 3.1 Flash、GPT-5.5-mini），負責實際執行工作。
- **Reviewer**：獨立代理，依照 supervisor 的原始計畫驗證整合後的輸出。

**架構**：LangGraph 仍是實作這類具狀態感知之階層迴圈的主流框架。到了 2026 年，Claude Agent SDK、Google ADK 與 Microsoft Agent Framework 也都原生支援此模式。

---

<a id="swarms"></a>
<a id="swarms-the-openai-pattern"></a>
## Swarms（OpenAI 模式）

Swarms 在 2024 年底開始普及，重點在於「Handoffs」。

- 一個代理會將對話「交接」給另一個代理。
- **核心概念**：`Handoff(TargetAgent)`。
- **優點**：沒有中央「Manager」瓶頸，對話能在各個專門實體之間自然流動。

---

<a id="graph-orchestration"></a>
<a id="graph-based-orchestration-2026-dominant-pattern"></a>
## 圖式編排（2026 主流模式）

到了 2026 年，架構演進明顯轉向 **graph-based orchestration**：代理工作流會被建模為具有型別狀態的有向圖。

<a id="why-graphs-won"></a>
### 為什麼 Graph 勝出

- **明確控制流程**：節點是代理或函式；邊定義狀態轉移，包含條件分支與迴圈
- **可視化**：團隊能以圖表形式檢視並除錯工作流
- **具狀態感知**：帶型別的 state object 會在圖中流動，因此可支援 checkpoint 與 resume

<a id="framework-support"></a>
### 框架支援

| Framework | Graph Model | 關鍵差異化優勢 |
|-----------|-------------|----------------|
| **LangGraph** (24k stars) | 命令式 DAG 搭配 typed state | 最成熟、社群最廣 |
| **Google ADK** (17k stars) | 內建 A2A 的 agent graphs | 原生整合 Google Cloud |
| **Microsoft Agent Framework** | Workflow graphs（sequential、concurrent、handoff） | 統一 .NET + Python，具企業治理能力 |
| **Claude Agent SDK** | 以 supervisor 為核心的階層樹 | 內建工具（bash、editor），可直接上生產 |

<a id="the-paperclip-pattern-hierarchical-agents-at-scale"></a>
### Paperclip 模式（大規模階層式代理）

2026 年一項值得注意的發展是 **Paperclip**（於 2026 年 3 月上線後三週內即達到 44,900 GitHub stars）。它採用階層式模型：CEO agent 接收高階目標、將其拆解後委派給 manager agents，再由 manager 生成並協調 worker agents。這個模式展示了深度階層化的多代理樹，如何處理複雜的真實世界任務。

> *已於 2026 年 5 月驗證。*

---

<a id="cross-vendor"></a>
<a id="cross-vendor-agent-orchestration-via-a2a"></a>
## 透過 A2A 進行跨廠商代理編排

**Agent-to-Agent (A2A) protocol**（見 [Tool Use and MCP](03-tool-use-and-mcp.md#a2a)）帶來一種新的多代理模式：**cross-vendor orchestration**。在 A2A 出現之前，多代理系統通常要求所有代理共用同一個 framework 與 runtime。現在則可以：

1. **Agent Discovery**：編排器透過 **Agent Cards**（描述能力的 JSON metadata）尋找專家代理
2. **Task Delegation**：編排器透過 HTTP/SSE 將結構化任務傳送給遠端代理
3. **Async Progress**：遠端代理串流回傳狀態更新；編排器也能平行委派其他代理
4. **Result Collection**：最終產物會回傳並整合進編排器的 state

**生產實例**：某採購系統中，編排器（LangGraph）將合規檢查委派給專門代理（Google ADK）、將庫存查詢交給透過 MCP 連接的工具、再將合約產生交給 CrewAI crew——各方分別透過 A2A 與 MCP 溝通。

> *已於 2026 年 5 月驗證。來源：a2a-protocol.org*

---

<a id="framework-landscape"></a>
<a id="the-2026-framework-landscape-for-multi-agent"></a>
## 2026 多代理框架版圖

如今每一家主要 AI 實驗室都推出了自己的 agent framework。以下是截至 2026 年 5 月的多代理編排版圖：

| Framework | Provider | Multi-Agent Model | 狀態 |
|-----------|----------|-------------------|------|
| **LangGraph** | LangChain | Graph-based、彈性最高 | Production (126k stars) |
| **Claude Agent SDK** | Anthropic | 具內建工具的 supervisor trees | GA（Python + TypeScript） |
| **Google ADK** | Google | Graph-based，原生支援 A2A | GA（Python、TS、Java、Go） |
| **Microsoft Agent Framework** | Microsoft | Workflows + group chat patterns | RC 1.0（2026 年 2 月），GA 於 2026 Q2 |
| **OpenAI Agents SDK** | OpenAI | 具 guardrails 的 handoff-based swarms | GA（Python + TypeScript） |
| **CrewAI** | CrewAI Inc. | 以角色為基礎的 crews 搭配 Flows | v1.13（60%+ Fortune 500） |
| **Smolagents** | HuggingFace | 輕量、開源 | 持續活躍開發中 |

**關鍵趨勢**：沒有單一框架能同時在四種多代理模式（supervisor、swarm、pipeline、debate）中都做到最好。團隊越來越常混搭框架——例如用 LangGraph 處理複雜編排，再搭配 CrewAI 處理面向商業使用者的自動化流程。

> *已於 2026 年 5 月驗證。*

---

<a id="state"></a>
<a id="state-management"></a>
## 狀態管理

多代理系統最大的挑戰是 **Shared Blackboard**。

1. **Local State**：只有特定代理看得到的上下文。
2. **Global State**：所有代理都可見的共享記憶（例如最終草稿）。
3. **Write Conflicts**：兩個代理同時嘗試修改同一份 Global State。
   - **最佳實務**：使用 **Transactional Handoffs**。只有在代理「擁有」鎖時，才能寫入全域狀態。

---

<a id="peer-to-peer-p2p-debate"></a>
## 點對點（P2P）辯論

在高準確性任務（例如法律或醫療）中，我們會使用 **Agentic Debate**。
- **Agent A**：提出答案。
- **Agent B**：嘗試找出 Agent A 答案中的缺陷。
- **Agent A**：根據 B 的批評修正答案。
- **結果**：收斂出比任何單一代理都更高品質的結果。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-what-are-the-main-failure-modes-of-a-supervisor-multi-agent-architecture"></a>
### Q：Supervisor 多代理架構的主要失敗模式有哪些？

**強答：**
主要失敗模式是 **Decomposition Failure**。如果 Supervisor agent 將任務拆成邏輯上不一致、或隱含相依關係未被看見的子任務，workers 就會對*錯誤的問題*給出正確答案。標準修正方法是 **Iterative Planning**：Supervisor 必須在 workers 開始執行前，先取得對「子任務可行性」的確認。另一個失敗模式是 **Context Dilution**，也就是全域狀態被大量 worker logs 塞滿，導致 Supervisor 失去對「Big Picture」的掌握。

<a id="q-how-do-you-choose-between-a-sequence-of-chains-and-a-multi-agent-graph"></a>
### Q：你會如何在「Sequence of Chains」與「Multi-Agent Graph」之間做選擇？

**強答：**
當任務是線性且具決定性的（例如 Extract -> Translate -> Summarize）時，我會使用 **Sequence of Chains**。當任務是 **Non-Linear**，或需要 **Conditional Loops** 時，我會使用 **Multi-Agent Graph**（例如 LangGraph）。舉例來說，如果「Translate」步驟可能失敗，並需要回到「Extract」取得更多上下文，靜態 chain 會失效，但 graph 可以透過回送至較早節點來自我修正。

<a id="q-when-would-you-use-a2a-for-multi-agent-orchestration-versus-keeping-all-agents-in-a-single-framework"></a>
### Q：你何時會在多代理編排中使用 A2A，而不是把所有代理都留在單一框架裡？

**強答：**
當團隊擁有所有代理、它們共用同一個 runtime，且代理呼叫之間的低延遲很關鍵時，我會把代理留在單一框架內。當需要跨越 **組織或供應商邊界** 時，我才會引入 A2A——例如編排器需要委派給另一個團隊維護的合規代理，或整合第三方專用代理（例如法律審查服務）。A2A 會增加 HTTP 開銷，但能帶來 **vendor neutrality**、**independent scaling**，以及透過 Agent Cards 進行 **capability discovery**。經驗法則是：同團隊、同框架；不同團隊或不同廠商，就用 A2A。

---

<a id="references"></a>
## 參考資料
- Wu et al.《AutoGPT: An Autonomous GPT-4 Experiment》（歷史背景／2025 更新）
- Li et al.《Camel: Communicative Agents for 'Mind' Exploration》（2023/2025）
- OpenAI.《Swarms Framework》（2024/2025）
- Google.《Agent2Agent Protocol》（2025/2026）
- Gartner.《Predicts 2026: AI Agent Market》（2025）
- Andrew Ng.《Agentic Design Patterns》（2025/2026）

---

*下一篇：[Agent Memory and State](05-agent-memory-and-state.md)*

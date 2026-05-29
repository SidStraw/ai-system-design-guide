<a id="microsoft-agent-framework-crewai-and-the-agent-sdk-landscape"></a>
# Microsoft Agent Framework、CrewAI 與 Agent SDK 生態版圖

過去一年，多代理框架的生態明顯整併。Microsoft **淘汰了 AutoGen**，並將其與 Semantic Kernel 合併為統一的 **Microsoft Agent Framework**（RC 1.0 於 2026 年 2 月推出；GA 目標為 2026 年 Q2）。CrewAI 成熟至 v1.13，具備企業級功能，並宣稱已有 60% 以上的 Fortune 500 公司採用。與此同時，各大 AI 實驗室也都推出了自己的 agent SDK：Anthropic 的 Claude Agent SDK、OpenAI 的 Agents SDK，以及 Google 的 ADK。

<a id="table-of-contents"></a>
## 目錄

- [CrewAI：管理者視角](#crewai)
- [Microsoft Agent Framework（AutoGen 的後繼者）](#microsoft-agent-framework)
- [Agent SDK 生態版圖](#agent-sdk-landscape)
- [Swarms 與點對點通訊](#swarms)
- [框架比較矩陣](#comparison)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="crewai-the-manager-perspective"></a><a id="crewai"></a>
## CrewAI：管理者視角

CrewAI 是圍繞 **Process** 這個概念建立的。
- **基於角色的代理**：你會定義「Researcher」、「Writer」與「Manager」。
- **任務**：具有明確輸出內容的具體目標。
- **流程編排**：Sequential、Hierarchical，或 Consensual（基於共識）。

<a id="crewai-flows"></a>
### CrewAI Flows

CrewAI 的 **Flows** 在經典 Crew 模式之上增加了一層 **state-machine layer**：

```python
from crewai.flow.flow import Flow, listen, start

class ContentFlow(Flow):
    @start()
    def research_topic(self):
        # Returns research output
        return research_crew.kickoff({"topic": self.state["topic"]})
    
    @listen(research_topic)
    def write_article(self, research):
        # Triggered after research completes
        return writing_crew.kickoff({"research": research})
    
    @listen(write_article)
    def publish(self, article):
        # Final step
        return publisher.publish(article)
```

<a id="crewai-v113-highlights"></a>
### CrewAI v1.13 重點

CrewAI v1.13 標誌著它朝向企業正式環境就緒邁出關鍵一步：

- **Enterprise SSO**：已完整文件化的企業部署 Single Sign-On
- **RBAC Improvements**：附有完整權限對照矩陣的 Role-Based Access Control
- **GPT-5 Compatibility**：修正 OpenAI 的 GPT-5 與新版 o-series 模型不再支援 `stop` 參數所帶來的相容性問題
- **A2A Task Execution**：以結構化、可決定的方式進行 Agent-to-Agent 動態任務委派
- **NVIDIA NemoClaw Integration**：適用於安全企業部署的基礎設施層級 policy enforcement
- **RuntimeState RootModel**：為複雜工作流程提供統一狀態序列化

**使用情境**：CrewAI + Flows 最適合 **商業流程自動化**（內容產線、資料分析工作流程）這類結構明確的場景。CrewAI 表示目前約支援了 20 億次 agentic executions。

> *已於 2026 年 5 月驗證。來源：docs.crewai.com/en/changelog*

---

<a id="microsoft-agent-framework-autogens-successor"></a><a id="microsoft-agent-framework"></a>
## Microsoft Agent Framework（AutoGen 的後繼者）

<a id="the-merger-autogen--semantic-kernel--agent-framework"></a>
### 合併：AutoGen + Semantic Kernel = Agent Framework

Microsoft 在 2025 年底停止將 AutoGen 作為獨立產品，並把它與 Semantic Kernel 合併成統一的 **Microsoft Agent Framework**。Release Candidate 1.0 於 2026 年 2 月推出，GA 目標為 2026 年 Q2。

**這次合併整合了：**
- **來自 AutoGen**：針對單代理與多代理對話模式（group chat、round-robin、handoffs）的簡潔抽象
- **來自 Semantic Kernel**：企業級 session management、type safety、filters、telemetry，以及廣泛的 model/embedding 支援

<a id="migration-path"></a>
### 遷移路徑

AutoGen 仍持續接收 bug fixes 與 security patches，但 **所有新功能都只會進入 Agent Framework**。Microsoft 也提供了官方 migration guide。如果是新專案，請直接使用 Agent Framework。

<a id="key-capabilities"></a>
### 核心能力

```python
# Microsoft Agent Framework: Graph-based workflow
from agent_framework import Agent, Workflow, HandoffStep

planner = Agent("Planner", model="gpt-5.5", system_message="Decompose tasks.")
executor = Agent("Executor", model="gpt-5.5-mini", system_message="Execute sub-tasks.")

workflow = Workflow(
    steps=[
        HandoffStep(from_agent=planner, to_agent=executor),
    ],
    state_management="session",  # Built-in session persistence
)
```

**框架亮點：**
- **統一 .NET 與 Python**：兩種語言採用相同的程式設計模型
- **Graph-based Workflows**：以明確控制支援 sequential、concurrent、handoff 與 group chat 模式
- **State Management**：為長時間執行與 human-in-the-loop 場景提供穩健的 session-based persistence
- **MCP Support**：原生整合 Model Context Protocol 以存取工具
- **Multi-provider**：支援 OpenAI、Azure OpenAI、Anthropic、Google 與本地模型

> *已於 2026 年 5 月驗證。來源：learn.microsoft.com/en-us/agent-framework*

---

<a id="the-agent-sdk-landscape"></a><a id="agent-sdk-landscape"></a>
## Agent SDK 生態版圖

現在每一家主要 AI 實驗室都推出了自己的 agent framework。以下是截至 2026 年 5 月的生態概況：

<a id="claude-agent-sdk-anthropic"></a>
### Claude Agent SDK（Anthropic）

Claude Agent SDK（由 Claude Code SDK 更名而來）提供了與 Claude Code 相同的 tools、agent loop 與 context management，並可作為 Python 與 TypeScript 函式庫使用。

- **內建工具**：檔案讀取、命令執行、程式碼編輯——代理無需自訂工具實作即可立即運作
- **Supervisor pattern**：具備 delegation 能力的階層式代理樹
- **部署**：支援 AWS Bedrock、Google Vertex AI 與 Azure
- **截至 2026 年 5 月**：Python v0.1.48+、TypeScript v0.2.71+

<a id="openai-agents-sdk"></a>
### OpenAI Agents SDK

OpenAI 的輕量級框架，使用原生 Python/TypeScript 結構來建立多代理工作流程：

- **基於 handoff**：代理透過 `Handoff(TargetAgent)` 彼此委派——不需要中央 supervisor
- **Guardrails**：內建輸入驗證與安全檢查
- **MCP integration**：原生支援 MCP server tools
- **Realtime agents**：以 gpt-realtime-1.5 支援語音代理

<a id="google-agent-development-kit-adk"></a>
### Google Agent Development Kit（ADK）

Google 的框架雖為 Google 生態最佳化，但本身不綁定特定模型：

- **多語言**：Python、TypeScript、Java、Go（截至 2026 年 5 月皆已達 1.0+）
- **原生 A2A**：內建 Agent-to-Agent protocol 支援跨供應商編排
- **Vertex AI integration**：可部署至 Agent Engine Runtime 進行代管
- **Graph-based**：代理工作流程以有向圖建模

> *已於 2026 年 5 月驗證。*

---

<a id="swarms-and-p2p"></a><a id="swarms"></a>
## Swarms 與 P2P

這兩個框架（以及更廣泛的 SDK 生態）都已採用 **Swarm Patterns**。
- **Handoff**：代理不再依賴中央 supervisor，而是把對話「交接」給最相關的專家。
- **範例**：某個「Sales Agent」察覺使用者提出的是技術問題，便會把對話串交接給「Support Agent」。

---

<a id="framework-comparison-matrix"></a><a id="comparison"></a>
## 框架比較矩陣

| 功能 | CrewAI | MS Agent Framework | LangGraph | Claude Agent SDK | OpenAI Agents SDK | Google ADK |
|---------|--------|-------------------|-----------|-----------------|-------------------|------------|
| **核心抽象** | Task/Process/Flow | Workflow/Agent | State/Graph | Supervisor/Tools | Handoff/Agent | Agent Graph |
| **架構** | 宣告式 + State Machine | Graph Workflows | 命令式 DAG | 階層樹 | Swarm Handoffs | Directed Graph |
| **易用性** | 高 | 中 | 低 | 中 | 高 | 中 |
| **控制力** | 低到中 | 中到高 | 高 | 中 | 低到中 | 中到高 |
| **最適合** | 商業自動化 | 企業 .NET/Python | 複雜編排 | Coding/Tool Agents | 快速多代理 | Google Cloud AI |
| **多語言** | Python | .NET + Python | Python | Python + TS | Python + TS | Python、TS、Java、Go |
| **MCP 支援** | 有 | 有 | 透過 tools | 原生 | 有 | 有 |
| **A2A 支援** | 透過 extension | 規劃中 | 透過 tools | 否（直接） | 否（直接） | 原生 |

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-when-would-you-use-crewai-instead-of-langgraph"></a>
### Q：你會在什麼情況下使用 CrewAI 而不是 LangGraph？

**有力的回答：**
**速度 vs. 精準度**。當我需要非常快速地為標準流程（例如內容生成或資料分析）建立一個代理團隊時，我會用 **CrewAI**。它提供了開箱即用的高階抽象，例如「Planning」與「Cooperation」。當我需要對每一次狀態轉移、人類多輪介入觸發，或不適合「角色扮演團隊」隱喻的複雜錯誤恢復邏輯進行 **細粒度控制** 時，我就會切換到 **LangGraph**。

<a id="q-microsoft-retired-autogen-in-favor-of-the-agent-framework-how-does-this-affect-existing-autogen-deployments"></a>
### Q：Microsoft 以 Agent Framework 取代 AutoGen，這會如何影響既有的 AutoGen 部署？

**有力的回答：**
AutoGen 仍持續接收 bug fixes 與 security patches，因此現有部署不會立刻失效。不過，**所有新功能的開發** 都已轉移到 Agent Framework。遷移路徑已有完整文件：AutoGen 的 `AssistantAgent` 對應到 Agent Framework 的 `Agent` 類別，`GroupChat` 對應到新的 `Workflow` 模式，而 Semantic Kernel 的企業功能（session management、telemetry、filters）現在也都已原生整合。遷移的主要好處在於 **統一的 .NET 與 Python 支援**，以及能對多代理執行路徑提供明確控制的 **graph-based workflows**。對新專案而言，應直接從 Agent Framework 開始。

<a id="q-how-do-you-prevent-infinite-loops-where-agents-keep-talking-to-each-other-without-solving-the-task"></a>
### Q：你要如何避免代理彼此不停對話卻始終無法解決任務的「Infinite Loops」？

**有力的回答：**
我們會使用 **Termination Conditions** 與 **Max Conversational Turns**。我們也會實作一個「Critic Agent」，其唯一工作就是偵測對話是否停滯。如果 Critic 偵測到循環，它就會觸發 user proxy 中斷流程，或強制將 group chat manager 切換到另一條推理路徑。我們也會監控 **Token Velocity**：如果一組代理在 2 分鐘內用了 100K tokens 卻沒有進展，就會自動終止該 session。到了 2026 年，像 Microsoft Agent Framework 與 LangGraph 這類框架也已提供內建的 workflow timeouts 與 state checkpointing，讓 loop detection 更系統化。

---

<a id="references"></a>
## 參考資料
- CrewAI. "The Multi-Agent Process Engine" (2025/2026, v1.13)
- Microsoft. "Agent Framework Overview" (2026) — learn.microsoft.com/en-us/agent-framework
- Microsoft. "AutoGen to Agent Framework Migration Guide" (2026)
- Anthropic. "Claude Agent SDK" (2026) — platform.claude.com/docs/en/agent-sdk
- OpenAI. "Agents SDK Documentation" (2026)
- Google. "Agent Development Kit" (2026) — google.github.io/adk-docs
- OpenAI Swarm. "Lightweight Multi-Agent Orchestration" (2024 tech report)

---

*下一篇：[框架選擇指南](08-framework-selection-guide.md)*

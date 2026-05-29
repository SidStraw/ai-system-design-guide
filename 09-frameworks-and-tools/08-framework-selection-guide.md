<a id="framework-selection-guide"></a>
# 框架選擇指南

過去一年，AI 框架的生態明顯整併。各大 AI 實驗室現在都推出了 agent SDK，Microsoft 也把 AutoGen 與 Semantic Kernel 合併成統一的 Agent Framework，而互通協定（MCP、A2A）則已成為基本配備。本指南提供一份**決策矩陣**，協助你根據正式環境需求、團隊專業能力與系統規模來選擇技術堆疊。

<a id="table-of-contents"></a>
## 目錄

- [框架生態版圖](#landscape)
- [決策矩陣](#matrix)
- [自建、採購還是使用框架](#build-vs-buy)
- [應避免的反模式](#anti-patterns)
- [Staff 級建議](#recommendation)
- [面試問題](#interview-questions)

---

<a id="the-framework-landscape"></a><a id="landscape"></a>
<a id="the-framework-landscape"></a>
## 框架生態版圖

<a id="orchestration-agent-frameworks"></a>
### 編排與 Agent 框架

| 框架 | 層級 | 主要價值 | 關鍵弱點 |
|-----------|------|---------------|--------------|
| **LangGraph** | L1（核心） | 精準的狀態控制、graph-based | 複雜、學習曲線陡峭 |
| **DSPy** | L1（核心） | 可靠性與最佳化 | 前期成本高（Training） |
| **LlamaIndex**| L2（資料） | 進階檢索（RAG） | 邏輯彈性 |
| **CrewAI** | L3（應用） | 商業流程速度、企業 RBAC | 會隱藏失敗 |
| **MS Agent Framework** | L1（企業） | 統一 .NET + Python，取代 AutoGen + SK | RC 狀態（GA 目標 2026 年 Q2） |

<a id="agent-sdks-lab-specific"></a>
### Agent SDK（各實驗室專屬）

| 框架 | 層級 | 主要價值 | 關鍵弱點 |
|-----------|------|---------------|--------------|
| **Claude Agent SDK** | L1（Agent） | 內建工具、正式環境 agent loop | 需要 Anthropic API |
| **OpenAI Agents SDK** | L1（Agent） | 輕量 handoffs、guardrails | 偏向 OpenAI |
| **Google ADK** | L1（Agent） | 多語言、原生 A2A + Google Cloud | 偏向 Google 生態 |

<a id="coding-agents"></a>
### Coding Agents

| 框架 | 層級 | 主要價值 | 關鍵弱點 |
|-----------|------|---------------|--------------|
| **Claude Code** | L1（Coding） | 自主式 CLI coding agent | 需要 Anthropic API |
| **Cursor / Windsurf** | L2（IDE） | 緊密的 IDE + agent 整合 | 封閉來源基礎設施 |
| **OpenHands** | L2（Coding） | 開源自主代理 | 需要自行託管 |

> **2026 年 4 月註記**：Semantic Kernel 已不再作為獨立框架列出，因為它已合併進 Microsoft Agent Framework。現有的 SK 使用者應規劃遷移。

---

<a id="the-decision-matrix"></a><a id="matrix"></a>
<a id="the-decision-matrix"></a>
## 決策矩陣

**請用以下邏輯來選擇你的技術堆疊：**

<a id="core-orchestration"></a>
### 核心編排
1. **這是一個純 RAG 應用嗎？** → **LlamaIndex**。
2. **它是否需要長時間執行的狀態／Human-in-the-loop？** → **LangGraph**。
3. **高可靠性（99%+）與跨模型可攜性是否至關重要？** → **DSPy**。
4. **你是 C#/.NET 企業團隊嗎？** → **Microsoft Agent Framework**（取代 Semantic Kernel + AutoGen）。
5. **你是在為商務使用者打造高階自動化嗎？** → **CrewAI + Flows**。

<a id="agent-sdks-choose-based-on-your-primary-model-provider"></a>
### Agent SDK（依主要模型供應商選擇）
6. **在 Claude / Anthropic API 上建構 agents？** → **Claude Agent SDK**（Python/TS，內建 file/code/command tools）。
7. **在 OpenAI API 上建構 agents？** → **OpenAI Agents SDK**（輕量 handoffs、guardrails、MCP 支援）。
8. **在 Google Cloud / Gemini 上建構 agents？** → **Google ADK**（原生 A2A、Vertex AI 部署、多語言）。
9. **需要跨供應商的 agent 通訊？** → 在上述任一框架之上使用 **A2A protocol**。

<a id="coding-agents"></a>
### Coding Agents
10. **你是否在進行自主式、檔案系統層級的 coding 任務？** → **Claude Code**（CLI）或 **Cline**（VS Code）。
11. **需要可搭配任何 LLM 的開源 coding agent？** → **OpenHands**（Docker）。
12. **想要最佳的 IDE AI 體驗？** → **Cursor**（closed）或 **Windsurf**（Codeium）。

---

<a id="build-vs-buy-vs-framework"></a><a id="build-vs-buy"></a>
<a id="build-vs-buy-vs-framework"></a>
## 自建、採購還是使用框架

作為 Staff Engineer，你必須抵抗 **Framework Bloat**。

- 當框架能解決**非平凡的電腦科學問題**時才**使用框架**（例如狀態持久化、Bayesian prompt optimization、Vector-Graph linking）。
- 當你只是要對 LLM 發出簡單呼叫時，就應**自行實作（薄封裝）**。對單輪代理而言，框架增加的延遲、更新震盪與除錯成本，通常不值得。

---

<a id="anti-patterns-to-avoid"></a><a id="anti-patterns"></a>
<a id="anti-patterns-to-avoid"></a>
## 應避免的反模式

1. **Framework Tunnelling**：試圖把複雜邏輯流程硬塞進不支援它的框架中（例如用純 RAG 函式庫來做 coding agent）。
2. **The Golden Hammer**：只是因為 LangChain 很熱門就使用它，明明一支 50 行的 Python script 會更快、更便宜。
3. **忽略可觀測性**：部署任何框架時沒有加上 LLOps 層（LangSmith/Phoenix）。

---

<a id="staff-level-recommendation"></a><a id="recommendation"></a>
<a id="staff-level-recommendation"></a>
## Staff 級建議

對於現代、正式等級的 agentic system：
- **編排**：LangGraph（處理狀態與迴圈）或 Microsoft Agent Framework（適合 .NET 團隊）。
- **Agent SDK**：依你的模型供應商選擇——Claude Agent SDK（Anthropic）、Agents SDK（OpenAI）、ADK（Google）。它們都支援以 MCP 存取工具。
- **最佳化**：DSPy（用來為不同模型層級編譯 prompts）。
- **檢索**：LlamaIndex（適合多階段 RAG）。
- **可觀測性**：LangSmith（用於 tracing 與 evaluation）。
- **跨供應商 agents**：A2A protocol，用於跨組織邊界的 agent-to-agent 協調。
- **自主 coding**：Claude Code（CLI）或 Cline（VS Code），適合檔案層級編輯任務。
- **開放式 coding agent**：OpenHands，適合自架或整合進 CI pipeline。

**2026 年的洞察：**
1. Agentic coding 工具（Claude Code、Cursor、OpenHands）不是編排框架的替代品——它們是**新的類別**，運作在檔案系統層級，位於 LLM API 之上、應用程式邏輯之下。
2. 協定層已經成熟：**MCP 用於 agent-to-tool**、**A2A 用於 agent-to-agent**，正逐漸成為基礎設施標準，而不是可有可無的附加功能。你的架構應設計成同時支援兩者。
3. 每家實驗室都推出自己的 agent SDK，會帶來**供應商鎖定風險**。可透過 MCP 處理工具存取（可跨 SDK 攜帶），以及透過 A2A 處理 agent 協調（供應商中立）來降低風險。

> *更新於 2026 年 5 月。*

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-why-do-we-see-a-trend-towards-programming-dspy-instead-of-prompting"></a>
### Q：為什麼我們看到趨勢正從「Prompting」走向「Programming」（DSPy）？

**有力的回答：**
**工業化**。Prompt engineering 比較像「Alchemy」：不穩定，也無法擴展。透過 DSPy 這類 Frameworks 來 Programming LLMs，可以讓我們把 AI 當成一門**軟體工程學科**來對待。我們可以套用 CI/CD、單元測試（metrics）與自動化最佳化。這讓 AI 從「Nondeterministic Magic」轉變為大型分散式系統中的**可預測元件**，而這正是任何 mission-critical 正式環境的必要條件。

<a id="q-if-you-had-to-build-a-system-that-works-across-openai-anthropic-and-local-llama-models-how-would-you-architect-it"></a>
### Q：如果你要建構一個能同時運作於 OpenAI、Anthropic 與本地 Llama 模型的系統，你會怎麼設計架構？

**有力的回答：**
我會把 **DSPy** 用於 prompt 層，把 **LangGraph** 用於編排層。DSPy 的 **Signatures** 讓我可以把任務定義與模型的具體行為解耦。接著我會使用 **Universal Model Gateway**（例如 LiteLLM 或內部 proxy）來處理不同的 API 格式。對於工具存取，我會使用 **MCP**——它與模型無關，因此無論底層使用哪個 LLM backend，都可以共用同一套 MCP servers。如果我需要跨團隊的 agent 協調，就會在邊界層使用 **A2A**。這套堆疊能確保我若因成本或延遲因素而必須從 GPT-4o 切換到 Claude Sonnet 4，不需要重寫 50 個 prompts；只需要重新編譯或更新設定即可。

<a id="q-with-every-ai-lab-shipping-its-own-agent-sdk-claude-agent-sdk-openai-agents-sdk-google-adk-how-do-you-avoid-vendor-lock-in"></a>
### Q：當每一家 AI 實驗室都推出自己的 agent SDK（Claude Agent SDK、OpenAI Agents SDK、Google ADK）時，你要如何避免供應商鎖定？

**有力的回答：**
關鍵在於**把編排層與模型層分離**。我會使用像 LangGraph 這種與框架無關的 orchestrator，或是為核心 workflow logic 建立一層薄的自訂封裝。特定模型的 SDK 很適合拿來快速原型，或用在你已經確定只會採用單一供應商的情況；但對正式的多供應商系統，我會把模型互動放在抽象層之後（LiteLLM gateway 或 DSPy signatures）。對於工具存取，**MCP** 提供可攜性——同一個 MCP server 可搭配任何 SDK。對於 agent 協調，**A2A** 提供供應商中立的 agent-to-agent 通訊。實務規則是：在葉節點（個別 agent 實作）使用實驗室專屬 SDK，但讓編排圖維持供應商中立。

---

<a id="references"></a>
## 參考資料
- Google Cloud. "Enterprise Generative AI Reference Architecture" (2025)
- Gartner. "Magic Quadrant for AI Application Frameworks" (2025)
- Gartner. "Predicts 2026: 40% of Enterprise Apps to Feature AI Agents" (2025)
- Thoughtworks. "Technology Radar: The Rise of Agentic Frameworks" (Nov 2024/2025)
- Microsoft. "Agent Framework Overview" (2026)
- Anthropic. "Claude Agent SDK" (2026)
- Google. "Agent Development Kit" (2026)
- OpenAI. "Agents SDK" (2026)

---

*新章節：[第 10 節：文件處理](../10-document-processing/01-ocr-and-layout.md)*

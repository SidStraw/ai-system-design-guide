<a id="agent-fundamentals"></a>
# 代理基礎

代理是由 LLM 驅動、從「聊天」進一步走向「自主解題」的系統。這個定義已從簡單的 ReAct 迴圈，轉向使用內建「System 2」思考的 **Closed-Loop Reasoning Systems**（Claude Opus 4.7 extended thinking、GPT-5.5 reasoning、DeepSeek-R2、Gemini 3.1 Pro Deep Think）。

<a id="table-of-contents"></a>
## 目錄

- [代理公式](#formula)
- [System 1（LLM）vs. System 2（推理模型）](#systems)
- [代理層級（自主光譜）](#levels)
- [核心元件](#components)
- [代理生命週期](#lifecycle)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="formula"></a>
<a id="the-agent-formula"></a>
## 代理公式

現代的 agent 常被描述為：
`Agent = Reasoning Model + Tool Use + Persistent Memory + Environment Feedback`

**細節補充**：在 2023 年，agents 多半只是包在 chat model 外層的「wrappers」。如今的 agents 越來越 **Integrated**。前沿模型（Claude Opus 4.7、具 reasoning 的 GPT-5.5、DeepSeek-R2）已把「Thinking」流程融入 pre-training，讓 agent loop 更穩定，也更不容易「卡住」。

---

<a id="systems"></a>
<a id="system-1-vs-system-2-thinking"></a>
## System 1 vs. System 2 思考

設計 agent 時，必須選對合適的「Thinking Mode」：

| 模式 | 認知類型 | 類比 | 當前常見組合 |
|------|----------|------|--------------|
| **System 1** | 快速、直覺、反應式 | 反射 | Claude Haiku 4.5 / Sonnet 4.6 / GPT-5.5-mini / Gemini 3.1 Flash |
| **System 2** | 緩慢、邏輯、規劃型 | 深思熟慮 | Claude Opus 4.7 / GPT-5.5 reasoning / DeepSeek-R2 / Gemini 3.1 Pro Deep Think |

**設計模式**：將 System 1 模型用於「Fast UI」與「Routing」，將 System 2 模型用於「Decision Gates」與「Complex Planning」。

---

<a id="levels"></a>
<a id="agency-levels"></a>
## 代理層級

不是每個自主系統都算是「Agent」。我們會依照 **Level of Agency** 來分類：

1. **L0: Scripted Chains**：固定順序（例如標準 LangChain）。
2. **L1: Tool-Enabled**：模型會選工具，但不會規劃。
3. **L2: ReAct Agent**：簡單的「Thought -> Action -> Observation」迴圈。
4. **L3: Autonomous Planner**：把目標拆成一張子任務圖。
5. **L4: Ambient Agent**：在背景中運作，只在必要時介入。

---

<a id="components"></a>
<a id="core-components"></a>
## 核心元件

<a id="1-the-reasoning-model-the-executive"></a>
### 1. 推理模型（The Executive）
代理的大腦。它決定「通往成功的路徑」。

<a id="2-tools-the-limbs"></a>
### 2. 工具（The Limbs）
讓代理能對世界施加影響的介面（API、Browser、DB 等）。
> [!Note]
> **Model Context Protocol (MCP)** 現在已是工具互通的業界標準，Anthropic、OpenAI、Google、Microsoft 與 AWS 都已採用。其治理在 2025 年 12 月移交給 Linux Foundation 的 Agentic AI Foundation。

<a id="3-memory-the-experience"></a>
### 3. 記憶（The Experience）
- **短期**：context window（KV Cache）。
- **長期**：Vector DB 或持久狀態（例如 Mem0）。

---

<a id="lifecycle"></a>
<a id="the-agent-lifecycle"></a>
## Agent 生命週期

1. **Intake**：接收使用者目標。
2. **Decomposition**：把目標拆成子步驟。
3. **Execution**：呼叫工具並處理結果。
4. **Reflection**：評估這次 observation 是否讓 agent 更接近目標。
5. **Completion**：為使用者整合出最終證據與答案。

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-why-is-a-reasoning-model-like-claude-opus-47-or-gpt-55-with-extended-thinking-better-for-agency-than-a-standard-llm"></a>
### Q：為什麼「Reasoning Model」（例如 Claude Opus 4.7 或帶有 extended thinking 的 GPT-5.5）比標準 LLM 更適合做 agent？

**強答案：**
標準 LLM（System 1）是根據模式匹配來預測*下一個 token*。當它們在工具呼叫中遇到錯誤時，常會幻想出一個修正方式，而不是承認失敗。Reasoning Models 會在推論時使用 **Chain-of-Thought (CoT)**。它們會在輸出回應前，先經過多輪隱藏的「思考」。對 agent 來說，這代表更高的 **Path Reliability**——模型較不容易陷入無限迴圈，也較不會重複嘗試相同的失敗動作，因為它已在內部先模擬過失敗情境。

<a id="q-how-do-you-prevent-agentic-drift-in-long-running-tasks"></a>
### Q：你會如何避免長時間任務中的「Agentic Drift」？

**強答案：**
當子步驟把 agent 帶離原始目標太遠，以至於失去上下文時，就會發生 Agentic Drift。標準解法是 **Goal Anchoring**：將「Original Objective」固定成一則 system message，並使用 **Secondary Observer Model**（較小、較便宜的模型）來針對每個 agent 動作與原始目標的對齊程度打分。如果分數低於門檻，就強制 agent 從根節點重新規劃。

---

<a id="references"></a>
## 參考資料
- Kahneman, D.《Thinking, Fast and Slow》（套用到 AI，2025）
- OpenAI.《Learning to Reason with LLMs》（2024）
- DeepSeek.《R1: Cold-Start Data for Reasoning》（2025）

---

*下一章：[Reasoning Loops：ReAct and Beyond](02-reasoning-loops-react-and-beyond.md)*

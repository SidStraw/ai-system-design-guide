<a id="human-in-the-loop-patterns"></a>
# Human-in-the-Loop 模式

沒有任何 agent 是 100% 可靠的。**Human-in-the-Loop（HITL）** 是高風險環境中確保安全與準確性的橋梁。生產環境堆疊早已超越「Approval Button」，進展到 **Co-Reasoning** 與 **Interrupt-Based Steering**，並在 LangGraph（interrupt+resume）與 Microsoft Agent Framework 等框架中原生提供。

<a id="table-of-contents"></a>
## 目錄

- [HITL 光譜](#spectrum)
- [中斷與斷點](#interrupts)
- [時間旅行除錯（狀態編輯）](#time-travel)
- [協同推理（共享 Scratchpad）](#co-reasoning)
- [基於信心的升級處理](#escalation)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="spectrum"></a>
<a id="the-hitl-spectrum"></a>
## HITL 光譜

| 模式 | Agent 自主性 | 人類角色 | 最適用場景 |
|---------|---------------|------------|----------|
| **Human-in-command** | 低 | 主導每一步 | 高風險法律／醫療 |
| **Human-as-filter** | 中 | 核准／編輯最終輸出 | 內容生成 |
| **Human-as-backup** | 高 | 僅在出錯時介入 | 客服支援 |
| **Human-on-the-loop** | 最高 | 完成後稽核日誌 | 高吞吐量分析 |

---

<a id="interrupts"></a>
<a id="interrupts-and-breakpoints"></a>
## 中斷與斷點

現代架構（LangGraph、Microsoft Agent Framework）使用 **Deterministic Breakpoints**。

- **模式**：系統被硬編碼為在呼叫特定敏感工具前先「暫停」（例如 `execute_purchase` 或 `delete_user`）。
- **決策**：環境等待使用者送出 `approve` 或 `reject` 訊號。
- **狀態保存**：在人工採取行動前，agent 的推理狀態會先「凍結」在 DB 中。

---

<a id="time-travel"></a>
<a id="time-travel-debugging-state-editing"></a>
## 時間旅行除錯（狀態編輯）

標準 agent 是「單向」的。如果它在 Step 3 犯錯，整個 session 往往就毀了。
- **創新點**：**State Injection**。人工審查者可以「回到」Step 3 的狀態，編輯 agent 的 observation 或 thought，然後再「恢復」執行。
- **影響**：這讓人類可以在不從零開始的情況下，把 agent 從錯誤路徑「引導」回來。

---

<a id="co-reasoning"></a>
<a id="co-reasoning-shared-scratchpads"></a>
## 協同推理（共享 Scratchpad）

此時人類不再只是「裁判」，而是成為 **「夥伴」**。
- Agent 會向人類展示它的 **Scratchpad**（Internal Thinking）。
- 常見表述方式：*「我打算使用 Tool A，因為 Fact B。你覺得這樣合理嗎？」*
- **好處**：在推理錯誤真正轉化成動作之前就先攔截。

---

<a id="escalation"></a>
<a id="confidence-based-escalation"></a>
## 基於信心的升級處理

透過支援「Logprobs」或內建推理步驟的模型，我們可以計算 **不確定性分數**。

- 如果分數超過門檻，agent 就會**自動暫停**並通知人工操作員。
- **例子**：某個 agent 嘗試處理複雜的帳務爭議時，意識到使用者意圖含糊不清。它會停下來並說：*「我無法百分之百確定該如何處理這個特定退款案例。請稍候，我找一位人類專家來查看。」*

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-how-do-you-design-an-hitl-system-that-doesnt-fatigue-the-human-operator"></a>
### 問：你如何設計一個不會讓人工操作員「疲勞」的 HITL 系統？

**強回答：**
我們會使用 **Threshold Tuning**。不是每個動作都要求核准。我們只在以下情況觸發 HITL：1）高風險的「Writing」工具；2）低信心的推理步驟；3）違反業務所設定「Policy」的動作。此外，我們會提供人類 **Contextual Summary**——不是整份日誌，而是一句話的「Diff」，說明 agent 想做什麼。這可以把「審查認知負荷」從數分鐘降到幾秒鐘。

<a id="q-what-is-the-over-reliance-risk-in-hitl-and-how-do-you-mitigate-it"></a>
### 問：HITL 中的「Over-Reliance」風險是什麼？你如何緩解？

**強回答：**
Over-reliance 發生在人類開始不看日誌就直接點擊「Approve」時。我們會用 **Forced Review Checkpoints** 來緩解（例如要求人工至少修改提議計畫中的一個字），或使用 **Synthetic Error Injections**（故意在 1% 的時間裡顯示一份「錯誤」計畫，看看他們是否能抓出來）。如果他們通過這個「陷阱」，就能繼續；如果失敗，就會被標記為需要額外訓練。

---

<a id="references"></a>
## 參考資料
- Wu et al.「Co-reasoning: Human-AI Collaboration Patterns」(2025)
- LangChain.「Human-in-the-loop in LangGraph」(2024/2025)
- Anthropic.「Designing for Safety and Human Oversight」(2024)

---

*下一章：[Agentic 安全與沙箱隔離](09-agentic-security-and-sandboxing.md)*

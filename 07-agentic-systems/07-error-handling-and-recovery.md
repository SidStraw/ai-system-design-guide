<a id="error-handling-and-recovery"></a>
# 錯誤處理與復原

Agent 會以非決定性的方式失敗。錯誤處理已從「Try-Catch 區塊」演進為 **Agentic Self-Correction** 與 **Stateful Rollbacks**，而 LangGraph 與 Microsoft Agent Framework 等框架也已原生提供 checkpoint/resume 原語。

<a id="table-of-contents"></a>
## 目錄

- [Agent 失敗的分類](#fail-types)
- [自我修正迴圈](#correction)
- [有狀態回滾（Checkpointing）](#rollbacks)
- [修復「卡在迴圈中」的方法](#stuck)
- [優雅降級](#degradation)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="fail-types"></a>
<a id="taxonomy-of-agent-failures"></a>
## Agent 失敗的分類

1. **虛構工具**：呼叫不存在的工具。
2. **Schema 違規**：把錯誤的參數傳給真實存在的工具。
3. **環境錯誤**：工具存在，但外部 API 當機。
4. **邏輯停滯**：Agent 重複執行同一個失敗動作（The ReAct Loop of Death）。

---

<a id="correction"></a>
<a id="self-correction-loops"></a>
## 自我修正迴圈

錯誤現在被視為 **資訊 Token**。

- **模式**：當工具失敗時，錯誤訊息不只是被記錄；它會作為 prompt 回饋給模型：*「動作因錯誤 X 而失敗。請反思為何會發生，並提供替代策略。」*
- **Reasoning Models**（Claude Opus 4.7 extended thinking、GPT-5.5 reasoning、DeepSeek-R2）：這些模型特別擅長處理這種情況，因為它們會在隱藏的 Chain-of-Thought 中「內化」錯誤，因此一次復原成功率高得多。

---

<a id="rollbacks"></a>
<a id="stateful-rollbacks-checkpointing"></a>
## 有狀態回滾（Checkpointing）

對於長時間運行的 agent，Step 9 出錯不應該讓整個專案崩潰。

- **Checkpoints**：高可靠度系統（使用 LangGraph 或類似框架）會在每次成功的工具呼叫後，把「狀態快照」存進 DB。
- **回滾**：如果 agent 進入邏輯停滯，supervisor agent 可以把 **common-state 重設**到 Step 5——也就是最後一個「安全」狀態——並強制改走另一條路徑。

---

<a id="stuck"></a>
<a id="the-stuck-in-a-loop-fix"></a>
## 修復「卡在迴圈中」的方法

無限迴圈是 agentic systems 中排名第一的成本黑洞。

**解法**：**基於計數器的介入**。
1. 如果同一個 `(Tool, Args)` tuple 在單一 session 中出現 3 次，orchestrator 就會中斷模型。
2. 它會注入強制性的 **「Pivot Instruction」**：*「你已經嘗試搜尋 'X' 三次。這條路走不通。你**必須**改用其他工具，或承認自己卡住了。」*

---

<a id="degradation"></a>
<a id="graceful-degradation"></a>
## 優雅降級

如果高推理能力 agent（Claude Opus 4.7、GPT-5.5 reasoning）持續失敗，我們就退回到：
- **簡化版 Agent**：使用更小、工具更少但更可靠的模型。
- **僅 RAG 模式**：停用動作，只根據知識庫提供概念性回答。
- **人工升級處理**：（見下一章）。

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-why-is-traditional-exception-handling-trycatch-insufficient-for-agentic-systems"></a>
### 問：為什麼傳統的「Exception Handling」（Try/Catch）不足以應對 Agentic Systems？

**強回答：**
在傳統軟體中，exception 是「停止」命令。但在 agentic system 中，模型是「駕駛員」。如果系統只是停止，使用者任務就失敗了。我們使用的是 **Error Injection**，而不是 Exception Handling。我們在平台層捕捉 exception，並把它轉換成模型可見的 **Synthesized Observation**。這讓模型能夠圍繞失敗進行「推理」。TRY/Catch 只能修好程式碼；Error Injection 則讓模型有機會修正**計畫**。

<a id="q-how-do-you-handle-silent-failures-where-the-tool-returns-200-ok-but-the-data-is-wrong"></a>
### 問：你如何處理「Silent Failures」（工具回傳 200 OK，但資料是錯的）？

**強回答：**
Silent failure 是最危險的。我們會實作 **Output Validation Agents**。對於關鍵步驟，我們不會直接接受工具輸出，而是把輸出送到一個「Verifier Agent」（通常是較小、較快的模型），它唯一的工作是檢查：*「這個工具輸出真的有回答原本的查詢嗎？」* 如果 Verifier 回答「沒有」，就會像遇到硬錯誤一樣觸發自我修正迴圈。

---

<a id="references"></a>
## 參考資料
- LangGraph.「Persistence and Checkpointing」(2025)
- Shinn et al.「Reflexion: Learning from Errors」(2024 update)
- Microsoft.「Managing Hallucinations in Agentic Systems」(2025)

---

*下一章：[Human-in-the-Loop 模式](08-human-in-the-loop-patterns.md)*

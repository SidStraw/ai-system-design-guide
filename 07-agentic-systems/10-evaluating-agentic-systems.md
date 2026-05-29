<a id="evaluating-agentic-systems"></a>
# 評估 Agentic Systems

評估 agent 與評估 RAG 有根本上的不同。RAG 著重於「Accuracy」，而 Agents 著重的是 **「Reliability」、「Efficiency」與「Safety」**。生產環境中的 agent eval 依賴 **Trajectory Benchmarks** 與 **LLM-as-Judge** 來檢驗多步推理，而 Langfuse、LangWatch、Braintrust 與 Arize Phoenix 等工具則提供原生的 trace-level scoring。

> [!NOTE]
> 關於標準 RAG 評估（Retrieval 與 Generation metrics），請參閱 [06-retrieval-systems/09-advanced-retrieval-patterns.md](../06-retrieval-systems/09-advanced-retrieval-patterns.md) 與第 14 節。本章特別聚焦於 agent 的*執行路徑*。

<a id="table-of-contents"></a>
## 目錄

- [評估轉向](#shift)
- [Trajectory Benchmarks（黃金標準）](#benchmarks)
- [關鍵指標：成功、成本與時間](#metrics)
- [用於步驟品質的 LLM-as-Judge](#judge)
- [生產環境評估（A/B Testing Agents）](#production)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="shift"></a>
<a id="the-evaluation-shift"></a>
## 評估轉向

| 指標 | RAG App | Agentic App |
|--------|---------|-------------|
| **評估單位** | 單一回應 | **Trajectory**（所有步驟） |
| **成功標準**| Groundedness/Faithfulness | 任務完成 / 邏輯健全性 |
| **複雜度** | 低（文字相似度） | 高（工具狀態驗證） |

---

<a id="benchmarks"></a>
<a id="trajectory-benchmarks"></a>
## Trajectory Benchmarks

現代 eval 會為 **「通往結果的路徑」**打分。
1. **最佳路徑**：解決任務所需的最短工具序列。
2. **Agent 路徑**：實際採取的步驟。
3. **分數**：`Efficiency = (Optimal Steps / Agent Steps)`。分數為 `0.2` 代表 agent 走了很多冤枉路，或陷入過度迴圈。

**常見 Benchmark**：
- **SWE-bench**：修復 GitHub issue（Code Agency）。
- **WebArena**：導覽選單與表單（Browser Agency）。
- **GAIA**：一般工具使用任務（Assistant Agency）。

---

<a id="metrics"></a>
<a id="key-metrics"></a>
## 關鍵指標

<a id="1-task-success-rate-tsr"></a>
### 1. 任務成功率（TSR）
最終狀態正確的任務比例。
> [!IMPORTANT]
> 在資深 production 場景中，透過「錯誤路徑」得到「正確答案」，分數仍然是 0。

<a id="2-action-success-rate-asr"></a>
### 2. 動作成功率（ASR）
回傳有效資料（而非錯誤或 hallucination）的單次工具呼叫比例。

<a id="3-unit-cost-per-task"></a>
### 3. 每項任務的單位成本
每個已完成目標的總 token 成本 + 基礎設施成本（Sandboxes、API calls）。

---

<a id="judge"></a>
<a id="llm-as-judge-for-step-quality"></a>
## 用於步驟品質的 LLM-as-Judge

我們使用更強的模型（Claude Opus 4.7、GPT-5.5 reasoning）來審查較小型 agent 的 **Reasoning Log**。
- **思考品質**：agent 使用 Tool X 的邏輯，是否真的從 Observation Y 推導而來？
- **冗餘檢查**：agent 是否重複進行了剛做過的搜尋？
- **回饋迴圈**：這個「Judge」輸出接著會被用於 **DPO（Direct Preference Optimization）**，以對齊 agent 未來的行為。

---

<a id="production"></a>
<a id="production-evaluation"></a>
## 生產環境評估

生產團隊會使用 **Shadow Execution**。
1. **V1 Agent** 回應使用者。
2. **V2（實驗版）Agent** 在「Hidden Sandbox」中執行相同查詢。
3. **比較**：我們比較兩條 trajectory。如果 V2 能穩定地用更少步驟解決任務，且沒有安全違規，就將它升級到 production。

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-how-do-you-evaluate-an-agent-when-the-environment-is-non-deterministic-eg-the-web"></a>
### 問：當環境具有非決定性（例如 web）時，你如何評估 agent？

**強回答：**
我們會使用 **Mock Environments** 或 **Snapshotted States**。在高擬真測試中，我們使用容器化瀏覽器，並在每次測試執行時重設為乾淨狀態。然後將 agent 的 trajectory 與 **Reference Trace** 比較。如果環境真的是 live 的，我們就使用 **State-Based Verification**——不比較文字，而是檢查外部世界的狀態（例如：「資料庫中是否新增了一列具有正確值的資料？」）。

<a id="q-why-is-meandering-taking-too-many-steps-a-critical-failure-in-staff-level-agent-design"></a>
### 問：為什麼「Meandering」（走太多步）在 Staff-level Agent 設計中屬於關鍵失敗？

**強回答：**
Meandering 會導致三種失敗：1）**成本**：每一步都是一次 LLM 呼叫；2）**延遲**：每一步都會增加 2-5 秒；3）**熵**：trajectory 越長，agent 遇到奇怪 edge case 並觸發 hallucination 的機率越高。標準修正方式是 **Step Budgets**：如果 agent 在 10 步內無法解決任務，我們就終止它並升級給人工處理，以避免「Token Leak」。

---

<a id="references"></a>
## 參考資料
- Jimenez et al.「SWE-bench: Can Language Models Resolve Real-World GitHub Issues?」(2024/2025 update)
- Microsoft Research.「AgentBench: A Comprehensive Benchmark for AI Agents」(2024)
- RAGAS.「Agentic Evaluation Module」(2025)

---

*下一章：[Memory Architectures](../08-memory-and-state/01-memory-architectures.md)*

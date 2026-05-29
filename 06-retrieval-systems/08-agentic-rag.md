<a id="agentic-rag"></a>
# Agentic RAG

Agentic RAG 把流程從「線性 Pipeline」推進到 **「推理迴圈（Reasoning Loop）」**。它不再只檢索一次，而是由 agent 決定 *何時* 與 *檢索什麼*，以解決一個 query。當前正式環境的主流模式包括 Self-RAG（模型發出 reflection tokens）、Corrective RAG（以 retrieval evaluator 進行 corrective routing）、Adaptive RAG（分類器決定 pipeline 深度）、對文件使用 ReAct，以及 multi-hop query decomposition。LangGraph 是最常見的有狀態迴圈 control-flow runtime；LlamaIndex Workflows 則常用於以單一 pipeline、檢索導向為主的變體。

<a id="table-of-contents"></a>
## 目錄

- [線性 RAG 與 Agentic RAG 的差異](#linear-vs-agentic-rag)
- [Self-RAG（自我反思）](#self-rag-self-reflection)
- [Corrective RAG（CRAG）](#corrective-rag-crag)
- [Multi-Hop 推理迴圈](#multi-hop-reasoning-loops)
- [Agentic 過濾與計畫修訂](#agentic-filtering-and-plan-revision)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="linear-vs-agentic-rag"></a>
## 線性 RAG 與 Agentic RAG 的差異

| 模型 | Linear RAG | Agentic RAG |
|-------|------------|-------------|
| **結構** | 預先決定的序列 | 動態迴圈 |
| **自我修正** | 無 | 高（可重新檢索） |
| **查詢複雜度**| 簡單（1 步） | 困難（多步） |
| **延遲** | 低（固定） | 可變（多輪） |

**原則**：當 query 需要的是「合成後的證據（Synthesized Proof）」，而不只是「文件匹配」時，就該用 Agentic RAG。也要替它編列預算：一個 3-4 次迭代的迴圈，端到端通常需要 8-12 秒；若你的 UX 需要 3 秒內回應，就應把簡單 query 導向快路徑（Adaptive RAG）。

---

<a id="self-rag-self-reflection"></a>
## Self-RAG（自我反思）

此模式在 2024/2025 年開始流行，**Self-RAG** 會使用「Critic Tokens」來評估自己的工作結果。

1. **Retrieve**：模型取回 Top-K chunks。
2. **Evaluate**：資訊是否相關？（CRITIC：`Relevant`）
3. **Generate**：答案是否有根據？（CRITIC：`Supported`）
4. **Iterate**：若答案沒有被支撐，模型會 *自動* 觸發更廣的搜尋。

---

<a id="corrective-rag-crag"></a>
## Corrective RAG（CRAG）

CRAG 在檢索與生成之間加入一層「可靠性層（Reliability Layer）」。

- **邏輯如下：**
  - 若 retrieval **Correct**：直接生成。
  - 若 retrieval **Ambiguous**：使用 Web-Search 工具補強。
  - 若 retrieval **Incorrect**：捨棄當前 context，改用外部搜尋或 fallback 邏輯。

---

<a id="multi-hop-reasoning-loops"></a>
## Multi-Hop 推理迴圈

對於像「收購 Figma 的那家公司，其 CEO 是誰？」這類問題，系統必須：
1. **Hop 1**：搜尋「Who acquired Figma?」（結果：Adobe）。
2. **Hop 2**：搜尋「CEO of Adobe」（結果：Shantanu Narayen）。

**Agentic 模式**：agent 會維護一個「State Object」，並在每次檢索後更新它的「Sub-goal」，直到整條推理鏈完成。

---

<a id="agentic-filtering-and-plan-revision"></a>
## Agentic 過濾與計畫修訂

現代 agent 會使用 **Sub-Step Plans**。
- 不再只做一次大型檢索，而是先寫出計畫：「我會先查內部資料庫中的 X，再查公開 API 中的 Y。」
- **修訂式規劃**：如果 Step 1 失敗，agent 會 *重寫* Step 2。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-what-is-the-reasoning-retrieval-balance-in-agentic-rag"></a>
### Q：Agentic RAG 中的「Reasoning-Retrieval Balance」是什麼？

**強答：**
Agentic 迴圈中的每一個「Reasoning turn」都會增加 token 成本與使用者延遲。正式環境工程師的目標，是找到那個「Retrieval Threshold」。我們會做 **Token-Budgeting**，例如只允許 agent 進行 3-5 個 turns，之後就強制產出最終答案。我們也會用 **Speculative Retrieval**：讓 agent 預測接下來兩步可能會做什麼，並同時替兩步檢索，以降低 round-trip latency。

<a id="q-why-does-agentic-rag-often-lead-to-higher-quality-but-lower-reliability-determinism"></a>
### Q：為什麼 Agentic RAG 常帶來更高品質，卻有較低的「Reliability」（Determinism）？

**強答：**
Agentic RAG 是非決定性的，因為模型在每一步都在「決定」要走哪條路。使用者 query 只要有微小變化，就可能讓 agent 選擇不同工具或搜尋策略，導致答案格式也不同。標準緩解方式是使用 **Constrained Agent Frameworks**（例如 LangGraph 或 DSPy），讓「可能路徑構成的圖」被嚴格定義，即使在這些路徑之間的選擇仍具有隨機性。

---

<a id="references"></a>
## 參考資料
- Asai et al. "Self-RAG: Learning to Retrieve, Generate, and Critique" (2024/2025)
- Yan et al. "Corrective Retrieval Augmented Generation (CRAG)" (2024)
- LangChain. "Agentic RAG with LangGraph" (2025)

---

*下一節：[Advanced Retrieval Patterns](09-advanced-retrieval-patterns.md)*

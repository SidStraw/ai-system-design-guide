<a id="planning-and-decomposition"></a>
# 規劃與分解

規劃是代理的「System 2」組件，讓代理能在不「迷航」的情況下解決多階段問題。生產級代理已從單純的「Chain-of-Thought」演進到 **Recursive Decomposition** 與 **Tree Search**，而以推理為原生能力的模型（Claude Opus 4.7、GPT-5.5 extended thinking、DeepSeek-R2）會在內部承擔大部分規劃工作。

<a id="table-of-contents"></a>
## 目錄

- [規劃光譜](#spectrum)
- [靜態 vs. 動態規劃](#static-vs-dynamic)
- [CoT 與 o1 推理](#cot)
- [遞迴式任務分解](#decomposition)
- [代理路徑的 Tree Search（MCTS）](#mcts)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="spectrum"></a>
<a id="the-planning-spectrum"></a>
## 規劃光譜

| Method | Strategy | Complexity | Best For |
|--------|----------|------------|----------|
| **Linear** | 一次一步 | 低 | 簡單工具 |
| **Branching** | If-Then-Else 邏輯 | 中 | 條件式流程 |
| **Hierarchical** | Master-Plan -> Sub-Plans | 高 | 軟體工程 |
| **Search-Based** | 在內部嘗試多條路徑 | 最高 | 科學研究 |

---

<a id="static-vs-dynamic"></a>
<a id="static-vs-dynamic-planning"></a>
## 靜態 vs. 動態規劃

<a id="static-plan-and-solve"></a>
### 靜態（Plan-and-Solve）
代理會先寫出 10 步驟計畫，然後嚴格照做。
- **優點**：效能高，容易平行化。
- **缺點**：脆弱。若第 2 步失敗，第 3 到第 10 步都失去意義。

<a id="dynamic-adaptive"></a>
### 動態（Adaptive）
代理會先寫計畫，但在每次工具呼叫後都會 **重新評估**。
- **最佳實務**：使用 **Checkpointed Planning**。代理在每個主要子目標後都必須把進度「提交」到 state store，這樣當計畫失敗時，才能進行復原與「Backtracking」。

---

<a id="cot"></a>
<a id="cot-and-o1-reasoning"></a>
## CoT 與 o1 推理

模型內部的「Thinking」視窗（Inference scaling）會充當 **Hidden Planner**。
- 與其使用獨立的「Planner LLM」，我們會使用推理模型（Claude Opus 4.7、GPT-5.5 extended thinking、DeepSeek-R2）生成一份「Mental Draft」。
- 接著再把這份草稿轉成由編排器執行的 **Task DAG（Directed Acyclic Graph）**。

---

<a id="decomposition"></a>
<a id="recursive-task-decomposition"></a>
## 遞迴式任務分解

對於超大型任務（例如「Build a full-stack app」），我們會使用 **Sub-Agent Spawning**。
1. **Master Agent**：把「Project」分解成「Frontend」、「Backend」與「DB」。
2. **Sub-Agents**：每個代理各自接收一個「Sub-Goal」，並再做自己的分解。
3. **Consolidation**：Master Agent 將結果合併。

**關鍵細節**：每個 sub-agent 都只會拿到 **Minimal Context**（僅限其所需資訊），以避免 token 膨脹與 hallucination。

---

<a id="mcts"></a>
<a id="tree-search-mcts"></a>
## Tree Search（MCTS）

在高風險決策中，我們會把 **Monte Carlo Tree Search (MCTS)** 放進代理迴圈內。
- 代理會「模擬」10 個可能的工具呼叫。
- 由 **Reward Model**（或另一個獨立的 LLM prompt）為每次模擬評分。
- 代理最後選擇獎勵最高的路徑。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-how-do-you-prevent-an-agent-from-infinite-recursion-during-task-decomposition"></a>
### Q：你要如何防止代理在任務分解時陷入「Infinite Recursion」？

**強答：**
我們會實作 **Decomposition Depth Limits**（通常 3 層）與 **Granularity Checks**。在生成 sub-agent 之前，我們會先問 Supervisor model：「這個任務是否已小到可以用單一工具呼叫解決？」如果是，就直接執行；如果不是，就繼續分解。我們也會使用 **Global Controller** 追蹤總「Agent Count」，避免出現遞迴炸彈（fork bomb）把 API 預算耗光。

<a id="q-why-is-plan-revision-often-more-expensive-than-plan-generation"></a>
### Q：為什麼「Plan Revision」通常比「Plan Generation」更昂貴？

**強答：**
計畫生成是一次「Fresh Start」。計畫修訂則需要 **Context Re-evaluation**——模型必須理解*已經完成了什麼*、*前一步為什麼失敗*，以及如何在不推翻先前成功成果的情況下修正它。這要求更高的「Reasoning Density」。在生產環境中，我們常會用更大的模型（例如 Sonnet 3.7 或 o1）來做 **Revision** 步驟，而初始計畫生成則交給較小的模型。

---

<a id="references"></a>
## 參考資料
- Silver et al.《Mastering the game of Go with deep neural networks and tree search》（Applied to LLMs, 2024/2025）
- Wang et al.《Self-Consistency Improves Chain of Thought Reasoning》（2022/2025 update）
- LangGraph.《Multi-Agent Planning Patterns》（2025）

---

*下一篇：[Error Handling and Recovery](07-error-handling-and-recovery.md)*

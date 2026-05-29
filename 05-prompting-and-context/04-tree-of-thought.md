<a id="tree-of-thought-tot"></a>
# Tree-of-Thought（ToT）

Tree-of-Thought（ToT）是一種進階提示架構，模型會探索多條推理路徑、評估它們，並在某條路徑走進死胡同時「回溯」。它是現代自主研究 agent 背後的藍圖。

<a id="table-of-contents"></a>
## 目錄

- [樹狀 vs. 鏈狀](#the-tree-vs-the-chain)
- [ToT 迴圈：提出、評估、搜尋](#the-tot-loop-propose-evaluate-search)
- [自我修正與回溯](#self-correction--backtracking)
- [MCTS 與 Search-as-Service](#mcts-and-search-as-service)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-tree-vs-the-chain"></a>
## 樹狀 vs. 鏈狀

雖然 **Chain-of-Thought** 是線性的（單一路徑），**Tree-of-Thought** 則允許分支。

| 特性 | Chain-of-Thought | Tree-of-Thought |
|------|------------------|-----------------|
| **拓樸** | 線性（1 條路徑） | 分支式（多條路徑） |
| **邏輯** | 序列式 | 平行 + 評估式 |
| **自我修正** | 低（commitment bias） | 高（回溯） |
| **適用情境** | 數學、簡單邏輯 | 解謎、程式架構、策略規劃 |

---

<a id="the-tot-loop-propose-evaluate-search"></a>
## ToT 迴圈：提出、評估、搜尋

一個 ToT 系統由三個模組組成：
1. **Thought Proposer**：為問題生成 3 到 5 個可能的「下一步」。
2. **State Evaluator**：替每一步打分（例如「Good」、「Maybe」、「Impossible」）。
3. **Search Algorithm**：（BFS 或 DFS）決定下一個要探索的分支。

```python
# The ToT logic (Simplified):
For each branch:
   Score = Evaluate(branch)
   If Score < Threshold:
      Prune branch (Backtrack)
   Else:
      Continue exploring
```

---

<a id="self-correction--backtracking"></a>
## 自我修正與回溯

ToT 的設計目的，正是為了克服 **Hallucination Cascades**。 
在線性鏈中，如果模型在 Step 1 出錯，後面的每一步都很可能跟著錯。在 ToT 中，「Evaluator」（可以是另一個模型，也可以是規則式檢查）會在 Step 1 就抓出錯誤，並迫使模型改試不同的起點。

---

<a id="mcts-and-search-as-service"></a>
## MCTS 與 Search-as-Service

ToT 已進一步演化成 LLM 的 **Monte Carlo Tree Search（MCTS）**。
- **Search-time Compute Scaling**：我們不再用一個大型提示，而是用 100 個小提示去「搜尋」最佳答案。
- **RAD-T（Reasoning-as-Data-Tree）**：專門的「Searcher」模型（Gemini 3.1 Pro Deep Think、GPT-5.5 extended thinking、Claude Opus 4.7）原生就接受過管理這些分支的訓練。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-when-is-tot-significantly-better-than-simple-cot"></a>
### 問：什麼情況下 ToT 會明顯優於單純的 CoT？

**強答案：**
當問題具有「大型搜尋空間」，並且需要「全域一致性」時，ToT 會更出色。例如在複雜的軟體重構中，單一路徑的 Chain-of-Thought 可能一開始看起來正確，卻在 10 步之後撞上約束衝突。用 ToT 時，模型可以提出 3 種不同的重構模式，評估每一種對整個程式庫的影響，並在真的寫任何程式碼之前，先捨棄會導致循環依賴的方案。

<a id="q-what-is-the-main-drawback-of-tree-of-thought-in-a-consumer-facing-app"></a>
### 問：在面向消費者的應用中，Tree-of-Thought 的主要缺點是什麼？

**強答案：**
主要缺點是 **成本與延遲呈指數成長**。探索 3 個分支、深度達 5，可能需要 15 到 20 次獨立的 LLM 呼叫。在消費型 app 中，這可能代表單次查詢要延遲 30 秒，成本達到 $0.50。標準緩解方式是採用「Hybrid Model」：將 ToT 用於高風險的離線任務（例如產生 golden datasets 或安全稽核），再把這些成果蒸餾到可用於即時互動的快速線性模型中。

---

<a id="references"></a>
## 參考資料
- Yao et al. "Tree of Thoughts: Deliberate Problem Solving with Large Language Models" (2023)
- Silver et al. "Mastering the Game of Go without Human Knowledge" (MCTS inspiration)

---

*下一篇：[Context Engineering](05-context-engineering.md)*

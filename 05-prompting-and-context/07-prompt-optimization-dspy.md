<a id="prompt-optimization-dspy"></a>
# 提示最佳化（DSPy）

提示工程正從「手動微調」時代走向「程式化」時代。**DSPy（Declarative Self-improving Language Programs）** 是 2025 年業界建立穩健 LLM pipeline 的標準做法之一：提示由演算法自動最佳化。

<a id="table-of-contents"></a>
## 目錄

- [DSPy 哲學：Programming vs. Prompting](#the-dspy-philosophy-programming-vs-prompting)
- [Signatures 與 Modules](#signatures--modules)
- [Teleprompters（Optimizers）](#teleprompters-optimizers)
- [「Prompt as Weight」類比](#the-prompt-as-weight-analogy)
- [以 Metric 驅動的最佳化](#metric-driven-optimization)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-dspy-philosophy-programming-vs-prompting"></a>
## DSPy 哲學：Programming vs. Prompting

在傳統提示工程中，若你更換模型（例如從 GPT-4o 換成 Llama-4），就必須重寫所有提示。
**DSPy 把 Logic 與 Formatting 分離。**

- **Logic**：由 **Modules** 定義（例如 ChainOfThought、ReAct）。
- **Optimization**：系統會自動為某個**特定**模型找出最能實現該邏輯的提示與範例。

---

<a id="signatures--modules"></a>
## Signatures 與 Modules

你不再直接撰寫 prompt，而是定義 **Signature**：輸入是什麼、輸出應該是什麼。

```python
# DSPy 2025 Pattern
class MultiHopQA(dspy.Signature):
    """Answer questions that require multiple context retrievals."""
    context = dspy.InputField()
    question = dspy.InputField()
    answer = dspy.OutputField(desc="A concise 1-sentence answer")

# Logic is handled by a Module
qa_system = dspy.ChainOfThought(MultiHopQA)
```

---

<a id="teleprompters-optimizers"></a>
## Teleprompters（Optimizers）

Teleprompters 是會反覆改進你程式的演算法，用來提升準確率。
1. **BootstrapFewShot**：自動為你的 prompt 找出高品質範例。
2. **MIPROv2（2025）**：一種 Bayesian optimizer，會嘗試不同的指令措辭，並選出能讓分數最高的版本。

**為什麼重要**：你不必再猜「Be helpful」還是「Think carefully」比較好。optimizer 會用資料證明答案。

---

<a id="the-prompt-as-weight-analogy"></a>
## 「Prompt as Weight」類比

在 DSPy 裡，你的 prompt 就像神經網路中的權重。你不會把權重「硬編碼」進去；你會訓練它們。
- 如果你更換模型，只要把程式**重新編譯**（重新訓練）即可。optimizer 會替新模型找出它更容易理解的新 few-shot 範例。

---

<a id="metric-driven-optimization"></a>
## 以 Metric 驅動的最佳化

最佳化需要一個 **Metric**（會回傳分數的函式）。
- **Exact Match**：`prediction.answer == target.answer`
- **LLM-as-Judge**：使用較大的模型（GPT-5.2）去評分較小模型（Llama 8B）的輸出。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-how-does-dspy-solve-the-fragility-of-prompt-engineering"></a>
### 問：DSPy 如何解決提示工程的「脆弱性」？

**強答案：**
DSPy 把「格式化」與「grounding」的複雜度，從人類身上轉移到編譯器。當我們手寫提示時，本質上是在「硬編碼」某個模型、某個時間點專用的行為（point-in-time tuning）。只要模型被更新或替換，提示就可能失效。DSPy 將 prompt 視為可學習參數。透過定義清楚的 **Signature** 與 **Metric**，我們讓系統能在數千次模擬迭代中「搜尋」最有效的提示，讓最終系統對模型變動更具韌性。

<a id="q-what-is-a-teleprompter-in-the-context-of-dspy"></a>
### 問：在 DSPy 裡，「Teleprompter」是什麼？

**強答案：**
Teleprompter 是一種程式化 optimizer。它的工作是接收一個 DSPy 程式（可能是一條複雜的 module 鏈）和一小組訓練範例，然後把它們「編譯」成最佳化版本。它會生成可能的「thinking patterns」與範例，透過 metric 測試，再挑出最有效的組合。簡單來說，Teleprompter 就是提示工程世界裡的「Gradient Descent」。

---

<a id="references"></a>
## 參考資料
- Khattab et al. "DSPy: Compiling Declarative Language Models" (2023/2024)
- Stanford NLP. "DSPy Documentation and Tutorials" (2025)

---

*下一篇：[Prompt Injection 與防禦](08-prompt-injection-defense.md)*

<a id="few-shot-and-in-context-learning-icl"></a>
# Few-Shot 與 In-Context Learning（ICL）

In-Context Learning（ICL）是指 LLM 僅透過在提示中看到範例，就能學會新任務的能力，且不需要任何權重更新。最大化 ICL 效率，是提升提示穩定性的關鍵槓桿。

<a id="table-of-contents"></a>
## 目錄

- [Few-Shot 範例的結構](#the-anatomy-of-a-few-shot-example)
- [需要多少範例？](#how-many-examples)
- [動態範例選擇](#dynamic-example-selection)
- [標註細緻度的重要性](#the-importance-of-labelling-nuance)
- [進階 ICL：類比與「Few-Shot CoT」](#advanced-icl-analogy-and-few-shot-cot)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-anatomy-of-a-few-shot-example"></a>
## Few-Shot 範例的結構

一個高品質範例由三個部分組成：
1. **輸入（Input）**：潛在使用者資料的真實樣本。
2. **推理（可選）**：簡短說明為什麼輸出會是這個結果。
3. **輸出（Output）**：作為「Gold Standard」的結果。

```markdown
User: "The weather is okay, but the flight was late."
Reasoning: The user is neutral about the weather but negative about the service.
Sentiment: Mixed
```

---

<a id="how-many-examples"></a>
## 需要多少範例？

| 模型規模 | 最佳甜蜜點 | 擴展行為 |
|----------|------------|----------|
| **小型（8B）** | 5 - 10 | 效益通常會持續到約 20 個範例。 |
| **中型（70B）** | 3 - 5 | 很早就會進入平台期；再增加範例只會提高延遲。 |
| **Frontier（405B）** | 1 - 2 | 能力很強；通常僅靠「Instruction Following」就足夠。 |

**經驗法則**：如果你需要超過 20 個範例才能得到穩定輸出，那多半表示任務對該模型來說太複雜，或你應該考慮 **Fine-tuning**。

---

<a id="dynamic-example-selection"></a>
## 動態範例選擇

在正式環境的 RAG 或 Classification 中，不要對每位使用者都使用同一組靜態範例。
**動態模式如下：**
1. 使用者提供查詢。
2. 到「Vector DB of Gold Examples」中搜尋 3 個**語意最相近**的案例。
3. 將這 3 個特定案例注入提示中。

**結果**：由於模型看到的是與當前使用者最相關的「局部」模式，因此準確率會顯著提升。

---

<a id="the-importance-of-labelling-nuance"></a>
## 標註細緻度的重要性

frontier models 對範例中的 **Distribution Bias** 很敏感。
- 如果你提供 5 個「Positive」範例和 1 個「Negative」範例，模型就會偏向預測「Positive」。
- **修正方式**：務必使用 **Label Balancing**。確保 few-shot 範例大致反映預期輸出分布，或完全平衡（1:1）。

---

<a id="advanced-icl-analogy-and-few-shot-cot"></a>
## 進階 ICL：類比與「Few-Shot CoT」

**Analogy Prompting**：不要只說「做 X」，而是提供一個類比。 
「Translate this code like a translator would move a poem from French to English—preserving the soul (logic) but changing the syntax.」

**Few-Shot CoT**：提供 2 個推理過程明確可見的範例。這能「預熱」模型的注意力，讓它聚焦在邏輯上，而不只是模仿輸出字串。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-not-just-provide-all-50-examples-we-have-in-the-prompt"></a>
### 問：為什麼不直接把我們手上的 50 個範例全部放進提示裡？

**強答案：**
主要有三個原因：
1. **Context Window Latency**：每個範例都會增加 token，進而提高「Prefill」時間與每次請求的成本。
2. **Attention Dilution**：即使有 128k context，模型仍可能在大量不相關資料中「遺失」特定約束（也就是「lost-in-the-middle」效應）。
3. **Overfitting**：提供過多狹窄的範例，可能讓模型過度模仿範例的**格式**，失去處理該集合之外 edge cases 的一般化能力。

<a id="q-what-is-label-bias-in-in-context-learning"></a>
### 問：In-Context Learning 中的「Label Bias」是什麼？

**強答案：**
Label bias 是指模型更頻繁預測某個特定標籤，只因為它在 few-shot 範例中出現得更多，或出現在清單末尾。標準緩解方式包括：
1. 對不同請求打亂範例順序。
2. 確保 positive / negative / neutral 樣本數量相等。
3. 在提示開發期間使用 **Permutation Testing**，確認模型回應的是內容，而不是順序。

---

<a id="references"></a>
## 參考資料
- Brown et al. "Language Models are Few-Shot Learners" (2020)
- Min et al. "Rethinking the Role of Demonstrations: What Makes In-Context Learning Work?" (2022)

---

*下一篇：[Chain-of-Thought](03-chain-of-thought.md)*

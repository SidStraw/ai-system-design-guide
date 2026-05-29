<a id="chain-of-thought-cot"></a>
# Chain-of-Thought（CoT）

Chain-of-Thought（CoT）是一種鼓勵 LLM 在給出最終答案前，先生成中間推理步驟的技術。它已從簡單的提示短語，演變成推理模型的核心架構特性（o1、DeepSeek-R2、具 extended thinking 的 Claude Opus 4.7、具 extended thinking 的 GPT-5.5）。

<a id="table-of-contents"></a>
## 目錄

- [CoT 革命](#the-cot-revolution)
- [Zero-Shot 與 Programmatic CoT](#zero-shot-vs-programmatic-cot)
- [「Thinking」模型的崛起（o1、DeepSeek-R1）](#the-rise-of-thinking-models-o1-deepseek-r1)
- [自我修正與驗證](#self-correction-and-verification)
- [CoT 何時會失效（過度思考）](#when-cot-fails-over-thinking)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-cot-revolution"></a>
## CoT 革命

標準 LLM 是「Next Token Predictors」。對於複雜數學或邏輯任務，一次生成往往不夠。CoT 為模型提供了可用來拆解子問題的「Scribble Pad」（工作記憶）。

**公式**：`Input -> Reasoning (Chain) -> Output`

---

<a id="zero-shot-vs-programmatic-cot"></a>
## Zero-Shot 與 Programmatic CoT

| 技術 | 觸發短語 | 效率 | 適用情境 |
|------|----------|------|----------|
| **Zero-Shot CoT** | "Let's think step by step." | 高 | 臨時查詢。 |
| **Few-Shot CoT** | （提供帶有邏輯的範例） | 穩定性更高 | 正式環境流程。 |
| **Programmatic CoT** | "1. Analyze X. 2. Verify Y. 3. Resolve Z." | **最適合 Agents** | 複雜的多工具任務。 |

---

<a id="the-rise-of-thinking-models-o1-deepseek-r1"></a>
## 「Thinking」模型的崛起

像 **OpenAI o1/GPT-5.5 extended thinking**、**DeepSeek-R2** 與 **Claude Opus 4.7** 這類模型，已透過 Reinforcement Learning（RL）把 CoT「內建」進去。

1. **System-Level CoT**：模型不只是「印出」推理，而是擁有專門的「Thinking Window」。
2. **Hidden CoT**：在許多企業版中，推理鏈對使用者隱藏，但系統仍可驗證，以防 prompt injection 或「thought leakage」。
3. **Scaling Law**：這些模型遵循 **Inference Scaling Law**——它們「思考」越久，越能解出困難問題（只要給 $o1$ 足夠時間，它能解出金牌等級的 IMO 數學題）。

---

<a id="self-correction-and-verification"></a>
## 自我修正與驗證

正式環境流程不再信任單一的 Chain-of-Thought，而是疊加 **Self-Verification**。

```markdown
# Process
1. Generate Answer A via CoT.
2. Critique: "Are there any errors in the logic above?"
3. If errors: "Correct the logic and provide Answer B."
```

**細節**：這現在已整合進用於程式開發的 **Execution-Verified CoT**，也就是模型先寫出邏輯、執行程式，再在程式失敗時自行修正。

---

<a id="when-cot-fails-over-thinking"></a>
## CoT 何時會失效（過度思考）

CoT 不是萬靈丹。對於簡單任務，它會額外帶來：
1. **延遲**：更多 token = 更慢的回應。
2. **成本**：每一個「thought」token 都要付費。
3. **過度思考**：模型可能在根本不複雜的地方幻想出複雜性（例如用 3 段文字解釋為什麼 2+2=4）。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-does-cot-improve-performance-on-mathematical-word-problems"></a>
### 問：為什麼 CoT 能提升數學文字題的表現？

**強答案：**
CoT 之所以能提升表現，是因為它讓模型的計算複雜度與任務的邏輯複雜度更一致。在標準的單次生成中，模型必須根據有限的局部資訊直接預測最終答案 token。有了 CoT 之後，模型可以把問題「拆開」成更小的自回歸步驟。每個步驟都把前一步的輸出當作上下文，使注意力機制能一次專注在一個子問題上（例如先把蘋果加總，再扣掉橘子），從而降低單次預測的「認知負荷」。

<a id="q-how-do-you-handle-cot-in-a-production-environment-where-latency-is-critical"></a>
### 問：在對延遲極度敏感的正式環境中，你會如何處理 CoT？

**強答案：**
我們會採用 **Hybrid Reasoning Architecture**：
1. **Tier 1（快速）**：先由分類器判斷查詢是否需要深度推理。
2. **Tier 2（濃縮 CoT）**：提示模型「Be concise in your reasoning」，或使用 **Knowledge Distillation**，讓較小模型只輸出最終答案，但受益於 teacher 的 CoT 式預訓練。 
3. **Tier 3（串流）**：把 CoT 串流給使用者（若系統透明）或背景程序，使系統能在最終結果完整出現前就開始「預處理」。

---

<a id="references"></a>
## 參考資料
- Wei et al. "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models" (2022)
- Wang et al. "Self-Consistency Improves Chain of Thought Reasoning in Language Models" (2023)
- OpenAI. "Learning to Reason with LLMs" (2024)

---

*下一篇：[Tree-of-Thought](04-tree-of-thought.md)*

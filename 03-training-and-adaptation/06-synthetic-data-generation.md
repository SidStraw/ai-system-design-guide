<a id="synthetic-data-generation"></a>
# 合成資料生成

產業已撞上「Data Wall」——高品質人類文本在網路上的供給耗盡。Synthetic data 現在已成為模型持續進步的主要引擎，也是所有現代前沿模型配方的核心。

<a id="table-of-contents"></a>
## 目錄

- [Data Wall 與合成轉向](#synthetic-shift)
- [Evol-Instruct 模式](#evol-instruct)
- [Constitutional AI 與自我修正](#constitutional-ai)
- [可驗證的合成資料（Math/Code）](#verifiable-data)
- [去偏差與多樣性](#diversity)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="after-the-data-wall-the-synthetic-shift"></a>
## 「Data Wall」之後：合成轉向

前沿模型（Llama 4、GPT-5.5、Claude Opus 4.7、Gemini 3.1 Pro）都在 100T+ tokens 上訓練。現實是，已經沒有足夠的人類文本可支撐這種規模擴張。
**實情是：**前沿 fine-tuning（以及 10% 的 pretraining）中，**超過 50% 的訓練混合資料**現在都是合成資料。

| 來源 | Human Data | Synthetic Data |
|--------|------------|----------------|
| **Volume** | 固定（有限） | 無限 |
| **Quality** | 波動大（有噪音） | 可控制（已淨化） |
| **Cost** | 高（Human Labelers） | 便宜（Inference/GPU） |
| **Bias** | 反映網際網路 | 可手動平衡 |

---

<a id="evol-instruct-pattern"></a>
## Evol-Instruct 模式

Evol-Instruct 是一種遞迴流程：LLM 先接收簡單指令，再把它演化成更複雜的指令。

**演化方向：**
1. **Breadth**：增加任務數量。
2. **Depth**：加入限制條件、複雜因素或多步驟邏輯。
3. **De-noising**：清理措辭，移除「AI 口吻」。

```python
# Simple Instruction: "Write a function to add two numbers."
# Evolved Instruction: "Write a thread-safe Python class that performs 
# matrix addition with error handling and unit tests, adhering to PEP8."
```

---

<a id="constitutional-ai-ai-feedback-rlaif"></a>
## Constitutional AI 與 AI Feedback（RLAIF）

這種方法由 Anthropic 提出，並已被整個產業廣泛採用。RLAIF 會使用一份「Constitution」（規則集合）來引導模型評估並改進自己的資料。

**迴圈如下：**
1. **Propose**：Model A 生成回應。
2. **Critique**：Model B（憲章裁判）依據指引指出缺陷。
3. **Revise**：Model A 根據評論生成更好的版本。
4. **Train**：最終的 `(Prompt, Revise)` 配對會加入 SFT 資料集。

---

<a id="verifiable-synthetic-data"></a>
## 可驗證的合成資料

合成資料最大的風險是 **Model Collapse**（模型學到自己的錯誤）。
**2025 年的解法**：聚焦在不需要 LLM 也能驗證「真相」的領域。

- **Math**：使用 Formal Verification（Lean/Isabelle）或 Python 執行結果驗證答案。
- **Code**：讓生成程式碼通過測試案例（Unit Tests）。
- **RAG**：使用「Gold Context」來產生答案明確存在於文本中的問題。

---

<a id="de-biasing-and-diversity"></a>
## 去偏差與多樣性

合成資料被用來「補齊」人類資料的空缺。
- **Languages**：透過翻譯概念模板，生成低資源語言（例如 Swahili、Marathi）的高品質文本。
- **Logic**：建立 1,000,000 個特定邏輯謬誤的變體，以提升模型對這些問題的韌性。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-what-is-the-risk-of-model-collapse-when-training-on-synthetic-data"></a>
### Q: 使用合成資料訓練時，「Model Collapse」的風險是什麼？

**強答：**
當模型使用自己早期版本生成的資料再訓練自己時，就會發生 Model Collapse。因為模型的分布比真實世界更窄（它對某些字詞與模式有偏好／偏見），整個訓練迴圈會變成一種錯誤與平庸化的「正回饋循環」。到了 2025 年，我們主要用以下方式緩解：
1. 混入 5-20% 的「Golden」人類驗證資料。
2. 使用「可驗證」的 rewards（Math/Code），讓錯誤永遠不會被學進去。
3. 用更強的「Teacher」模型為「Student」模型生成資料。

<a id="q-how-do-you-ensure-the-quality-of-a-synthetic-dataset-of-10-million-rows"></a>
### Q: 你如何確保一個 1,000 萬列合成資料集的*品質*？

**強答：**
我們會使用 **多階段過濾管線**：
1. **Semantic Deduplication**：用 embeddings 移除幾乎相同的群集。
2. **LLM-as-Judge**：抽樣 1% 資料，交給更強的模型（例如 GPT-5.2）評估其邏輯與安全性。
3. **Perplexity Filtering**：用小模型計算文本 perplexity。若太高（胡言亂語）或太低（過度重複／太簡單），就丟棄。
4. **Verifiable Execution**：若資料包含程式碼或數學，必須通過本地 compiler/interpreter 檢查。

---

<a id="references"></a>
## 參考資料
- Xu et al. "WizardLM: Empowering Large Language Models to Follow Complex Instructions" (2023)
- Bai et al. "Constitutional AI: Harmlessness from AI Feedback" (2022)
- OpenAI. "Weak-to-Strong Generalization" (2023)

---

*下一篇：[Quantization Deep Dive](07-quantization-deep-dive.md)*

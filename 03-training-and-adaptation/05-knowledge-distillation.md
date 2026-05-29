<a id="knowledge-distillation"></a>
# 知識蒸餾

知識蒸餾是把大型且複雜模型（「Teacher」）的智慧轉移到更小、更高效率模型（「Student」）的過程。這正是當今許多參數量不高、卻能展現超規格表現的小型 open-weight 模型背後的秘密。

<a id="table-of-contents"></a>
## 目錄

- [Teacher-Student 範式](#teacher-student-paradigm)
- [蒸餾如何運作](#how-distillation-works)
- [Feature vs. Output Distillation](#feature-vs-output)
- [Self-Distillation from Proof (SDP)](#self-distillation-proof)
- [量化感知蒸餾](#quantization-aware-distillation)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-teacher-student-paradigm"></a>
## Teacher-Student 範式

小模型（例如 Llama 4 8B、Gemini 3.1 Flash、Claude Haiku 4.5）並不只是靠原始網路資料訓練。它們也會使用由更大型模型（例如 GPT-5.5、Claude Opus 4.7 或 Llama 4 405B）產生或策展的 **Synthetic Data**。

| 模型 | 角色 | 智慧來源 |
|-------|------|---------------------|
| **Teacher** | 大型（100B+ Params） | 在 50T+ tokens 上預訓練 |
| **Student** | 小型（1B - 8B Params） | Teacher 過濾後的邏輯／輸出 |

---

<a id="how-distillation-works"></a>
## 蒸餾如何運作

<a id="1-hard-label-distillation"></a>
### 1. Hard Label Distillation
Student 學習 Teacher 的最終預測（例如某題的答案）。

<a id="2-soft-label-distillation-temperature-scaling"></a>
### 2. Soft Label Distillation（Temperature Scaling）
Student 學習 Teacher 的**機率分布**（Logits）。這種資訊更豐富，因為它不只告訴 Student 正確答案，也說明哪些錯誤答案其實「差一點就對」。

```python
# Distillation Loss (KL Divergence):
Loss = KL_Div(Teacher_Logits / T, Student_Logits / T)
```
*其中 T 是 Temperature（通常為 2.0 - 5.0）。*

---

<a id="feature-vs-output-distillation"></a>
## Feature vs. Output Distillation

<a id="output-distillation-standard"></a>
### Output Distillation（標準作法）
Student 對齊 Teacher 的文字回應。
- **Pros**：可透過 API 輕鬆實作。
- **Cons**：只能學到表層行為模式。

<a id="featurehidden-state-distillation"></a>
### Feature/Hidden State Distillation
Student 對齊 Teacher 的內部 **Hidden States**（向量表徵）。
- **Requirement**：你必須能存取 Teacher 的權重（Open Weights）。
- **Pro**：Student 能學到 Teacher 的「內部概念地圖」，因此推理深度更高。

---

<a id="self-distillation-from-proof-sdp"></a>
## Self-Distillation from Proof (SDP)

**推理能力的突破。**
像 o1、DeepSeek-R1 與 Claude Opus 4.7 這類模型，會用 SDP 在沒有新增人工資料的情況下持續進步。

1. **Generation**：模型對困難的數學／程式問題產生 100 個可能解法。
2. **Verification**：規則式系統（compiler/calculator）找出其中 1 個正確解。
3. **Distillation**：模型以導向正確解的 **Chain of Thought (CoT)** 為資料進行微調。

**結果**：模型透過只保留高品質推理路徑來「蒸餾自己」。

---

<a id="quantization-aware-distillation"></a>
## 量化感知蒸餾

標準量化（例如從 16-bit 降到 4-bit）會帶來些微準確率下降。
**解法**：在量化過程中同步使用 Knowledge Distillation。16-bit 模型擔任 Teacher，引導 4-bit 模型把誤差降到最低。這就是現代 4-bit 模型能追上 16-bit 效能的方法。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-is-a-distilled-8b-model-better-than-an-8b-model-trained-from-scratch-on-the-same-tokens"></a>
### Q: 為什麼蒸餾後的 8B 模型，會比用相同 tokens 從零訓練的 8B 模型更好？

**強答：**
從零訓練（Pretraining）直接吃原始網路資料會很嘈雜；模型得花大量容量去學會如何穿越這些雜訊。蒸餾模型則是在「淨化後」的課程上訓練。Teacher 模型像高品質濾鏡，提供結構化邏輯、清楚說明與更乾淨的語言分布。從本質上說，Teacher 透過它的 logit 分布給 Student「提示」，告訴它語言中哪些特徵最值得學。

<a id="q-what-are-the-risks-of-using-gpt-4o-as-a-teacher-to-distill-a-llama-student"></a>
### Q: 若用 GPT-4o 當 Teacher 來蒸餾 Llama Student，有哪些風險？

**強答：**
1. **Model Collapse**：若 Student 只看 Teacher 的輸出，可能會失去創意或多樣知識的「長尾」，最後只學到 Teacher 較狹窄的偏見。
2. **License Violations**：多數專有模型（OpenAI、Anthropic）都禁止把其輸出用於訓練「競爭」模型。對企業而言，拿 API 輸出蒸餾自家模型是重大法律風險。
3. **Linguistic Mimicry**：Student 可能只學會*說得*很有自信（像 Teacher 一樣），卻沒有同等的邏輯深度，導致自信但錯誤的 hallucinations。

---

<a id="references"></a>
## 參考資料
- Hinton et al. "Distilling the Knowledge in a Neural Network" (2015)
- Gou et al. "Knowledge Distillation: A Survey" (2021)
- DeepSeek. "DeepSeek-R1: Incentivizing Reasoning Capability" (2025)

---

*下一篇：[Synthetic Data Generation](06-synthetic-data-generation.md)*

<a id="fine-tuning-strategies"></a>
# 微調策略

微調會讓預訓練模型適應特定任務、領域或風格。如今的微調已較少是在「教模型事實」，更多是在「教模型格式與行為」。

<a id="table-of-contents"></a>
## 目錄

- [何時該微調](#when-to-fine-tune)
- [監督式微調（SFT）](#supervised-fine-tuning)
- [持續預訓練（領域適應）](#continued-pretraining)
- [指令微調](#instruction-tuning)
- [PEFT vs. 全參數微調](#peft-vs-full-parameter)
- [超參數調整](#hyperparameter-tuning)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="when-to-fine-tune"></a>
## 何時該微調

在微調之前，先問：**這能用 Prompt Engineering 或 RAG 解決嗎？**

| 需求 | 更好的解法 | 原因 |
|-------------|-----------------|-----|
| 新事實 / 新知識 | **RAG** | LLM 不擅長從 FT 記住事實；RAG 更容易更新。 |
| 特定輸出格式 | **Fine-Tuning** | 可讓模型穩定輸出 JSON/XML，而不需複雜提示。 |
| 語氣 / Persona | **Fine-Tuning** | 比 system prompt 一致得多。 |
| 降低延遲 | **Fine-Tuning** | 可減少對長 few-shot prompts 的需求。 |
| 私有領域語言 | **Continued Pretraining** | 可教會專門詞彙（醫療、法律、自訂程式碼）。 |

---

<a id="supervised-fine-tuning-sft"></a>
## 監督式微調（SFT）

這是預訓練後的第一步。模型會在 `(Prompt, Response)` 配對資料上訓練。

<a id="the-quality-hierarchy"></a>
### 品質層級
**1,000 個「完美」範例勝過 1,000,000 個嘈雜範例。**
- **Golden Sets：**由領域專家人工策展（技術任務通常需要 PhD 等級）。
- **Negative Constraint Training：**納入模型**不應該**做什麼的範例（例如「不要道歉」「不要提到你是 AI」）。

---

<a id="continued-pretraining-domain-adaptation"></a>
## 持續預訓練（領域適應）

也稱作「Second-stage Pretraining」。
- **How**：在特定領域的原始文字上訓練（例如金融模型可使用所有 SEC filings）。
- **Objective**：學習該領域語言的統計分布。
- **Nuance**：需要低得多的 learning rate（約原本的 1/10）以避免「catastrophic forgetting」。

---

<a id="peft-vs-full-parameter"></a>
## PEFT vs. 全參數微調

| 特性 | Full-Parameter FT | PEFT (LoRA, QLoRA) |
|---------|-------------------|--------------------|
| GPU VRAM | 很高（模型大小 * 4-12） | 低（模型大小 * 1.5） |
| 速度 | 基準 | 快 2x-3x |
| 風險 | 高（Catastrophic Forgetting） | 低 |
| 部署 | 每個任務一個模型 | 一個 base model + 多個 adapters |
| **結論**| 保留給基礎模型訓練 | **正式環境標準** |

---

<a id="hyperparameter-tuning"></a>
## 超參數調整

<a id="1-learning-rate-lr"></a>
### 1. Learning Rate (LR)
- **SFT**：`1e-5` 到 `5e-5` 是常見標準。
- **Too high**：模型會「崩掉」，開始重複或輸出胡言亂語。

<a id="2-rank-r-for-lora"></a>
### 2. LoRA 的 Rank（r）
- 較高 rank（`r=64` 到 `r=256`）適合複雜推理任務。
- 較低 rank（`r=8`）適合單純風格／語氣調整。

<a id="3-packaged-training-packing"></a>
### 3. 打包式訓練（Packing）
為了最大化吞吐量，我們會把多個短樣本「打包」進單一 4k 或 8k 序列中，並以 EOS tokens 分隔。
- **Challenge**：Self-attention 可能在樣本間洩漏資訊。
- **Solution**：使用 **FlashAttention with block-masking** 防止跨樣本 attention。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-use-continued-pretraining-instead-of-just-putting-domain-data-in-the-sft-set"></a>
### Q: 為什麼要用持續預訓練，而不是直接把領域資料放進 SFT 資料集？

**強答：**
SFT 在資料建立上很「昂貴」——你需要 prompt/answer 配對。持續預訓練則能利用大量原始、未標註的領域文字，把專門詞彙與風格教進模型的內部表徵。當模型先「學會這個語言」後，你再用小型 SFT 資料集教它該語言中的「任務」（例如分類、摘要）。

<a id="q-how-do-you-prevent-a-model-from-unlearning-general-capabilities-during-fine-tuning"></a>
### Q: 如何避免模型在微調時「遺忘」一般能力？

**強答：**
這就是「Catastrophic Forgetting」。主要有兩種緩解方式：
1. **Rehearsal：**在微調資料集中混入 5-10% 原始預訓練資料。
2. **PEFT (LoRA)：**因為只訓練少量權重，原本的「知識」仍凍結在 base model 權重中，因此大幅降低遺忘風險。

---

<a id="references"></a>
## 參考資料
- Hu et al. "LoRA: Low-Rank Adaptation of Large Language Models" (2021)
- Ouyang et al. "Training language models to follow instructions" (InstructGPT, 2022)
- Dettmers et al. "QLoRA: Efficient Finetuning of Quantized LLMs" (2023)

---

*下一篇：[LoRA, QLoRA, and PEFT](03-lora-qlora-peft.md)*

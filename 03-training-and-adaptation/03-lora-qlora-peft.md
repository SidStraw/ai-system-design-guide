<a id="lora-qlora-and-peft"></a>
# LoRA、QLoRA 與 PEFT

Parameter-Efficient Fine-Tuning (PEFT) 是調整 LLM 的業界標準。本章涵蓋 LoRA 與其他 PEFT 方法的機制與進階變體。

<a id="table-of-contents"></a>
## 目錄

- [PEFT 革命](#the-peft-revolution)
- [LoRA 機制](#lora-mechanics)
- [QLoRA：4-bit 微調](#qlora)
- [進階變體（DoRA、Vera、RS-LoRA）](#advanced-variants)
- [Multi-LoRA Serving（Adapters）](#multi-lora-serving)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-peft-revolution"></a>
## PEFT 革命

對多數企業而言，前沿模型（GPT-5.5、Claude Opus 4.7、Llama 4 405B）的全量微調在經濟上不可行。PEFT 帶來：
1. **記憶體效率**：可在單張 A100 上訓練 70B 模型。
2. **速度**：只更新 <1% 權重，訓練速度提升 2x。
3. **模組化**：可在不重新載入權重的情況下，把不同「技能」（adapters）切換到共享的 base model 上。

---

<a id="lora-mechanics"></a>
## LoRA 機制

LoRA（Low-Rank Adaptation）會在 transformer layers 中注入可訓練的低秩分解矩陣。

```python
# The LoRA Equation for a Weight Matrix W:
h = Wx + (BA)x * (alpha/r)
```
- **W**：預訓練權重（Frozen，Gradient = None）
- **A, B**：LoRA adapters（可訓練）
- **r**：Rank（例如 8、16、64）
- **alpha**：縮放因子（通常為 2 * rank）

<a id="principal-nuance-target-modules"></a>
### 關鍵細節：Target Modules
過去通常只針對 query/value projections（`q_proj`、`v_proj`）。
**現代標準**：即使在較低 rank 下，也會針對**所有**線性層（`q, k, v, o, gate, up, down`），以獲得最佳穩定性與效能。

---

<a id="qlora-4-bit-fine-tuning"></a>
## QLoRA：4-bit 微調

QLoRA 透過把 base model 量化為 4-bit（NF4），同時維持 16-bit gradients，進一步提升效率。

| 最佳化 | 方法 | 效益 |
|--------------|--------|---------|
| **NF4 Quantization** | Normalized Float 4 | 資訊密度優於標準 Int4 |
| **Double Quant** | 對量化常數再次量化 | 每個模型可省下約 0.5 GB VRAM |
| **Paging** | Unified Memory (Nvidia) | 透過溢寫到 CPU RAM 避免 OOM |

---

<a id="advanced-variants"></a>
## 進階變體

<a id="1-dora-weight-decomposed-low-rank-adaptation"></a>
### 1. DoRA（Weight-Decomposed Low-Rank Adaptation）
DoRA 會把權重更新拆成**Magnitude** 與 **Direction**。
- **Result**：學習速度比 LoRA 快 2x，且表現更接近全量微調。
- **Why it wins**：它能讓模型獨立調整「改多少」與「改什麼」。

<a id="2-vera-vector-based-random-aggregation"></a>
### 2. Vera（Vector-based Random Aggregation）
Vera 不使用低秩矩陣 `A` 與 `B`，而是改用固定隨機投影搭配小型可訓練向量。
- **Efficiency**：相較於 LoRA，adapter 大小減少 **10x**。
- **Use Case**：大規模 Multi-LoRA serving。

<a id="3-rs-lora-rank-stabilized-lora"></a>
### 3. RS-LoRA（Rank-Stabilized LoRA）
使用 `alpha / sqrt(r)` 作為縮放因子。
- **Benefit**：可以把 rank 提高到 256+，而不會讓模型變得不穩定，也不需要降低 learning rate。

---

<a id="multi-lora-serving-adapters"></a>
## Multi-LoRA Serving（Adapters）

正式系統現在會服務單一 base model（例如 Llama 4 70B），並在同一個 batch 中動態切換 adapters。

```python
# vLLM/LMCache Multi-LoRA Pattern:
# Request 1 -> Base + Finance_Adapter
# Request 2 -> Base + Legal_Adapter
# Request 3 -> Base + Medical_Adapter
```
**技術關鍵**：**Continuous Batching + PagedAttention v3** 讓系統能服務 100+ adapters，且相較於 base model 只增加 5-10% 延遲。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-is-the-lora-alpha-parameter-usually-set-to-2x-the-rank"></a>
### Q: 為什麼 LoRA 的 alpha 參數通常設為 rank 的 2 倍？

**強答：**
`alpha` 是 LoRA 更新量的縮放因子。初始化 LoRA 矩陣時，B 通常會以零初始化，而 A 是隨機值。訓練過程中，更新大小會依賴 rank `r`。將 `alpha=2r`（或其他常數倍）可確保日後若調整 rank（例如從 8 改成 16）時，不需要重新調 learning rate。`alpha/r` 這個比例能讓更新幅度相對於 learning rate 維持正規化。

<a id="q-what-is-dora-and-why-would-you-use-it-over-standard-lora"></a>
### Q: 什麼是 DoRA？為什麼會用它取代標準 LoRA？

**強答：**
DoRA（Weight-Decomposed Low-Rank Adaptation）是 2024 年提出的技術，它像 Weight Normalization 一樣，把預訓練權重更新拆成 magnitude 與 direction 兩個部分。標準 LoRA 會同時更新兩者，而 DoRA 允許它們獨立學習。實證上，DoRA 收斂更好、準確率更高，且常能在低 rank 下逼近全參數微調，因此是高風險領域適應中的偏好選項。

---

<a id="references"></a>
## 參考資料
- Hu et al. "LoRA: Low-Rank Adaptation of Large Language Models" (2021)
- Liu et al. "DoRA: Weight-Decomposed Low-Rank Adaptation" (2024)
- Dettmers et al. "QLoRA: Efficient Finetuning of Quantized LLMs" (2023)

---

*下一篇：[RLHF and DPO](04-rlhf-and-dpo.md)*

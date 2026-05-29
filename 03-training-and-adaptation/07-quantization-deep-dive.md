<a id="quantization-deep-dive"></a>
# 量化深度解析

量化是降低模型權重精度（例如從 16-bit 降到 4-bit）以節省記憶體並提升推論速度的過程。這是把大型模型部署到消費級或單 GPU 硬體上的主要工具。

<a id="table-of-contents"></a>
## 目錄

- [精度與效能的取捨](#precision-performance)
- [量化方法（NF4、GPTQ、AWQ）](#methods)
- [GGUF vs. EXL2](#formats)
- [KV Cache 量化（VRAM 救星）](#kv-cache)
- [量化感知微調](#qaft)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-precision-performance-tradeoff"></a>
## 精度與效能的取捨

傳統模型使用 **BF16**（16-bit）。量化的目標是進一步降到 **8-bit (FP8)**、**4-bit (Int4/NF4)**，甚至 **1.5-bit (BitNet)**。

| 精度 | Bits | 權重大小（8B Model） | 品質損失 | GPU 相容性 |
|-----------|------|------------------------|--------------|-------------------|
| **BF16** | 16 | 16 GB | 0%（基準） | 所有現代 GPU |
| **FP8** | 8 | 8 GB | < 1% | H100 / B200 / RTX 4090 |
| **4-bit (NF4)**| 4 | 5 GB | 1-2% | 所有現代 GPU |
| **2-bit** | 2 | 2.5 GB | 10-15% | 研究 / 特化場景 |

---

<a id="quantization-methods"></a>
## 量化方法

<a id="1-nf4-normalfloat4"></a>
### 1. NF4 (NormalFloat4)
這是 fine-tuning（QLoRA）的黃金標準。它假設權重服從常態分布，並將它們映射到 16 個值中。

<a id="2-awq-activation-aware-weight-quantization"></a>
### 2. AWQ（Activation-aware Weight Quantization）
AWQ 不會平均量化所有權重，而是找出對品質最重要的 **1%「顯著權重」**，並維持較高精度。
- **Pro**：準確率優於 GPTQ。

<a id="3-fp8-multi-node-standard"></a>
### 3. FP8（多節點標準）
這是 Nvidia Transformer Engine 支援的硬體原生量化方式。
- **Why it wins**：它擁有 Int8 的速度，卻具備 Float16 的動態範圍，因此對訓練與推論都很穩定。

---

<a id="gguf-vs-exl2"></a>
## GGUF vs. EXL2

<a id="gguf-llamacpp"></a>
### GGUF（llama.cpp）
- **Deployment**：CPU + GPU offloading。
- **Pros**：跨平台（Mac、Linux、Windows）、單檔案、高可攜。
- **Cons**：比純 GPU 格式更慢。

<a id="exl2-exllamav2"></a>
### EXL2（ExLlamaV2）
- **Deployment**：僅限 GPU（Nvidia）。
- **Pros**：**Nvidia GPU 上最快的 4-bit 格式**。效能明顯優於 AutoGPTQ/AWQ。
- **Cons**：彈性不足（僅支援 Nvidia）。

---

<a id="kv-cache-quantization-the-vram-saver"></a>
## KV Cache 量化（VRAM 救星）

在長上下文 RAG（1M+ tokens）中，**KV Cache** 常常比模型權重本身更吃 VRAM。

- **BF16 KV Cache**：2M tokens ≈ 32GB VRAM（以 8B 模型為例）。
- **FP8/Int4 KV Cache**：2M tokens ≈ 8GB - 16GB VRAM。

**細節**：現代 serving frameworks（vLLM、SGLang、TensorRT-LLM）現在已支援 **Streaming Quantization**，可在執行時即時壓縮 KV cache，讓同一張 GPU 的併發數提升 4 倍。

---

<a id="quantization-aware-training-qat"></a>
## Quantization-Aware Training (QAT)

QAT 不是在模型訓練完成*之後*才量化（Post-training Quantization），而是在訓練*過程中*模擬量化。
- **Result**：模型會學著補償精度損失。
- **Status**：對於小於 3B 參數的模型來說，若要在 4-bit 下仍保持實用，這已是必要做法。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-do-we-use-nf4-instead-of-standard-float4-for-qlora"></a>
### Q: 為什麼 QLoRA 要用 NF4，而不是標準 Float4？

**強答：**
標準 Float4 使用固定網格，無法良好對應 LLM 權重的實際分布，而這些權重通常是以零為中心的常態分布。NF4（NormalFloat4）是一種經數學最佳化的資料型別，讓每個量化區間都能包含相同數量的常態分布值。這能避免權重「擠成一團」，確保模型保留盡可能多的資訊（entropy），因此準確率明顯高於標準 4-bit 整數。

<a id="q-how-does-awq-differ-from-gptq"></a>
### Q: AWQ 和 GPTQ 有何不同？

**強答：**
GPTQ 是一種「逐層」量化方法，目標是最小化權重的均方誤差。AWQ（Activation-aware Weight Quantization）則是「輸入感知」的。它會根據小型校準執行中觀察到的 activation 值，辨識哪些權重最「顯著」。透過只保留這些重要權重（通常 1%）的較高精度，並量化其他部分，AWQ 能比 GPTQ 取得更好的 perplexity，尤其是在小模型或更激進的量化（例如 3-bit）場景中更明顯。

---

<a id="references"></a>
## 參考資料
- Dettmers et al. "QLoRA: Efficient Finetuning of Quantized LLMs" (2023)
- Frantar et al. "GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers" (2022)
- Lin et al. "AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration" (2023)

---

*下一篇：[Inference Fundamentals](../04-inference-optimization/01-inference-fundamentals.md)*

<a id="inference-fundamentals"></a>
# 推論基礎

推論是從已訓練模型產生預測的過程。為了在 Hopper（H100）與 Blackwell（B200）等級硬體上處理重推理工作負載，推論最佳化已從「單純加速」轉向「架構效率」。

<a id="table-of-contents"></a>
## 目錄

- [推論的兩個階段](#two-phases)
- [瓶頸：Compute-Bound vs. Memory-Bound](#bottlenecks)
- [效能指標：TTFT 與 TPOT](#metrics)
- [硬體加持的最佳化（FP8）](#hardware-optimizations)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-two-phases-of-inference"></a>
## 推論的兩個階段

LLM 推論不是單一操作；它由兩個截然不同的計算階段組成。

<a id="1-the-prefill-phase-prompt-processing"></a>
### 1. Prefill 階段（Prompt Processing）
模型會一次處理整個輸入 prompt。
- **Computation**：高平行度矩陣乘法。
- **Bottleneck**：**Compute-bound**（受 GPU TFLOPS 限制）。
- **Time Complexity**：$O(N)$，其中 $N$ 為輸入長度（但可平行化）。

<a id="2-the-decode-phase-token-generation"></a>
### 2. Decode 階段（Token Generation）
模型會逐 token 生成內容，而每個 token 都依賴前一個 token。
- **Computation**：序列式處理，每次只處理權重矩陣的一列。
- **Bottleneck**：**Memory-bound**（受記憶體頻寬限制）。
- **Time Complexity**：$O(M)$，其中 $M$ 為輸出長度（序列式）。

---

<a id="bottlenecks-compute-bound-vs-memory-bound"></a>
## 瓶頸：Compute-Bound vs. Memory-Bound

理解系統真正卡在哪裡，是選擇正確最佳化方式的關鍵。

| 階段 | 瓶頸 | 為什麼？ | 主要最佳化 |
|-------|------------|------|----------------------|
| **Prefill** | Compute (FLOPs) | 平行處理會吃滿 GPU 算術單元。 | FlashAttention、FP8/FP16 precision。 |
| **Decode** | 記憶體頻寬 | 權重必須為*每一個 token*從 VRAM 重新載入。 | Quantization (4-bit)、GQA、Batching。 |

**The Memory Wall 洞察**
隨著模型越來越大，記憶體頻寬（HBM3/HBM3e）的成長速度不如算力（TFLOPS）。因此 Decode 階段成為正式環境最佳化的主要目標。

---

<a id="performance-metrics"></a>
## 效能指標

| 指標 | 全名 | 目標 | 重要性 |
|--------|-----------|------|------------|
| **TTFT** | Time To First Token | < 200ms | 使用者感受到的回應速度。 |
| **TPOT** | Time Per Output Token | < 30ms | 閱讀速度與對話流暢度。 |
| **Throughput** | Tokens/Second (Agg) | 越高越好 | 決定每次查詢成本。 |
| **Latency** | End-to-End Time | < 2.0s | Agent 的總往返時間。 |

---

<a id="hardware-enabled-optimizations-fp8"></a>
## 硬體加持的最佳化（FP8）

**FP8（8-bit Floating Point）** 是 H100 與 B200 GPU 上的推論原生精度。

- **Benefit**：比 FP16/BF16 快 2 倍，且準確率損失可忽略（<0.1%）。
- **How it works**：它使用較小 mantissa 與較大 exponent，相比 Int8 更能精準表達 LLM activations 的動態範圍，而不需要複雜校準。

**Principal-level 細節**：現在的 serving frameworks 已採用 **Dynamic FP8 Scaling**，可逐層調整量化尺度，避免少數 outliers 拉低整體模型邏輯。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-is-llm-generation-slower-than-classification"></a>
### Q: 為什麼 LLM 生成會比分類慢？

**強答：**
分類屬於「Prefill-only」任務；它會在一次平行 pass 中處理整個輸入並產生單一輸出，因此是 compute-optimal。LLM 生成則是**自回歸**的，每個 token 都依賴前一個 token，迫使系統進入序列式的 Decode 迴圈。因為這個迴圈中的每一步都受限於記憶體頻寬（載入數 GB 權重，只為了產生數 byte 資料），所以大部分時間都花在等待記憶體傳輸，而不是做數學運算。

<a id="q-how-do-you-optimize-ttft-vs-tpot"></a>
### Q: 你會如何分別最佳化 TTFT 與 TPOT？

**強答：**
若要最佳化 **TTFT**，就必須最佳化 Prefill 階段：使用 FlashAttention-3、提高計算平行度（Tensor Parallelism），或使用 Prefix Caching 直接跳過 prefill。若要最佳化 **TPOT**，則必須最佳化 Decode 階段的記憶體頻寬：使用量化（4-bit 權重）降低從 VRAM 移動的資料量、使用 Grouped Query Attention (GQA) 降低 KV cache 大小，或用 Speculative Decoding 一次產生多個 token。

---

<a id="references"></a>
## 參考資料
- Pope et al. "Efficiently Scaling Transformer Inference" (2022)
- NVIDIA. "Transformer Engine Documentation" (2024)
- vLLM Blog. "Understanding LLM Inference Latency" (2023)

---

*下一篇：[KV Cache and Context Caching](02-kv-cache-and-context-caching.md)*

<a id="speculative-decoding"></a>
# 推測式解碼

推測式解碼已成為標準技術，能讓大型 Models（LLMs）在一次 forward pass 中生成多個 tokens，實質打破序列式解碼的記憶體頻寬瓶頸。

<a id="table-of-contents"></a>
## 目錄

- [核心概念](#the-core-concept)
- [Draft-Verify 範式](#draft-verify)
- [Medusa 與 Multi-Token Heads](#medusa)
- [Lookahead Decoding](#lookahead-decoding)
- [Hardware-Aware Speculation](#hardware-aware)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-core-concept"></a>
## 核心概念

LLM 解碼受到記憶體頻寬限制：載入 140GB 權重（70B 模型）只為了產生單一 2-byte token，其實很沒效率。
**Speculative Decoding** 會用一種成本更低的方法「猜測」接下來的 $N$ 個 tokens，再讓大型模型用單次平行、類似「Prefill-style」的 pass 一次驗證它們。

---

<a id="draft-verify-paradigm"></a>
## Draft-Verify 範式

1. **Drafting**：一個小而快的「Draft Model」（例如 1B 或 7B）先生成 $K$ 個候選 tokens。
2. **Verification**：大型「Target Model」一次處理全部 $K$ 個 tokens。
3. **Acceptance**：使用 target model 的 logits 來接受或拒絕候選。如果 token $i$ 被拒絕，後面所有 token 都會丟棄。

| Model | 大小 | 速度 | 每 token 延遲 |
|-------|------|-------|-------------------|
| **Draft** | 1B | 快 | 5ms |
| **Target**| 70B| 慢 | 50ms |
| **Speculative**| - | **快**| **15ms - 25ms** |

**整體結果**：在**零品質損失**下，實際時間可加速 2x 到 3x。

---

<a id="medusa-multi-token-heads"></a>
## Medusa 與 Multi-Token Heads

產業已逐漸放棄獨立 draft models（它們會增加 VRAM 負擔），改採 **Medusa Heads**。

- **What it is**：附加在 target model 最後一層上的額外「heads」（小型線性層）。
- **How it works**：不只預測 token $t+1$；Head 1 預測 $t+1$，Head 2 預測 $t+2$，依此類推。
- **Benefit**：不需要第二個模型；只增加少量 VRAM，卻能換到 2.5x 加速。

---

<a id="lookahead-decoding"></a>
## Lookahead Decoding

另一種方法是利用模型過去的 hidden states 找出重複模式（n-grams），藉此「往前看」並預測未來 tokens。
- **Best For**：結構化資料、程式碼，以及高度重複的技術寫作。

---

<a id="hardware-aware-speculation"></a>
## Hardware-Aware Speculation

前沿 serving frameworks（vLLM、TensorRT-LLM）現在會使用 **Dynamic Draft Lengths**。
- 若 GPU 利用率偏低（batch 小），系統就增加 draft tokens 數量（$K$）。
- 若 GPU 已飽和（batch 大），就降低 $K$，優先追求總 throughput，而非單一請求延遲。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-doesnt-speculative-decoding-work-well-for-high-temperature-creative-writing"></a>
### Q: 為什麼推測式解碼不適合高 temperature 的創意寫作？

**強答：**
推測式解碼依賴「Draft Model」能準確預測「Target Model」會說什麼。在高 temperature 的創意寫作中，機率分布會更平坦，模型也更傾向選擇低機率 token。因此 **Acceptance Rate** 會很低（draft model 的猜測常被拒絕）。一旦猜錯，target model 那次平行驗證就等於白做，系統還會退回標準序列式解碼，反而多出 draft model 的延遲成本。

<a id="q-how-does-medusa-differ-from-traditional-speculative-decoding"></a>
### Q: Medusa 和傳統推測式解碼有何不同？

**強答：**
傳統推測式解碼需要額外一個較小模型（Draft Model），因此會占用更多 VRAM，並帶來獨立的 KV cache 管理成本。Medusa 則是在 base model 最後 hidden state 上增加多個「heads」，每個 head 負責預測不同 offset（例如下一個 token、下下個 token）。這消除了第二模型需求，也把步驟間的通訊成本降到最低，因為所有「猜測」都在同一個 base model 架構中的單次 forward pass 內完成。

---

<a id="references"></a>
## 參考資料
- Chen et al. "Accelerating Transformer Decoding via Speculative Decoding" (2023)
- Cai et al. "Medusa: Simple LLM Acceleration via Multiple Decoding Heads" (2024)
- Fu et al. "Lookahead Decoding" (2024)

---

*下一篇：[Batching Strategies](04-batching-strategies.md)*

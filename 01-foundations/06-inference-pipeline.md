<a id="inference-pipeline"></a>
# 推論流程

本章涵蓋 LLM 在推論時如何生成文字、其中涉及的運算階段，以及正式上線服務時的重要指標。

<a id="table-of-contents"></a>
## 目錄

- [生成基礎](#generation-basics)
- [Prefill 與 Decode 階段](#prefill-and-decode-phases)
- [取樣策略](#sampling-strategies)
- [停止條件](#stopping-conditions)
- [潛在優化：Speculative Decoding](#speculative-decoding)
- [延遲指標與 TTFT vs. TPS](#latency-metrics)
- [記憶體與運算需求](#memory-and-compute-requirements)
- [Continuous Batching 與 Prefix Caching](#continuous-batching-and-prefix-caching)
- [Multi-LoRA Serving](#multi-lora-serving)
- [串流](#streaming)
- [正式環境考量](#production-considerations)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="generation-basics"></a>
## 生成基礎

LLM 會以 autoregressive 的方式生成文字：一次產生一個 token，並使用所有先前 token 作為上下文。

```
Input: "The quick brown"
Step 1: Generate "fox" -> "The quick brown fox"
Step 2: Generate "jumps" -> "The quick brown fox jumps"
Step 3: Generate "over" -> "The quick brown fox jumps over"
...
```

<a id="the-generation-loop"></a>
### 生成迴圈

```python
def generate(prompt: str, max_tokens: int, model) -> str:
    tokens = tokenize(prompt)
    
    for _ in range(max_tokens):
        # Forward pass: get logits for next token
        logits = model.forward(tokens)
        
        # Sample next token from probability distribution
        next_token = sample(logits[-1])
        
        # Check for stop condition
        if next_token == EOS_TOKEN:
            break
        
        tokens.append(next_token)
    
    return detokenize(tokens)
```

---

<a id="prefill-and-decode-phases"></a>
## Prefill 與 Decode 階段

推論分成兩個特性不同的階段：

<a id="prefill-phase"></a>
### Prefill 階段

會平行處理整個輸入 prompt。

```
Input: "The quick brown fox" (4 tokens)

Prefill:
- Process all 4 tokens simultaneously
- Compute attention across all pairs
- Populate KV cache for all positions
- Output: logits for next token
```

**特性：**
- Compute-bound（大量矩陣運算）
- 可在 token 間平行化
- 時間會隨 prompt 長度增加
- 每次生成只會發生一次

<a id="decode-phase"></a>
### Decode 階段

一次生成一個 token。

```
Decode step 1:
- Input: new token position only
- Attend to all KV cache (prompt + previously generated)
- Generate one token

Decode step 2:
- Append new K, V to cache
- Input: newest token position
- Generate next token

...repeat until done
```

**特性：**
- Memory-bound（需從 HBM 載入 KV cache）
- 序列化進行（每一步完成後下一步才能開始）
- 每個 token 的時間大致固定
- 直到滿足停止條件前都會重複

<a id="why-this-matters"></a>
### 為何這很重要

| 階段 | 瓶頸 | 最佳化 |
|-------|------------|--------------|
| Prefill | Compute（GPU cores） | Flash Attention、更好的 GPU |
| Decode | 記憶體頻寬 | GQA、batching、quantization |

**對 serving 的意涵：**
- 長 prompt 會拉長 prefill 時間（影響 TTFT）
- 長生成會拉長 decode 時間（影響總延遲）
- Batching 對 decode 效率的幫助大於 prefill

---

<a id="sampling-strategies"></a>
## 取樣策略

在算出 logits 之後，我們需要選出下一個 token。不同策略會產生不同輸出。

<a id="greedy-decoding"></a>
### Greedy Decoding

永遠選機率最高的 token：

```python
def greedy_sample(logits):
    return torch.argmax(logits)
```

**特性：**
- 決定性
- 長生成時常會重複
- 適合事實型／結構化輸出

<a id="temperature-sampling"></a>
### Temperature Sampling

在 softmax 前縮放 logits，以控制隨機性：

```python
def temperature_sample(logits, temperature=1.0):
    scaled_logits = logits / temperature
    probs = torch.softmax(scaled_logits, dim=-1)
    return torch.multinomial(probs, num_samples=1)
```

**Temperature 的效果：**

| Temperature | 行為 | 使用情境 |
|-------------|----------|----------|
| 0 | Greedy（決定性） | 事實型 Q&A、程式碼 |
| 0.3-0.7 | 低隨機性 | 一般任務 |
| 1.0 | 基準 | 創意寫作 |
| 1.5+ | 高隨機性 | 腦力激盪 |

<a id="top-k-sampling"></a>
### Top-K Sampling

只考慮機率最高的 K 個 token：

```python
def top_k_sample(logits, k=50):
    values, indices = torch.topk(logits, k)
    probs = torch.softmax(values, dim=-1)
    sampled_idx = torch.multinomial(probs, num_samples=1)
    return indices[sampled_idx]
```

**效果：** 過濾掉低機率、可能毫無意義的 token。

<a id="top-p-nucleus-sampling"></a>
### Top-P（Nucleus）Sampling

納入 token 直到累積機率超過 P：

```python
def top_p_sample(logits, p=0.9):
    sorted_probs, sorted_indices = torch.sort(
        torch.softmax(logits, dim=-1), descending=True
    )
    cumulative_probs = torch.cumsum(sorted_probs, dim=-1)
    
    # Find cutoff
    cutoff_idx = torch.searchsorted(cumulative_probs, p)
    
    # Sample from truncated distribution
    selected_probs = sorted_probs[:cutoff_idx + 1]
    selected_probs = selected_probs / selected_probs.sum()
    sampled_idx = torch.multinomial(selected_probs, num_samples=1)
    
    return sorted_indices[sampled_idx]
```

**相較於 Top-K 的優點：** 會根據機率分布動態調整。高信心預測只納入較少 token；不確定時則會納入更多。

<a id="common-configurations"></a>
### 常見設定

| 使用情境 | Temperature | Top-P | Top-K |
|----------|-------------|-------|-------|
| 程式碼生成 | 0-0.2 | 0.95 | - |
| 事實型 Q&A | 0.1-0.3 | 1.0 | - |
| 一般聊天 | 0.7 | 0.9 | - |
| 創意寫作 | 1.0 | 0.95 | - |
| 腦力激盪 | 1.2 | 1.0 | - |

<a id="repetition-penalties"></a>
### 重複懲罰

降低最近生成過 token 的機率：

```python
def apply_repetition_penalty(logits, generated_tokens, penalty=1.2):
    for token_id in set(generated_tokens):
        logits[token_id] /= penalty
    return logits
```

**變體：**
- Presence penalty：懲罰所有出現過的 token
- Frequency penalty：依出現次數成比例懲罰

---

<a id="stopping-conditions"></a>
## 停止條件

生成會持續進行，直到滿足某個停止條件：

<a id="eos-token"></a>
### EOS Token

模型生成序列結束 token：

```python
if next_token == tokenizer.eos_token_id:
    break
```

<a id="max-tokens"></a>
### Max Tokens

對生成長度設下硬上限：

```python
for i in range(max_tokens):
    # generate...
```

<a id="stop-sequences"></a>
### Stop Sequences

可自訂用來終止生成的字串：

```python
stop_sequences = ["###", "

", "Human:"]

for seq in stop_sequences:
    if output.endswith(seq):
        output = output[:-len(seq)]
        break
```

<a id="latent-optimization-speculative-decoding"></a>
## 潛在優化：Speculative Decoding

**2025 年高頻寬 serving 的標準做法。**

Speculative decoding 會用較小的「draft model」一次預測多個未來 token，再由較大的「target model」平行驗證。

```
Draft Model (Small): Predicts 5 tokens -> "The", "quick", "brown", "fox", "jumps"
Target Model (Large): Verifies all 5 tokens in ONE forward pass.
Result: If target agrees on 4 tokens, we've generated 4 tokens for the cost of 1 large forward pass.
```

| 方法 | 作法 | 加速比 | 範例 |
|--------|----------|---------|---------|
| Draft Model | 小模型（例如 1B）+ 大模型（70B） | 2x-3x | vLLM, TGI |
| **Medusa Heads** | 同一模型上的多個 LM heads | 1.5x-2x | Medusa, Eagle |
| Prompt Lookup | 使用 prompt 中的子字串來做 speculative decoding | 1.2x | RAG / Code completion |

---

<a id="latency-metrics"></a>
## 延遲指標

<a id="time-to-first-token-ttft"></a>
### Time to First Token（TTFT）

從收到請求到產生第一個 token 的時間。

```
TTFT = network_latency + queue_time + prefill_time
```

**影響 TTFT 的因素：**
- Prompt 長度（prefill 是 O(n)）
- 模型大小
- GPU 速度
- 佇列深度

**目標值：**
- 互動式聊天：< 500ms
- 即時應用：< 200ms
- Batch：較不關鍵

<a id="tokens-per-second-tps"></a>
### Tokens Per Second（TPS）

第一個 token 之後的生成速率。

```
TPS = (total_tokens - 1) / (total_time - TTFT)
```

**影響 TPS 的因素：**
- 模型大小
- Batch size
- GPU 記憶體頻寬
- KV cache 大小

**典型數值：**
- H100 上的 Llama 70B：每個 request 30-50 tokens/sec
- 透過 API 的 GPT-4：20-80 tokens/sec（會變動）
- 小模型（7B）：100+ tokens/sec

<a id="total-latency"></a>
### 總延遲

```
Total = TTFT + (output_tokens / TPS)
```

**範例：**
- TTFT：200ms
- TPS：50 tokens/sec
- 輸出：100 tokens
- 總計：200ms + 2000ms = 2.2s

<a id="throughput"></a>
### 吞吐量

單位時間內完成的 request 數：

```
Throughput = concurrent_requests * TPS / average_output_tokens
```

較大的 batch size 能提升吞吐量，但也可能增加單一 request 的延遲。

---

<a id="memory-and-compute-requirements"></a>
## 記憶體與運算需求

<a id="model-weights"></a>
### 模型權重

```
Memory = parameters * bytes_per_parameter

70B model in FP16:
= 70B * 2 bytes
= 140 GB

70B model in INT4:
= 70B * 0.5 bytes
= 35 GB
```

<a id="kv-cache"></a>
### KV Cache

```
Per token: 2 * layers * heads * head_dim * bytes
Per request: per_token * sequence_length

Llama 70B (80 layers, 64 heads, 128 dim, FP16):
= 2 * 80 * 64 * 128 * 2 bytes
= 2.6 MB per token

At 4K context: 10.5 GB per request
At 8K context: 21 GB per request
```

<a id="total-gpu-memory"></a>
### 總 GPU 記憶體

```
Total = model_weights + kv_cache * batch_size + activations

Example: Llama 70B serving
- Weights (INT4): 35 GB
- KV cache (8K, batch 4): 84 GB
- Activations: ~5 GB
- Total: ~124 GB (fits on 2x H100 80GB)
```

<a id="flops-per-token"></a>
### 每個 Token 的 FLOPs

```
Forward pass FLOPs ≈ 2 * parameters

70B model:
≈ 140 TFLOPs per token

At 40 tokens/sec:
≈ 5.6 PFLOPs sustained
```

---

<a id="streaming"></a>
## 串流

對互動式應用而言，應在 token 生成時立即串流回傳：

<a id="server-side-events-sse"></a>
### Server-Side Events（SSE）

```python
# Server
async def generate_stream(prompt: str):
    for token in model.generate_iter(prompt):
        yield f"data: {json.dumps({'token': token})}

"
    yield "data: [DONE]

"

# Client
async for event in sse_client.stream("/generate"):
    token = json.loads(event.data)["token"]
    display(token)
```

<a id="benefits"></a>
### 優點

| 面向 | 串流 | 非串流 |
|--------|-----------|---------------|
| 感知延遲 | 只有 TTFT | 完整生成時間 |
| 使用者體驗 | 漸進顯示 | 等待後一次完成 |
| 提前終止 | 使用者可中止 | 必須等待 |
| 記憶體 | 較低 | 較高（需緩衝回應） |

<a id="implementation-details"></a>
### 實作細節

- 每個 token 後都要 flush
- 妥善處理連線中斷
- 對極快的生成可考慮 buffering
- 某些 framework 預設會 buffer；做串流時要停用

---

<a id="production-considerations"></a>
## 正式環境考量

<a id="batching-for-throughput"></a>
### 為吞吐量進行 Batching

合併多個 request，以最大化 GPU 利用率：

```python
# Without batching: GPU underutilized
for request in requests:
    response = model.generate(request)

# With batching: parallel processing
batch = collect_requests(timeout=10ms, max_batch=32)
responses = model.generate_batch(batch)
```

<a id="continuous-batching-and-prefix-caching"></a>
### Continuous Batching 與 Prefix Caching

**Continuous Batching（逐 iteration 排程）：**
不同於靜態 batching，continuous batching 會在批次中任一 request 命中 EOS token 時，立刻插入新 request。這能把吞吐量提升到最多 20 倍。

**Prefix Caching（RAD-O）：**
會快取常見前綴（例如 system prompts、few-shot examples）的 KV tensors。
- **TTFT 降幅**：90%
- **機制**：使用前綴的 hash 值，在 GPU 記憶體中的 LRU cache 查找 KV tensors。

<a id="multi-lora-serving"></a>
### Multi-LoRA Serving

**情境：** 在一個 base model 上服務 1000 個不同的 fine-tuned models（adapters）。
**挑戰：** 若載入 1000 個獨立模型，會消耗 TB 等級的 VRAM。

**解法（LoRAX / S-LoRA）：**
1. 在 VRAM 中載入一個 base model。
2. 將 LoRA adapters（MB 等級）存放在 host RAM 或 SSD。
3. 根據 request ID，在 forward pass 期間動態切換 adapter。
4. **實作**：使用專門的 kernel（S-LoRA），在同一個 batch 中對多個不同 adapter 執行 matrix-vector multiplication。

<a id="request-prioritization"></a>
### 請求優先順序

```python
class RequestQueue:
    def __init__(self):
        self.high_priority = asyncio.Queue()
        self.low_priority = asyncio.Queue()
    
    async def get_next(self):
        if not self.high_priority.empty():
            return await self.high_priority.get()
        return await self.low_priority.get()
```

**優先條件：**
- 客戶等級
- Request 類型
- 等待時間
- 預估運算成本

<a id="timeout-handling"></a>
### Timeout 處理

```python
async def generate_with_timeout(prompt: str, timeout: float):
    try:
        result = await asyncio.wait_for(
            model.generate(prompt),
            timeout=timeout
        )
        return result
    except asyncio.TimeoutError:
        return {"error": "timeout", "partial": partial_output}
```

<a id="graceful-degradation"></a>
### 優雅降級

```python
async def generate_with_fallback(prompt: str):
    try:
        return await primary_model.generate(prompt)
    except RateLimitError:
        return await fallback_model.generate(prompt)
    except TimeoutError:
        return await small_fast_model.generate(prompt)
```

<a id="cost-tracking"></a>
### 成本追蹤

```python
@dataclass
class RequestMetrics:
    input_tokens: int
    output_tokens: int
    model: str
    latency_ms: float
    cost_usd: float

def calculate_cost(metrics: RequestMetrics) -> float:
    pricing = {
        "gpt-4o": {"input": 2.50, "output": 10.00},
        "gpt-4o-mini": {"input": 0.15, "output": 0.60},
    }
    rates = pricing[metrics.model]
    return (
        (metrics.input_tokens / 1_000_000) * rates["input"] +
        (metrics.output_tokens / 1_000_000) * rates["output"]
    )
```

---

<a id="interview-questions"></a>
## 面試題

<a id="q-explain-the-difference-between-prefill-and-decode-phases"></a>
### 問：說明 prefill 與 decode 階段的差異。

**理想答案：**
LLM 推論有兩個截然不同的階段：

**Prefill：**
- 一次處理整個輸入 prompt
- 所有 token 彼此平行做 attention
- 為所有 prompt 位置填入 KV cache
- Compute-bound：能有效使用 GPU cores
- 時間隨 prompt 長度增加

**Decode：**
- 一次生成一個 token
- 新 token 會對所有 KV cache 項目做 attention
- 將新的 K、V 附加到 cache
- Memory-bound：瓶頸在於載入 KV cache
- 每個 token 的時間大致固定

這對系統設計很重要，因為：
- 長 prompt 會增加 TTFT（prefill 密集）
- Batching 對 decode 的幫助大於 prefill
- 兩個階段的最佳化策略不同

<a id="q-how-do-temperature-and-top-p-affect-generation"></a>
### 問：temperature 與 top-p 如何影響生成？

**理想答案：**
兩者都用來控制 token 選擇的隨機性：

**Temperature：**
- 在 softmax 前縮放 logits
- 低（0-0.3）：較具決定性，偏向高機率 token
- 高（1.0+）：更隨機，會拉平機率分布
- 零：等同 greedy decoding

**Top-p（nucleus sampling）：**
- 只保留累積機率 > p 的最小 token 集合
- 會根據分布動態調整截斷點
- 高信心：考慮較少 token
- 低信心：考慮較多 token

常見正式環境設定：
- 事實型 Q&A：temperature 0.1、top-p 0.95
- 一般聊天：temperature 0.7、top-p 0.9
- 創意任務：temperature 1.0+、top-p 0.95

關鍵洞察是兩者會一起作用。Temperature 會重塑分布；top-p 則會截斷分布。

<a id="q-what-determines-ttft-vs-tps"></a>
### 問：TTFT 與 TPS 分別由什麼決定？

**理想答案：**
**TTFT（Time to First Token）：**
- 到達伺服器的網路延遲
- 佇列等待時間
- Prefill 計算時間
- 主導因素：prompt 長度、GPU 計算速度

**TPS（Tokens Per Second）：**
- Decode 階段效率
- 載入 KV cache 的記憶體頻寬
- 主導因素：記憶體頻寬、batch size、模型大小

最佳化策略不同：
- TTFT：可行時縮短 prompt、使用更快的網路、降低排隊時間
- TPS：提高 batch size、使用 GQA/MQA 模型、最佳化記憶體存取

取捨在於：batching 能提升 TPS（吞吐量），但若 request 需要等待成 batch，也可能增加 TTFT（延遲）。

<a id="q-how-would-you-estimate-gpu-requirements-for-serving-a-model"></a>
### 問：你會如何估算服務模型所需的 GPU 資源？

**理想答案：**
有三個主要的記憶體消耗來源：

1. **模型權重：**
   - FP16：parameters * 2 bytes
   - INT8：parameters * 1 byte
   - INT4：parameters * 0.5 bytes

2. **KV cache：**
   - 每個 token：2 * layers * kv_heads * head_dim * 2 bytes（FP16）
   - 每個 request：per_token * sequence_length
   - 總量：per_request * batch_size

3. **Activations：** 通常額外增加 5-10%

以服務 Llama 70B 為例：
- 權重（INT4）：35 GB
- KV cache（8K context，batch 8）：168 GB
- 需求：約 200 GB 總量

硬體選項：
- 3x A100 80GB，搭配 tensor parallelism
- 2x H100 80GB，搭配 tensor parallelism
- 8x A100 40GB，使用更多平行化

接著再用 benchmark 驗證吞吐量是否符合需求。

---

<a id="references"></a>
## 參考資料

- Holtzman et al. "The Curious Case of Neural Text Degeneration" (nucleus sampling, 2020)
- Kwon et al. "Efficient Memory Management for Large Language Model Serving with PagedAttention" (vLLM, 2023)
- [vLLM Documentation](https://docs.vllm.ai/)
- [TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM)
- [OpenAI API Documentation](https://platform.openai.com/docs/api-reference)

---

*上一章：[Embeddings and Vector Spaces](05-embeddings-and-vector-spaces.md) | 下一章：[Model Taxonomy](../02-model-landscape/01-model-taxonomy.md)*

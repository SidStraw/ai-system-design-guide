<a id="attention-mechanisms"></a>
# 注意力機制

注意力是讓 transformer 成立的核心創新。本章涵蓋數學基礎、常見變體，以及對系統設計與面試至關重要的最佳化方式。

<a id="table-of-contents"></a>
## 目錄

- [注意力基礎](#attention-fundamentals)
- [縮放點積注意力](#scaled-dot-product-attention)
- [多頭注意力](#multi-head-attention)
- [注意力模式](#attention-patterns)
- [高效率注意力變體](#efficient-attention-variants)
- [Flash Attention（v2 與 v3）](#flash-attention)
- [Multi-head Latent Attention（MLA）](#multi-head-latent-attention-mla)
- [KV Cache 最佳化與 Context Caching](#kv-cache-optimizations)
- [實務影響](#practical-implications)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="attention-fundamentals"></a>
## 注意力基礎

<a id="the-core-idea"></a>
### 核心概念

注意力讓序列中的每個位置都能從其他所有位置蒐集資訊。和 recurrence（逐步傳遞資訊）不同，注意力會直接建立位置之間的連結。

**給分散式系統工程師的心智模型：**
- RNN：沿著鏈條進行 message passing
- Attention：像是 pub/sub，每個節點都能查詢其他所有節點

<a id="query-key-value-framework"></a>
### Query、Key、Value 架構

注意力會對輸入做三種投影：

| 元件 | 角色 | 類比 |
|-----------|------|---------|
| Query (Q) | 我在找什麼？ | Search query |
| Key (K) | 我包含什麼？ | 文件索引 |
| Value (V) | 我能貢獻什麼？ | 文件內容 |

```python
# Input: x of shape [batch, seq_len, d_model]

Q = x @ W_q  # [batch, seq_len, d_k]
K = x @ W_k  # [batch, seq_len, d_k]
V = x @ W_v  # [batch, seq_len, d_v]
```

---

<a id="scaled-dot-product-attention"></a>
## 縮放點積注意力

最基本的注意力運算如下：

```python
def scaled_dot_product_attention(Q, K, V, mask=None):
    d_k = Q.shape[-1]
    
    # Compute attention scores
    scores = Q @ K.transpose(-2, -1)  # [batch, seq_len, seq_len]
    scores = scores / math.sqrt(d_k)  # Scale
    
    # Apply mask (for causal attention)
    if mask is not None:
        scores = scores.masked_fill(mask == 0, float('-inf'))
    
    # Convert to probabilities
    attention_weights = F.softmax(scores, dim=-1)
    
    # Weighted sum of values
    output = attention_weights @ V
    
    return output, attention_weights
```

<a id="why-scale-by-square-root-of-d_k"></a>
### 為什麼要除以 d_k 的平方根？

**面試常考題**：這題在測試你對數值直覺的理解。

如果不做縮放，點積會隨維度增長：
- 對於維度為 d 的隨機單位向量 q 與 k
- E[q . k] = 0，但 Var[q . k] = d
- 標準差 = sqrt(d)

當 d 很大（512 以上）時，點積可能變得非常大或非常小。Softmax 遇到大數值時會趨近 one-hot，導致 gradient vanishing。

```python
# Demonstration
import numpy as np

d = 512
q = np.random.randn(d)
k = np.random.randn(d)

unscaled = np.dot(q, k)      # Magnitude ~ sqrt(512) ~ 22
scaled = unscaled / np.sqrt(d)  # Magnitude ~ 1
```

<a id="causal-masking"></a>
### 因果遮罩

在 autoregressive generation 中，每個位置只能注意到先前的位置：

```python
def create_causal_mask(seq_len):
    # Lower triangular matrix
    mask = torch.tril(torch.ones(seq_len, seq_len))
    return mask

# Example for seq_len=4:
# [[1, 0, 0, 0],
#  [1, 1, 0, 0],
#  [1, 1, 1, 0],
#  [1, 1, 1, 1]]
```

mask=0 的位置會得到負無限大的分數，經過 softmax 後就會變成 0。

---

<a id="multi-head-attention"></a>
## 多頭注意力

與其只用一個注意力函式，不如使用多個會關注不同面向的「head」：

```python
class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads):
        super().__init__()
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
    
    def forward(self, x, mask=None):
        batch_size, seq_len, d_model = x.shape
        
        # Project to Q, K, V
        Q = self.W_q(x)  # [batch, seq_len, d_model]
        K = self.W_k(x)
        V = self.W_v(x)
        
        # Reshape to multiple heads
        Q = Q.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        K = K.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        V = V.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        # Now: [batch, num_heads, seq_len, d_k]
        
        # Attention per head
        attn_output, _ = scaled_dot_product_attention(Q, K, V, mask)
        
        # Concatenate heads
        attn_output = attn_output.transpose(1, 2).contiguous()
        attn_output = attn_output.view(batch_size, seq_len, d_model)
        
        # Final projection
        output = self.W_o(attn_output)
        return output
```

**為什麼要多個 head？**
1. 不同 head 會學到不同模式（syntax、semantics、coreference）
2. 提供表徵多樣性（類似 ensemble effect）
3. 能在 head 之間平行計算

<a id="head-count-patterns"></a>
### 頭數模式

| 模型 | d_model | heads | 每個 head 的 d_k |
|-------|---------|-------|--------------|
| BERT-base | 768 | 12 | 64 |
| GPT-2 | 768 | 12 | 64 |
| GPT-3 175B | 12288 | 96 | 128 |
| Llama 2 70B | 8192 | 64 | 128 |

64 或 128 的 d_k 在不同模型規模之間非常一致。

---

<a id="attention-patterns"></a>
## 注意力模式

<a id="what-attention-learns"></a>
### 注意力會學到什麼

不同的 head 會專精於不同模式：

| 模式類型 | 它捕捉什麼 | 例子 |
|--------------|------------------|---------|
| 位置 | 相鄰 token | 前一個／下一個字詞 |
| 語法 | 文法關係 | 主詞－動詞 |
| 語意 | 意義關係 | Coreference |
| 分隔符 | 標點、結構 | 章節邊界 |
| 稀有 | 不常見模式 | 稀有詞複製 |

<a id="visualizing-attention"></a>
### 視覺化注意力

注意力權重可以畫成 heatmap，顯示哪些位置在關注哪些位置：

```
Query positions (rows) vs Key positions (columns)

"The cat sat on the mat"

         The  cat  sat  on   the  mat
The     [□    ○    ○    ○    ○    ○ ]
cat     [●    □    ○    ○    ○    ○ ]
sat     [○    ●    □    ○    ○    ○ ]
on      [○    ○    ●    □    ○    ○ ]
the     [○    ○    ○    ○    □    ○ ]
mat     [○    ●    ○    ●    ●    □ ]

● = high attention, ○ = low attention
```

「mat」會強烈關注「cat」（語意）、「on」（語法）與「the」（限定詞）。

---

<a id="efficient-attention-variants"></a>
## 高效率注意力變體

標準注意力在序列長度上是 O(n^2)。許多變體會將它降低為：

<a id="sparse-attention"></a>
### 稀疏注意力

只注意部分位置，而不是全部位置：

| 變體 | 模式 | 複雜度 | 例子 |
|---------|---------|------------|---------|
| Local | 每個位置周圍的視窗 | O(n * w) | Longformer |
| Strided | 每隔 k 個位置 | O(n^2 / k) | Sparse Transformer |
| Global | 特殊 token 可注意所有位置 | O(n * g) | Longformer, BigBird |
| Block | 區塊對角注意力 | O(n * b) | BigBird |

**Longformer 模式：**
```
Local window + Global tokens

[G] [L] [L] [L] [L] [G] [L] [L] [L] [L]

G: Global tokens (attend to/from all)
L: Local tokens (attend within window)
```

<a id="linear-attention"></a>
### 線性注意力

以可線性化的替代方案取代 softmax：

```python
# Standard attention (quadratic)
attention = softmax(Q @ K.T) @ V

# Linear attention approximation
attention = (Q @ (K.T @ V))  # Associativity trick
```

**常見變體：**
- Performer：隨機特徵近似
- Linear Transformer：elu(Q) @ (elu(K).T @ V)

**取捨：**速度更快，但品質會下降，特別是在需要精確注意力的任務上。

<a id="complexity-comparison"></a>
### 複雜度比較

| 方法 | 時間 | 空間 | 品質 | 備註 |
|--------|------|-------|---------|-------|
| Standard | O(n^2) | O(n^2) | 最佳 | 基準線 |
| Sparse (Longformer) | O(n) | O(n) | 接近最佳 | 適合長文件 |
| Linear (Performer) | O(n) | O(n) | 較差 | 適合超長序列 |
| Flash Attention | O(n^2) | O(n) | 最佳 | 兼顧兩者 |

---

<a id="flash-attention"></a>
## Flash Attention

Flash Attention 是目前最先進的實作方式，能在計算精確注意力的同時，把記憶體使用降到 O(n)。

<a id="the-problem-it-solves"></a>
### 它解決的問題

標準注意力需要將 n x n 的注意力矩陣完整 materialize：
- 對 8K context：64M floats = 每層每個 head 256 MB
- 對 100K context：10B floats = 每層每個 head 40 GB

這樣的記憶體需求會限制 batch size 與 context length。

<a id="how-it-works"></a>
### 運作方式

Flash Attention 使用 tiling 與 recomputation，避免儲存完整注意力矩陣：

```
Standard: Q, K -> Attention Matrix (n x n) -> Output
Flash:    Q, K -> Tiles (block_size x block_size) -> Incremental Output
```

**核心想法：**
1. 以能放入 SRAM 的區塊來處理注意力
2. 永遠不要在 HBM 中 materialize 完整注意力矩陣
3. 在 backward pass 時重新計算注意力（比從 HBM 載入更快）

<a id="performance-impact"></a>
### 效能影響

<a id="flashattention-2-work-partitioning"></a>
### FlashAttention-2（工作分割）

透過改善 head 與 sequence length 上的平行度，為 A100/H100 做最佳化。

<a id="flashattention-3-fp8--h100-optimization"></a>
### FlashAttention-3（FP8 與 H100 最佳化）

**2025 年 H100/B200 叢集的標準：**
- **非同步執行**：在 H100 上利用 TMA（Tensor Memory Accelerator）重疊 GEMM（矩陣乘法）與 softmax。
- **FP8 支援**：原生支援 FP8 精度，透過 stochastic rounding 在維持注意力準確度的同時，讓吞吐量相較 FP16 倍增。
- **加速效果**：對長 context prefill 而言，約比 FlashAttention-2 快 ~1.5x-2.0x。

---

<a id="multi-head-latent-attention-mla"></a>
## Multi-head Latent Attention（MLA）

由 DeepSeek（V2/V3）提出，**MLA 是面對極端 KV cache 壓力時，取代 GQA 的現代方案**。

MLA 不只是將 head 分組，而是在寫入 cache 前，先把 Key 與 Value 向量壓縮到**低維 latent space**。

```
Query (Up-projected) ────────┐
                             ▼
Key, Value (Down-projected) ─▶ [Low-dim Latent Cache] ─▶ [Output]
                             ▲
                             └─ Projection Matrices
```

| 指標 | MHA | GQA | MLA (Dec 2025) |
|--------|-----|-----|----------------|
| KV Cache 大小 | 100% | 12.5% | **~5%** |
| 品質 | Baseline | 接近 Baseline | **優於 GQA** |
| 延遲 | Baseline | 更快 | **最快（I/O 更少）** |

**MLA 為何勝出**：它使用「Decoupled Rotary Positional Embeddings」，讓壓縮後的 latent KV 可以不經解碼就直接重用，在長 context generation 時大幅節省記憶體頻寬。

---

<a id="kv-cache-optimizations"></a>
<a id="kv-cache-optimizations--context-caching"></a>
## KV Cache 最佳化與 Context Caching

<a id="context-caching-system-level"></a>
### Context Caching（系統層級）

API provider（OpenAI、Gemini、Anthropic）現在都提供 **Context Caching**。
- **運作方式**：先預計算並儲存長前綴的 KV tensor（例如一本 100k token 的法律書）。
- **好處**：對重複前綴可將 TTFT（Time to First Token）降低 90%，並把成本降 50-90%。

<a id="sliding-window-attention-swa"></a>
### Sliding Window Attention（SWA）

Mistral/Gemma 模型使用它把注意力深度限制在固定視窗內（例如 4096 tokens），避免 KV cache 無限成長。

<a id="multi-query-attention-mqa"></a>
### Multi-Query Attention（MQA）

讓所有 query heads 共用同一組 K 與 V：

```python
# Standard MHA
Q: [batch, num_heads, seq, d_k]  # 32 heads
K: [batch, num_heads, seq, d_k]  # 32 separate K
V: [batch, num_heads, seq, d_k]  # 32 separate V

# MQA
Q: [batch, num_heads, seq, d_k]  # 32 heads
K: [batch, 1, seq, d_k]          # 1 shared K
V: [batch, 1, seq, d_k]          # 1 shared V
```

**效果：**KV cache 大小減少 32 倍，但會有一些品質損失。

<a id="grouped-query-attention-gqa"></a>
### Grouped-Query Attention（GQA）

讓多個 query heads 以群組方式共用 K 與 V：

```python
# GQA with 8 KV heads for 64 query heads (8:1 ratio)
Q: [batch, 64, seq, d_k]  # 64 query heads
K: [batch, 8, seq, d_k]   # 8 KV heads
V: [batch, 8, seq, d_k]   # 8 KV heads

# Each KV head serves 8 query heads
```

**效果：**KV cache 可縮小 8 倍，而且品質損失極小。

**使用 GQA 的模型：**
- Llama 2 70B：64 個 query heads 對 8 個 KV heads
- Mistral 7B：32 個 query heads 對 8 個 KV heads
- Gemma：多種配置

<a id="comparison"></a>
### 比較

| 注意力 | KV Cache | 品質 | 模型 |
|-----------|----------|---------|--------|
| MHA | 完整 | 最佳 | GPT-3 |
| GQA | 常見為 1/8 | 接近最佳 | Llama 2, Mistral |
| MQA | 1/n_heads | 較低 | PaLM, Falcon |

---

<a id="practical-implications"></a>
## 實務影響

<a id="for-system-design"></a>
### 對系統設計的意義

1. **Batch size 與 context 的取捨：**
   - 總 GPU 記憶體 = Model + KV cache * batch_size
   - Context 越長，batch 越小
   - GQA 模型可以服務更多並發請求

2. **延遲預算配置：**
   - Attention 的計算量是 O(n^2)，使用 Flash 時記憶體為 O(n)
   - Prefill（處理 prompt）會隨 prompt 長度擴展
   - Decode（生成）會隨 generated + prompt 長度擴展

3. **記憶體頻寬瓶頸：**
   - Generation 通常是 memory-bound
   - 每個 token 都要載入 KV cache，往往是主成本
   - 較大的 batch 能攤提這個成本

<a id="prefill-vs-decode"></a>
### Prefill 與 Decode

| 階段 | 計算模式 | 瓶頸 |
|-------|-----------------|------------|
| Prefill | 處理所有輸入 token | Compute（GPU cores） |
| Decode | 一次生成一個 token | Memory（bandwidth） |

這也是為什麼 TTFT（time to first token）與 TPS（tokens per second）通常會分開衡量。

<a id="context-length-scaling"></a>
### 上下文長度擴展

| Context | Attention Compute | KV Cache（Llama 70B） |
|---------|-------------------|---------------------|
| 4K | Baseline | 10.7 GB |
| 8K | 4x | 21.5 GB |
| 32K | 64x | 86 GB |
| 128K | 1024x | 344 GB |

長 context 通常需要：
- Flash Attention（節省記憶體）
- GQA 或 MQA（較小的 KV cache）
- 視情況採用 model parallelism

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-explain-the-attention-mechanism-and-why-it-scales-quadratically"></a>
### 問：請解釋注意力機制，以及它為何呈二次方擴展。

**強答範例：**
注意力會計算所有位置兩兩之間的互動。對 n 個位置而言：

1. Q @ K^T 會產生一個 n x n 的分數矩陣
2. 每個注意力分數都是 query 與 key 的點積
3. 總數：n^2 個點積

因此它會隨序列長度呈二次方成長。對 8K tokens 來說，每層每個 head 需要 6400 萬個成對分數；對 128K tokens 則是 160 億。

這種二次方擴展會限制 context length。常見解法包括：
- Flash Attention：計算是 O(n^2)，但記憶體是 O(n)
- Sparse attention：透過只關注子集合，把成本降為 O(n)
- Linear attention：使用 O(n) 的近似方法

<a id="q-what-is-the-kv-cache-and-why-is-it-critical-for-serving"></a>
### 問：什麼是 KV cache？它為何對 serving 如此關鍵？

**強答範例：**
在 autoregressive generation 中，我們一次只產生一個 token。如果沒有快取，每個新 token 都要重新計算所有先前位置的 K 與 V。

KV cache 會儲存先前位置的 K 與 V tensor。對每個新 token：
1. 只為新位置計算 Q、K、V
2. 把新的 K、V 追加到 cache
3. 對完整快取中的 K、V 做注意力運算

這會把每個 token 在 projection 計算上的複雜度，從 O(n) 降到 O(1)。

代價是記憶體：KV cache 會隨序列長度線性成長。以 Llama 70B 的 8K context 為例，每個 request 約需要 21 GB。這會直接限制 batch size 與 throughput。

GQA 與 MQA 透過讓 query heads 共用 K、V 來降低這個成本。

<a id="q-compare-mha-gqa-and-mqa"></a>
### 問：比較 MHA、GQA 與 MQA。

**強答範例：**
| 變體 | K、V heads | KV Cache | 品質 | 使用情境 |
|---------|-----------|----------|---------|----------|
| MHA | 與 Q heads 相同 | 完整 | 最佳 | 訓練、品質優先 |
| GQA | 少於 Q heads | 較小 | 接近 MHA | 生產 serving |
| MQA | 1 | 最小 | 較低 | 記憶體受限 |

MHA：每個 query head 都有自己的 K 與 V。品質最好，但 KV cache 最大。

GQA：一組 query heads 共用 K 與 V。Llama 2 以 64 個 query heads 對 8 個 KV heads（8:1）。快取小 8 倍，品質損失極小。

MQA：所有 query heads 共用同一組 K 與 V。能節省最多記憶體，但品質下降較明顯。PaLM 採用了這種方式。

對 serving 來說，GQA 是最佳折衷：能支援更大的 batch size（更高 throughput），而品質幾乎與 MHA 相同。

<a id="q-how-does-flash-attention-achieve-on-memory"></a>
### 問：Flash Attention 如何做到 O(n) 記憶體？

**強答範例：**
標準注意力會在 GPU 記憶體中 materialize 完整的 n x n 注意力矩陣。Flash Attention 透過以下方式避免這件事：

1. **Tiling：**以能放入 on-chip SRAM 的 Q 與 K 區塊進行處理
2. **Online softmax：**增量計算 softmax，而不儲存所有分數
3. **Recomputation：**在 backward pass 重新計算注意力，而不是載入先前保存的值

關鍵洞見是：GPU SRAM（每個 SM 約 20 MB）比 HBM（80 GB）快 10 倍。透過在 SRAM 中做更多算術、減少 HBM 讀寫，Flash Attention 不但更快，也更省記憶體。

結果是：在 O(n) 記憶體下得到精確注意力（不是近似值），並可帶來 2-4 倍加速。

---

<a id="references"></a>
## 參考資料

- Vaswani et al. "Attention Is All You Need" (2017)
- Dao et al. "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness" (2022)
- Dao "FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning" (2023)
- Beltagy et al. "Longformer: The Long-Document Transformer" (2020)
- Ainslie et al. "GQA: Training Generalized Multi-Query Transformer Models" (2023)
- Shazeer "Fast Transformer Decoding: One Write-Head is All You Need" (MQA, 2019)
- [Flash Attention Repository](https://github.com/Dao-AILab/flash-attention)

---

*上一章：[Tokenization 深入解析](02-tokenization-deep-dive.md) | 下一章：[Transformer 架構](04-transformer-architecture.md)*

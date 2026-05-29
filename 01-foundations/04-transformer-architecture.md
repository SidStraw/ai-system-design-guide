<a id="transformer-architecture"></a>
# Transformer 架構

本章提供完整 transformer 架構的全貌，將前幾章介紹的元件整合成一致的理解。

<a id="table-of-contents"></a>
## 目錄

- [架構總覽](#architecture-overview)
- [輸入處理](#input-processing)
- [Transformer Block](#the-transformer-block)
- [輸出處理](#output-processing)
- [現代架構變體（Hybrid MoE、MLA）](#mixture-of-experts-moe--hybrid-architectures)
- [Untied 與 Tied Embeddings](#untied-vs-tied-embeddings)
- [擴展特性](#scaling-properties)
- [架構比較表](#architecture-comparison-table)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="architecture-overview"></a>
## 架構總覽

decoder-only transformer（GPT、Claude、Llama 採用的架構）包含：

```
┌─────────────────────────────────────────────────────────────────┐
│                     Token Embeddings                            │
│              + Position Embeddings (or RoPE)                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│    ┌─────────────────────────────────────────────────────┐      │
│    │                  Transformer Block                   │      │
│    │  ┌─────────────────────────────────────────────┐    │      │
│    │  │              RMSNorm/LayerNorm              │    │      │
│    │  └───────────────────┬─────────────────────────┘    │      │
│    │                      ▼                              │      │
│    │  ┌─────────────────────────────────────────────┐    │      │
│    │  │         Masked Multi-Head Attention         │    │      │
│    │  │            (with KV Cache)                  │    │      │
│    │  └───────────────────┬─────────────────────────┘    │      │
│    │                      │                              │      │
│    │                  + Residual                         │      │
│    │                      │                              │      │
│    │  ┌─────────────────────────────────────────────┐    │      │
│    │  │              RMSNorm/LayerNorm              │    │      │
│    │  └───────────────────┬─────────────────────────┘    │      │
│    │                      ▼                              │      │
│    │  ┌─────────────────────────────────────────────┐    │      │
│    │  │             Feed-Forward Network            │    │      │
│    │  │               (SwiGLU/GELU)                 │    │      │
│    │  └───────────────────┬─────────────────────────┘    │      │
│    │                      │                              │      │
│    │                  + Residual                         │      │
│    └──────────────────────┴──────────────────────────────┘      │
│                           │                                     │
│                    Repeat × N layers                            │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Output RMSNorm                             │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Language Model Head                           │
│              (Linear: hidden_dim → vocab_size)                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
                         Logits
```

---

<a id="input-processing"></a>
## 輸入處理

<a id="token-embedding"></a>
### Token Embedding

把 token ID 轉成稠密向量：

```python
class TokenEmbedding(nn.Module):
    def __init__(self, vocab_size, d_model):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, d_model)
    
    def forward(self, token_ids):
        return self.embedding(token_ids)
```

**維度：**
- 輸入：[batch_size, seq_len] token IDs
- 輸出：[batch_size, seq_len, d_model] embeddings

<a id="position-information"></a>
### 位置資訊

位置資訊通常透過以下方式之一加入：

**1. Rotary Position Embedding (RoPE)：**
在 attention 內套用，而不是加到 embedding 上：
```python
def apply_rope(q, k, positions):
    # Rotate q and k vectors based on position
    freqs = compute_frequencies(positions)
    q_rotated = rotate_embeddings(q, freqs)
    k_rotated = rotate_embeddings(k, freqs)
    return q_rotated, k_rotated
```

**2. Learned Position Embeddings：**
直接加到 token embeddings 上：
```python
position_embeddings = nn.Embedding(max_seq_len, d_model)
x = token_embeddings + position_embeddings(positions)
```

**現代模型（Llama、Mistral、GPT-4）會使用 RoPE**，因為它對長度泛化更好。

---

<a id="the-transformer-block"></a>
## Transformer Block

<a id="pre-norm-structure"></a>
### Pre-Norm 結構

現代 transformer 採用 pre-normalization：

```python
class TransformerBlock(nn.Module):
    def __init__(self, config):
        super().__init__()
        self.attn_norm = RMSNorm(config.d_model)
        self.attn = GroupedQueryAttention(
            d_model=config.d_model,
            n_heads=config.n_heads,
            n_kv_heads=config.n_kv_heads
        )
        self.ff_norm = RMSNorm(config.d_model)
        self.ff = SwiGLUFFN(
            d_model=config.d_model,
            d_ff=config.d_ff
        )
    
    def forward(self, x, mask=None, kv_cache=None):
        # Attention with residual
        h = x + self.attn(self.attn_norm(x), mask, kv_cache)
        
        # FFN with residual
        out = h + self.ff(self.ff_norm(h))
        
        return out
```

<a id="attention-component"></a>
### 注意力元件

```python
class GroupedQueryAttention(nn.Module):
    def __init__(self, d_model, n_heads, n_kv_heads):
        super().__init__()
        self.n_heads = n_heads
        self.n_kv_heads = n_kv_heads
        self.head_dim = d_model // n_heads
        
        self.q_proj = nn.Linear(d_model, n_heads * self.head_dim)
        self.k_proj = nn.Linear(d_model, n_kv_heads * self.head_dim)
        self.v_proj = nn.Linear(d_model, n_kv_heads * self.head_dim)
        self.o_proj = nn.Linear(n_heads * self.head_dim, d_model)
    
    def forward(self, x, mask, kv_cache):
        B, T, D = x.shape
        
        # Project
        q = self.q_proj(x).view(B, T, self.n_heads, self.head_dim)
        k = self.k_proj(x).view(B, T, self.n_kv_heads, self.head_dim)
        v = self.v_proj(x).view(B, T, self.n_kv_heads, self.head_dim)
        
        # Apply RoPE
        q, k = apply_rope(q, k, positions)
        
        # Update KV cache
        if kv_cache is not None:
            k = torch.cat([kv_cache.k, k], dim=1)
            v = torch.cat([kv_cache.v, v], dim=1)
            kv_cache.update(k, v)
        
        # Repeat KV heads for GQA
        k = k.repeat_interleave(self.n_heads // self.n_kv_heads, dim=2)
        v = v.repeat_interleave(self.n_heads // self.n_kv_heads, dim=2)
        
        # Attention (using Flash Attention in practice)
        attn_out = flash_attention(q, k, v, mask)
        
        # Output projection
        out = self.o_proj(attn_out.view(B, T, -1))
        return out
```

<a id="feed-forward-network"></a>
### Feed-Forward Network

```python
class SwiGLUFFN(nn.Module):
    def __init__(self, d_model, d_ff):
        super().__init__()
        # SwiGLU has 3 projections instead of 2
        self.gate_proj = nn.Linear(d_model, d_ff, bias=False)
        self.up_proj = nn.Linear(d_model, d_ff, bias=False)
        self.down_proj = nn.Linear(d_ff, d_model, bias=False)
    
    def forward(self, x):
        gate = F.silu(self.gate_proj(x))  # SiLU = Swish
        up = self.up_proj(x)
        return self.down_proj(gate * up)
```

**FFN hidden dimension** 在使用 SwiGLU 時通常是模型維度的 2.7 倍（標準 GELU FFN 則常見為 4 倍）。

<a id="rmsnorm"></a>
### RMSNorm

```python
class RMSNorm(nn.Module):
    def __init__(self, d_model, eps=1e-6):
        super().__init__()
        self.weight = nn.Parameter(torch.ones(d_model))
        self.eps = eps
    
    def forward(self, x):
        rms = torch.sqrt(torch.mean(x ** 2, dim=-1, keepdim=True) + self.eps)
        return self.weight * (x / rms)
```

它比 LayerNorm 更簡單也更快，因為省略了 mean centering。

---

<a id="output-processing"></a>
## 輸出處理

<a id="final-normalization"></a>
### 最終正規化

在最後一個 transformer block 之後套用 RMSNorm：

```python
hidden_states = self.output_norm(hidden_states)
```

<a id="language-model-head"></a>
### Language Model Head

投影到 vocabulary size：

```python
class LMHead(nn.Module):
    def __init__(self, d_model, vocab_size):
        super().__init__()
        self.linear = nn.Linear(d_model, vocab_size, bias=False)
    
    def forward(self, x):
        return self.linear(x)  # Returns logits
```

<a id="untied-vs-tied-embeddings"></a>
## Untied 與 Tied Embeddings

**標準模式（GPT-3、Llama 2）：**Weight Tying
- 輸出 head 與輸入 embeddings 共用權重。
- **優點**：節省記憶體（vocab_size * hidden_dim）。
- **缺點**：強制輸入與輸出的 latent space 完全相同，未必最佳。

**2025 年前沿模式（Llama 3/4、GPT-5.2）：**Untied Embeddings
- 輸出 head 擁有自己的權重。
- **原因？**：更大的 vocabulary（128k+）讓 embedding table 成為模型的顯著部分。Untie 之後，輸出 head 可專精於「predictive logic」，而輸入 embeddings 專注於「semantic understanding」。
- **系統影響**：參數量增加，但對多語言與程式碼任務的 perplexity 往往更好。

<a id="getting-predictions"></a>
### 取得預測

```python
# During generation
logits = lm_head(hidden_states[:, -1, :])  # Last position only
next_token = sample(logits)

# During training
logits = lm_head(hidden_states)  # All positions
loss = cross_entropy(logits, targets)
```

---

<a id="modern-architecture-variations"></a>
## 現代架構變體

<a id="llama-23-architecture"></a>
### Llama 2/3 架構

| 元件 | 實作 |
|-----------|----------------|
| Attention | Grouped Query Attention (GQA) |
| Position | Rotary Position Embedding (RoPE) |
| Normalization | RMSNorm（pre-norm） |
| Activation | SwiGLU |
| Bias | 線性層不使用 bias |

<a id="mistral-architecture"></a>
### Mistral 架構

和 Llama 類似，但另外加入：
- **Sliding Window Attention：**每一層只注意 4K tokens
- 透過堆疊仍可達到有效的 32K+ context

<a id="mixture-of-experts-moe--hybrid-architectures"></a>
### Mixture of Experts (MoE) 與 Hybrid Architectures

2025 年的最先進模型經常使用 **Hybrid MoE/Dense** blocks：
- **Periodic Dense Layers：**每隔幾個 MoE layer，就加入一個 dense layer，確保「全域」知識能在所有 experts 間共享。
- **Expert Parallelism：**把不同 experts 分散到不同 GPU。這使得 **跨節點頻寬**（NVLink/InfiniBand）成為主要的架構瓶頸。

<a id="multi-head-latent-attention-mla-integration"></a>
### Multi-head Latent Attention（MLA）整合

[DeepSeek-V3](file:///Users/om/play/ai-system-design-guide/01-foundations/03-attention-mechanisms.md#multi-head-latent-attention-mla) 與其他同級的 2025 架構，會以低秩 latent compression 取代標準的 Q/K/V projection。
- **架構轉變：**「KV Cache」現在是壓縮後的 latent representation，改變了整個 transformer block 的記憶體／計算比例。

<a id="comparison-of-choices"></a>
### 選項比較

| 選項 | 舊方法 | 現代方法 | 好處 |
|--------|--------------|-----------------|---------|
| Norm | Post-LN | Pre-LN / RMSNorm | 訓練穩定、速度快 |
| Position | Sinusoidal/Learned | RoPE | 更好的外插能力 |
| Activation | GELU | SwiGLU | 品質更好（benchmark 約 +1%） |
| Attention | MHA | GQA | KV cache 小 8 倍 |
| Bias | 有 bias | 無 bias | 參數更少、品質相近 |

---

<a id="scaling-properties"></a>
## 擴展特性

<a id="parameter-counts"></a>
### 參數量

| 元件 | 參數 |
|-----------|------------|
| Token embedding | vocab_size * d_model |
| 每層 Q/K/V | 3 * d_model * d_model（MHA） |
| 每層 O projection | d_model * d_model |
| 每層 FFN | 3 * d_model * d_ff（SwiGLU） |
| LM head | d_model * vocab_size（通常 tied） |

**對 decoder-only 的近似公式：**
```
Total ≈ 12 * n_layers * d_model^2 (for d_ff = 4 * d_model, MHA)
```

<a id="compute-requirements"></a>
### 計算需求

**Training：**每個 token 的 FLOPs ≈ 6 * parameters（forward + backward）

**Inference：**每個 token 的 FLOPs ≈ 2 * parameters（只有 forward）

<a id="scaling-laws"></a>
### Scaling Laws

Chinchilla scaling law 建議的最佳分配是：

```
D (data tokens) ≈ 20 * N (parameters)
```

對 70B 模型而言，若要達到計算最優，應訓練約 1.4T tokens。

**但：**許多現代模型相對於 Chinchilla 會做更多訓練，以換取更高的 inference 效率。Llama 訓練時使用了 2T+ tokens。

---

<a id="architecture-comparison-table"></a>
## 架構比較表

| 模型 | 參數 | 層數 | d_model | heads | KV heads | FFN | Context |
|-------|--------|--------|---------|-------|----------|-----|---------|
| GPT-3 | 175B | 96 | 12288 | 96 | 96 | GELU | 2K |
| Llama 2 70B | 70B | 80 | 8192 | 64 | 8 | SwiGLU | 4K |
| Llama 3 405B| 405B | 126 | 16384 | 128 | 16 | SwiGLU | 128K |
| DeepSeek V3 | 671B | 128 | 7168 | 128 | MLA | MoE | 128K |
| Llama 4 (spec)| 1T+ | 140+ | 18432 | 192 | 24 | MoE/H | 1M+ |

*Mistral 使用 sliding window attention 來達成有效長 context。

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-walk-me-through-the-forward-pass-of-a-transformer"></a>
### 問：帶我走過一次 transformer 的 forward pass。

**強答範例：**
對於會生成文字的 decoder-only 模型：

1. **Tokenization：**把輸入文字轉成 token IDs

2. **Embedding：**從 embedding table 查出 token embeddings

3. **對每一層 transformer：**
   - 對輸入套用 RMSNorm
   - 計算 Q、K、V projections
   - 對 Q 與 K 套用 RoPE 以加入位置資訊
   - 在生成時：把新的 K、V 追加到 KV cache
   - 計算 attention（帶遮罩，因此每個位置只能看到前面）
   - 投影 attention 輸出並加上 residual
   - 套用 RMSNorm
   - 通過 SwiGLU feed-forward network
   - 加上 residual

4. **輸出正規化：**套用最後的 RMSNorm

5. **LM head：**投影到 vocabulary size，得到 logits

6. **Sample：**用 temperature/top-p 從 logits 選出下一個 token

在生成過程中，會對每個新 token 重複步驟 3-6，並重用先前位置的 KV cache。

<a id="q-what-is-the-difference-between-pre-norm-and-post-norm"></a>
### 問：pre-norm 與 post-norm 的差異是什麼？

**強答範例：**
差異在於 layer normalization 相對於 sublayer（attention、FFN）是套用在前還是後：

**Post-norm（原始 transformer）：**
```
x = LayerNorm(x + Sublayer(x))
```
在加上 residual 之後再正規化。

**Pre-norm（現代 transformer）：**
```
x = x + Sublayer(LayerNorm(x))
```
先正規化，再進入 sublayer。

pre-norm 之所以較受青睞，是因為：
1. Gradient 能更直接地穿過 residual connection
2. 訓練更穩定，特別是深層模型
3. 對初始化與 learning rate 較不敏感
4. 不太需要 learning rate warmup

代價是在某些 benchmark 上最終表現可能稍低，但對大型模型來說，訓練穩定性更值得。

<a id="q-explain-gqa-and-why-it-matters-for-serving"></a>
### 問：請解釋 GQA，以及它為何對 serving 很重要。

**強答範例：**
Grouped Query Attention（GQA）會讓多個 Query heads 共享 Key 與 Value heads。

標準 Multi-Head Attention：64 個 query heads、64 個 KV heads（1:1）
GQA：64 個 query heads、8 個 KV heads（8:1）

實作方式：每個 KV head 會透過重複，被 8 個 query heads 共用。

**它的重要性在於：**
Generation 過程中，KV cache 需要儲存所有位置的 K 與 V。以 Llama 70B 的 8K context 為例：
- MHA：2.6 MB/token * 8K = 每個 request 21 GB
- GQA（8:1）：約 2.6 GB / request

8 倍縮減帶來：
- 更大的 batch size（更多並發使用者）
- 更長的 context
- 更低的 GPU 記憶體需求

品質影響很小。研究顯示 GQA 可達到 MHA 99% 以上的品質。

<a id="q-what-changed-between-gpt-2-and-llama-2"></a>
### 問：GPT-2 到 Llama 2 之間改變了什麼？

**強答範例：**
主要的架構改進有：

| 元件 | GPT-2 | Llama 2 |
|-----------|-------|---------|
| Norm | Post-LayerNorm | Pre-RMSNorm |
| Position | Learned absolute | RoPE（rotary） |
| Activation | GELU | SwiGLU |
| Attention | MHA | GQA（70B 版） |
| Bias | 有 | 移除 |

影響：
- RMSNorm：更快，而且同樣有效
- RoPE：長度外插能力更好
- SwiGLU：約 1% 品質提升
- GQA：serving 時 KV cache 小 8 倍
- 無 bias：參數更少，且沒有品質損失

這些變更讓更大的模型能更穩定地訓練，也能更有效率地提供服務。

---

<a id="references"></a>
## 參考資料

- Vaswani et al. "Attention Is All You Need" (2017)
- Touvron et al. "Llama: Open and Efficient Foundation Language Models" (2023)
- Touvron et al. "Llama 2: Open Foundation and Fine-Tuned Chat Models" (2023)
- Zhang and Sennrich. "Root Mean Square Layer Normalization" (2019)
- Shazeer. "GLU Variants Improve Transformer" (2020)
- Su et al. "RoFormer: Enhanced Transformer with Rotary Position Embedding" (2021)
- Jiang et al. "Mistral 7B" (2023)

---

*上一章：[注意力機制](03-attention-mechanisms.md) | 下一章：[Embeddings 與向量空間](05-embeddings-and-vector-spaces.md)*

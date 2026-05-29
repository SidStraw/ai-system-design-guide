<a id="kv-cache-and-context-caching"></a>
# KV Cache 與 Context Caching

KV Cache 是長上下文 AI 系統中最主要的記憶體消耗來源。能否有效管理這個快取，決定了系統是能擴展到 2M tokens，還是在 10k 就崩潰。

<a id="table-of-contents"></a>
## 目錄

- [KV Cache 問題](#kv-cache-problem)
- [GQA：Grouped Query Attention](#gqa)
- [Context Caching（Self-hosted）](#context-caching-self-hosted)
- [API 層級 Context Caching（Prompt Caching）](#api-prompt-caching)
- [RAD-O：Retrieval Augmented Decoding](#rad-o)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-kv-cache-problem"></a>
## KV Cache 問題

在生成期間，模型需要保留所有先前 tokens 的 Key (K) 與 Value (V) tensors。將它們存入記憶體的代價很高。

**VRAM 計算（Llama 4 70B）：**
- **Tokens**：128,000
- **Precision**：BF16（2 bytes/param）
- **Memory**：`2 (KV) * layers (80) * context (128k) * heads (8) * head_dim (128) * 2 bytes`
- **Total**：在 128k context 下，**每位使用者約需 ~42 GB**。

---

<a id="gqa-grouped-query-attention"></a>
## GQA：Grouped Query Attention

GQA 是目前在不犧牲效能前提下降低 KV Cache 大小的標準做法。

| 方法 | 比例 | KV Cache 降幅 | 品質損失 |
|--------|-------|-------------------|--------------|
| **Multi-Head (MHA)** | 1:1 | 1x（基準） | 0% |
| **Grouped Query (GQA)** | 8:1 | **8x** | < 0.2% |
| **Multi-Query (MQA)** | All:1 | 64x-128x | 2-3% |

**細節**：GQA 讓多個「推理」head 共用相同的 KV「記憶體」，大幅降低 Decode 階段所需的記憶體頻寬。

---

<a id="context-caching-self-hosted"></a>
## Context Caching（Self-hosted）

正式系統會對具有共同前綴的 prompts 使用 **Shared KV Caches**（例如 1,000 位使用者共用一份 100 頁知識庫）。

<a id="disk-vs-vram-caching"></a>
### Disk vs. VRAM Caching
- **VRAM Cache**：即時存取，但容量嚴格受限。
- **Disk/SSD Cache**：存取較慢，但容量幾乎無上限。像 **SGLang** 這類框架會使用分層系統：`Most Recent (VRAM) -> Frequent (HBM) -> Occasional (SSD)`。

---

<a id="api-level-context-caching-prompt-caching"></a>
## API 層級 Context Caching（Prompt Caching）

主要供應商（OpenAI、Anthropic、Google、DeepSeek）現在都提供 **Prompt Caching** 折扣。

| Provider | 功能名稱 | 定價（快取輸入） | 最適用場景 |
|----------|--------------|------------------------|----------|
| **Anthropic** | Context Caching | 9 折折扣（Sonnet 4.6 cached: $0.30/1M） | 長 system prompts、tool schemas |
| **OpenAI** | Prompt Caching | 快取輸入約 5 折（GPT-5.5 cached: ~$2.50/1M） | 多輪聊天 |
| **Google** | Context Caching | Cache reads $0.20/1M（Gemini 3.1 Pro 於 200K 以下）；另有每小時儲存費 | 長篇共享語料 |
| **DeepSeek** | Context Caching | **$0.003625/M (V4 Pro) / $0.0028/M (V4 Flash)** | 大型 codebase RAG；市面最便宜快取層 |

**Break-even 細節**：如果快取前綴的重用率超過 **1.1x 到 1.5x**，使用 caching 就比原始 tokens 更便宜。Anthropic 對 cache writes 會加收 25%，所以短前綴的 break-even 會更高（3-5x 重用）。DeepSeek 在 2026 年 4 月 26 日把 cache-hit 價格降到上市時的 1/10。對快取密集型工作負載而言，V4 Flash 現在每個快取 token 的成本大約比 GPT-5.5 低 30-50 倍。

---

<a id="rad-o-retrieval-augmented-decoding"></a>
## RAD-O：Retrieval Augmented Decoding

RAD-O 是一種 context-caching 技術，模型會把長文件的 KV cache **壓縮**成「Latent tokens」。
- **How**：它不是儲存 1M tokens 的完整 KV 向量，而是儲存一個小 10 倍的壓縮表示。
- **Impact**：讓原本只能支援 200k 的硬體，也能跑到 2M+ token 上下文。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-how-does-pagedattention-help-with-kv-cache-management-simplified"></a>
### Q: PagedAttention 如何幫助管理 KV Cache？（簡化版）

**強答：**
標準 KV cache 需要連續記憶體配置（一大塊 RAM）。這會導致**外部碎片化**（有記憶體，但零碎到無法用）。PagedAttention（vLLM 使用）會把 KV cache 切成小而固定大小的「pages」（類似作業系統虛擬記憶體）。這讓 cache 不必連續，因此我們可以在需要時精準配置記憶體，並在不同請求有相同前綴時共享 pages。這通常能把記憶體效率從 60% 提高到 96% 以上。

<a id="q-why-is-context-caching-better-than-rag-for-a-50k-token-document"></a>
### Q: 為什麼對 50k token 文件來說，Context Caching 會比 RAG 更好？

**強答：**
在便宜的 context caching（DeepSeek、Gemini、Anthropic）條件下，RAG 對中型文件來說常常是「殺雞用牛刀」。
1. **Recall**：Context caching 提供 100% recall（整份文件都在視窗內），而 RAG 取決於檢索準確率。
2. **Coherence**：模型可以看到整份文件內的交叉引用。
3. **Economics**：在 50k tokens 規模下，快取輸入成本通常低於維護向量資料庫與檢索管線的複雜度。

---

<a id="references"></a>
## 參考資料
- Kwon et al. "Efficient Memory Management with PagedAttention" (2023)
- Anthropic. "Prompt Caching Documentation" (2024)
- DeepSeek. "Context Caching Technical Report" (2025)

---

*下一篇：[Speculative Decoding](03-speculative-decoding.md)*

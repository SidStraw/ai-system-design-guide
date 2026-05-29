<a id="pagedattention"></a>
# PagedAttention

PagedAttention 是高吞吐量 serving engine（vLLM、SGLang、TensorRT-LLM）背後的核心演算法。它解決了過去限制 LLM 擴展性的「Memory Fragmentation」問題。

<a id="table-of-contents"></a>
## 目錄

- [連續記憶體問題](#contiguous-memory)
- [PagedAttention 如何運作](#how-it-works)
- [管理虛擬記憶體（Block Manager）](#block-manager)
- [KV Cache 共享（Copy-on-Write）](#sharing)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-contiguous-memory-problem"></a>
## 連續記憶體問題

標準深度學習框架會以大型、連續區塊來配置記憶體。
對一個 LLM 請求來說，你可能會為 `max_sequence_length` 8192 tokens 預先保留記憶體。

**浪費之處：**
1. **Internal Fragmentation**：如果使用者只生成 10 個 tokens，保留區塊的 99.9% 都被浪費。
2. **External Fragmentation**：記憶體被切成許多小縫隙，即使總可用記憶體很多，也不足以容納新的「大區塊」。

---

<a id="how-pagedattention-works-vllm"></a>
## PagedAttention 如何運作（vLLM）

PagedAttention 的靈感來自作業系統中的 Virtual Memory。

1. **Tokens to Blocks**：一個請求的 KV cache 會被拆成小而固定大小的 **Blocks**（例如每塊 16 tokens）。
2. **Logical vs. Physical**：模型以為自己在關注一段連續序列（Logical memory），但實際上這些 blocks 分散在 VRAM 各處（Physical memory）。
3. **Lookup Table**：一份 **Block Table** 會把邏輯索引映射到實體位址。

**主要效益**：記憶體浪費可從約 60-80% 降到**低於 4%**。

---

<a id="managing-virtual-memory-block-manager"></a>
## 管理虛擬記憶體（Block Manager）

Serving frameworks（vLLM、SGLang）就像 GPU 的「迷你作業系統」。

- **Allocation**：新請求開始時，Block Manager 會分配一組空的實體 blocks。
- **Eviction**：若 VRAM 已滿，管理器可把閒置 KV blocks「換頁」到 CPU RAM，需要時再搬回來（Paged Swap）。

---

<a id="kv-cache-sharing-copy-on-write"></a>
## KV Cache 共享（Copy-on-Write）

PagedAttention 讓「共同前綴」的共享變得非常容易。

**情境：**100 位使用者都在和同一份 5,000-token system prompt 對話。
- **Traditional**：這份 5,000-token 的 KV cache 要存 100 次（VRAM 內共 **500k tokens**）。
- **PagedAttention**：透過 Block Table **只存一次**，再讓 100 位使用者都指向同一組實體 blocks。
- **Copy-on-Write**：若某位使用者生成了獨特 token，就只為他建立新 block；共享 blocks 保持不變。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-does-pagedattention-significantly-increase-throughput"></a>
### Q: 為什麼 PagedAttention 能大幅提升 throughput？

**強答：**
PagedAttention 能提升 throughput，是因為它讓系統能容納大得多的 **batch sizes**。由於它消除了內部與外部記憶體碎片化，我們可以把更多請求塞進同一張 GPU 的 VRAM 中。傳統 serving 可能因為必須保留最大長度區塊，只能放下 4 個請求；使用 PagedAttention 後，因為只對實際存在的 tokens 配置記憶體，往往能放下 20-30 個請求。更大的 batch 代表更好的 GPU 利用率，以及更高的總 tokens per second。

<a id="q-explain-the-block-table-in-the-context-of-vllm"></a>
### Q: 請用 vLLM 的脈絡解釋「Block Table」。

**強答：**
Block Table 是一種映射結構，用來銜接模型對連續資料的期待，以及實際分散式記憶體的現實。表中的每個條目對應一個 tokens 的「Logical Block」，並記錄該 block 的 key 與 value tensors 實際儲存在 GPU 哪個記憶體位址。這使框架能以小區塊動態配置與釋放記憶體，從而支援 prefix sharing、有效率的多執行緒等進階能力。

---

<a id="references"></a>
## 參考資料
- Kwon et al. "Efficient Memory Management for Large Language Model Serving with PagedAttention" (SOSP 2023)
- vLLM Documentation. "PagedAttention Logic" (2024)

---

*下一篇：[Serving Infrastructure](06-serving-infrastructure.md)*

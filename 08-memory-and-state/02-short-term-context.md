
<a id="short-term-context-management"></a>
# 短期上下文管理

短期上下文（L1 記憶）是推理發生的高速介面。妥善管理它，已不再只是處理「訊息列表」，而是關於 **KV Cache 最佳化** 與 **動態上下文配置**。

<a id="table-of-contents"></a>
## 目錄

- [上下文生命週期](#the-context-lifecycle)
- [KV Cache 分塊與 PagedAttention](#kv-cache-tiling)
- [Prefix Caching（System Prompt 保留）](#prefix-caching)
- [Sliding Windows 與摘要化](#sliding-windows-vs-summarization)
- [情境式壓縮（選擇性捨棄）](#contextual-compression)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="the-context-lifecycle"></a>
## 上下文生命週期

上下文會經歷三個階段：
1. **輸入（Intake）**：使用者查詢 + 近期歷史 + 系統指令。
2. **處理（Processing）**：GPU 為新 token 計算 KV Cache。
3. **驅逐（Eviction）**：一旦達到上限，移除舊 token 以騰出空間給新 token。

---

<a id="kv-cache-tiling"></a>
## KV Cache 分塊

現代推論引擎（vLLM、TensorRT-LLM）使用 **PagedAttention**。
- **概念**：不是為上下文配置一整塊連續的 GPU 記憶體，而是把記憶體切分為 **Blocks**（頁面）。
- **效率**：可將記憶體碎片降低 **60-80%**，從而在相同硬體上支援顯著更大的 batch size 與更長的 context window。

---

<a id="prefix-caching"></a>
## Prefix Caching

這是任何生產環境 LLM stack 在延遲上的 **聖杯**。
- **問題**：每次 agent 呼叫 LLM 時，都會送出相同的 2,000-token System Prompt + 50 個 Tool Schemas。這會浪費算力。
- **解法**：使用 **Persistent Prefix Caching**。伺服器會將提示中「靜態」部分（prefix）的 KV Cache 保留在記憶體中。
- **結果**：你只需要為訊息中*新的*部分付出算力成本（並等待其計算）。

---

<a id="sliding-windows-vs-summarization"></a>
## Sliding Windows 與摘要化

| 方法 | 機制 | 優點 | 缺點 |
|--------|-----------|-----|-----|
| **Sliding Window** | 精確保留最後 N 個 token。 | 對近期內容保真度高。 | 「Dory」效應（忘記開頭）。 |
| **Summarization** | 將舊輪次壓縮為文字。 | 保留「關鍵事實」。 | 會失去細節／格式。 |
| **Hybrid** | 保留最後 10 輪 + 1 則摘要。 | 兼具兩者優點。 | 複雜度略高。 |

---

<a id="contextual-compression"></a>
## 情境式壓縮

目前的前沿模型支援 **Prompt Hardening**。
- **選擇性捨棄（Selective Dropping）**：自動移除前幾輪中不相關的「Thought」區塊，以節省空間。
- **Token Pruning**：先用較小的模型把一則很長的使用者訊息改寫成短 **50%**、但語意等價的 prompt，再送給「Reasoning」模型。

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-what-is-the-difference-between-model-context-window-and-application-context-window"></a>
### 問：什麼是「Model Context Window」與「Application Context Window」之間的差異？

**理想回答：**
**Model Context Window** 是由架構定義的硬性上限（例如 GPT-4o 的 128K）。**Application Context Window** 則是工程師設定的配置（例如 16K 上限），用來管理 **Latency 與 Cost**。在生產環境中，我們很少在每一輪都用滿整個 model window，因為 attention 的開銷會隨著上下文大小增加，導致生成變慢。我們會保留一個 **Buffer Zone**，為模型的新回應預留空間。

<a id="q-how-does-prefix-caching-change-how-you-design-system-prompts"></a>
### 問：「Prefix Caching」如何改變你設計 System Prompts 的方式？

**理想回答：**
它會迫使我把 **靜態內容放前面**，把 **動態內容放後面**。早期 LLM 的模式常把使用者名稱或日期放在最前面，但那會破壞 prefix cache。我會把「Immutable Rules」與「Tool Schemas」放在開頭，把每輪都會變動的「User Context」放在最後。這能確保前 5,000 個 token 在所有使用者之間都完全一致，從而最大化推論伺服器上的 cache hit。

---

<a id="references"></a>
## 參考資料
- vLLM Team. "PagedAttention: Software-Defined Memory for LLM Serving" (2024/2025)
- NVIDIA. "Optimizing Inference with TensorRT-LLM" (2025)
- Anthropic. "Prompt Caching: Scale while reducing costs" (2024/2025)

---

*下一步：[長期記憶](03-long-term-memory.md)*

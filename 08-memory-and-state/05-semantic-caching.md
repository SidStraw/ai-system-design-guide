
<a id="semantic-caching"></a>
# 語意快取

快取已從精確字串匹配演進為 **語意匹配**。語意快取可藉由重用「等價」查詢的完成結果，將成本降低 **30-70%**，並把延遲從數秒縮短到數毫秒。

<a id="table-of-contents"></a>
## 目錄

- [精確快取 vs. 語意快取](#exact-cache-vs-semantic-cache)
- [語意匹配流程](#the-semantic-matching-pipeline)
- [RedisVL 與 GPTCache](#redisvl-and-gptcache)
- [評估：命中率 vs. 幻覺式漂移](#evaluation-hit-rate-vs-hallucinated-drift)
- [多模態語意快取](#multimodal-semantic-caching)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="exact-cache-vs-semantic-cache"></a>
## 精確快取 vs. 語意快取

| 功能 | 精確快取 (Redis/Memcached) | 語意快取 (RedisVL/Qdrant) |
|---------|-------------------------------|---------------------------------|
| **鍵** | 雜湊後的查詢字串 | 查詢 embedding 向量 |
| **匹配**| 100% 字串完全相同 | Cosine Similarity > Threshold |
| **效率**| 低（輕微拼字錯誤就會破壞快取） | 高（能理解意圖） |
| **風險** | 零 | 語意漂移（Semantic Drift，回傳錯誤答案） |

---

<a id="the-semantic-matching-pipeline"></a>
## 語意匹配流程

1. **產生 embedding（Embed）**：傳入的查詢會被轉換為向量（例如使用 `text-embedding-3-small`）。
2. **搜尋（Search）**：在快取中搜尋最近鄰。
3. **門檻檢查（Threshold Check）**：若 `distance < 0.05`（非常相似），則回傳快取結果。
4. **LLM 驗證（LLM Verification）**：對於高風險查詢，小型的「驗證模型（Verifier Model）」（例如 GPT-5.5-mini、Claude Haiku 4.5）會檢查快取回應是否真的回答了新查詢。
5. **更新（Update）**：如果沒有命中，呼叫 LLM 並將新結果儲存到向量快取中。

---

<a id="redisvl-and-gptcache"></a>
## RedisVL 與 GPTCache

標準堆疊：
- **RedisVL**：直接在 Redis 實例內提供低延遲向量搜尋。
- **混合式快取（Hybrid Caching）**：同時使用 Redis 儲存中繼資料（keys）與向量 payloads。
- **TTL**：語意快取應具有 TTL（Time-To-Live）。常見模式是 **Dynamic TTL**：熱門答案保留更久，而「過時」資訊會被定期淘汰。

---

<a id="multimodal-semantic-caching"></a>
## 多模態語意快取

隨著原生多模態前沿模型（Gemini 3.1 Pro、GPT-5.5、Claude Opus 4.7）的出現，我們現在也會快取 **影像與音訊查詢**。
- **Visual Similarity**：如果先前已處理過語意相似的影像，就快取該影像的描述。
- **Audio Fingerprinting**：為相似的語音指令快取逐字稿。

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-what-is-semantic-drift-in-caching-and-how-do-you-prevent-it"></a>
### 問：在快取中，「語意漂移（Semantic Drift）」是什麼？你要如何防止它？

**優秀回答：**
當相似度門檻設得太寬鬆時（例如 0.8 而不是 0.95），就會發生 Semantic Drift。像是 *「我該如何修理我的車？」* 這類查詢，可能會匹配到 *「我該如何清洗我的車？」* 的快取回應。為了避免這種情況，我們會使用 **多階段驗證**：1) 向量相似度檢查，2) **Entity-Match 檢查**（確保兩個查詢都涉及「Car」以及相同的「Verb」），以及 3) **收緊 Threshold**：對於技術或醫療查詢，我們要求 $>0.98$ 的相似度才會回傳快取結果。

<a id="q-why-is-a-semantic-cache-sometimes-more-expensive-than-a-raw-llm-call-at-low-volume"></a>
### 問：為什麼在低流量時，語意快取有時會比直接呼叫原始 LLM 更昂貴？

**優秀回答：**
因為語意快取本身需要額外的 **Embedding API 呼叫** 和 **Vector Search 查詢**。如果 embedding 模型成本是 $0.02、搜尋耗時 100ms，而你的主要 LLM 呼叫只要 $0.05 且耗時 500ms，那麼相對節省就很有限。語意快取只有在 **高規模**（數百萬次請求）時，當快取命中率足夠高，能抵銷這筆「Embedding Tax」並大幅降低整體延遲時，才會成為顯著優勢。

---

<a id="references"></a>
## 參考資料
- Redis。「RedisVL：Redis Vector Library 的 Python 用戶端」(2025)
- Akiba 等人。「GPTCache：建立語意快取的函式庫」(2024/2025)
- Google Cloud。「生成式 AI 快取模式」(2025)

---

*下一篇：[狀態管理模式](06-state-management-patterns.md)*

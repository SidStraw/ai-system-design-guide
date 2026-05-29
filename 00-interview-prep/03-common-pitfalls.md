<a id="common-pitfalls-in-ai-system-design-interviews"></a>
# AI 系統設計面試中的常見陷阱

本章整理候選人在 AI 系統設計面試中常犯的錯誤、這些錯誤為何會拉低你的評價，以及如何避免它們。

<a id="table-of-contents"></a>
## 目錄

- [架構層面的陷阱](#architecture-pitfalls)
- [技術知識陷阱](#technical-knowledge-pitfalls)
- [溝通陷阱](#communication-pitfalls)
- [面試策略陷阱](#interview-strategy-pitfalls)
- [AI 特有陷阱](#ai-specific-pitfalls)
- [自我檢查清單](#checklists-for-self-review)

---

<a id="architecture-pitfalls"></a>
## 架構層面的陷阱

<a id="pitfall-1-skipping-the-data-pipeline"></a>
### 陷阱 1：跳過 Data Pipeline

**會出什麼問題：**
候選人把 inference path 講得很細，卻幾乎沒提資料是怎麼進入系統的。

**為什麼重要：**
資料品質決定 AI 系統品質。就算 RAG 架構畫得很漂亮，如果文件 ingestion pipeline 產生的是垃圾 chunks，整個系統一樣沒有用。

**面試官會注意到：**
- 完全沒提文件如何被處理
- 假設 embeddings 會憑空出現
- 忽略更新與刪除流程

**較佳做法：**
```
「在討論 retrieval 之前，我先走一遍 data pipeline：

1. Document ingestion：檔案上傳、API integration、crawler
2. Preprocessing：格式轉換、清理、metadata extraction
3. Chunking：根據文件結構採用 [Strategy]
4. Embedding：用 [model] 做 batch processing
5. Indexing：連同 metadata upsert 到 vector database
6. Updates：文件變更時做 incremental re-indexing」
```

---

<a id="pitfall-2-one-size-fits-all-model-selection"></a>
### 陷阱 2：用同一種模型解所有問題

**會出什麼問題：**
候選人說他們會用 GPT-4（或任何單一模型）處理所有事情。

**為什麼重要：**
不同任務有不同需求。拿 frontier model 做分類很浪費；拿小模型做複雜推理則會失敗。

**面試官會注意到：**
- 沒討論成本影響
- 沒考慮延遲需求
- 沒提 model cascade 或 routing

**較佳做法：**
```
「模型選擇會依任務而變：

- Intent classification：Fine-tuned BERT 或 GPT-4o-mini
- 簡單回應：Claude Haiku 或 GPT-4o-mini
- 複雜推理：GPT-4o 或 Claude 3.5 Sonnet
- Code generation：Claude 3.5 Sonnet 或 Codestral

我會實作一個 router，先判斷 query complexity，
再導到合適的模型。這通常可以在幾乎不影響品質的情況下，
把成本降低 60-70%。」
```

---

<a id="pitfall-3-ignoring-the-evaluation-layer"></a>
### 陷阱 3：忽略 Evaluation Layer

**會出什麼問題：**
候選人會說明系統怎麼建，但不會說明怎麼判斷它是否真的有效。

**為什麼重要：**
AI 系統常以很隱晦的方式失效。沒有 evaluation，你就會把壞掉的系統上線，而且永遠不會發現品質退化。

**面試官會注意到：**
- 沒提 test set
- 沒定義品質指標
- 沒有 production 問題的監控

**較佳做法：**
```
「Evaluation 有三層：

1. Offline：每次變更都跑 golden test set
   - Retrieval：Precision@5、Recall@5、MRR
   - Generation：Faithfulness、relevance（RAGAS）
   - End-to-end：答案正確性 vs ground truth

2. Online：在 production 中做抽樣評估
   - 對 5% 的 requests 使用 LLM-as-judge
   - User feedback（thumbs up/down）
   - 任務型查詢的 completion rate

3. Alerting：自動偵測
   - 品質分數低於門檻
   - 延遲超出 SLA
   - 錯誤率飆升」
```

---

<a id="pitfall-4-underestimating-multi-tenancy-complexity"></a>
### 陷阱 4：低估 Multi-Tenancy 的複雜度

**會出什麼問題：**
候選人把 multi-tenant RAG 當成只是多加一個 `tenant_id` 欄位。

**為什麼重要：**
多租戶 AI 系統在資料外洩、隔離，以及公平資源分配上有獨特的失效模式。

**面試官會注意到：**
- Post-retrieval filtering（重大警訊）
- 沒討論 cache isolation
- 沒考慮 noisy neighbor

**較佳做法：**
```
「RAG 的 multi-tenancy 比傳統系統更難：

1. Retrieval isolation：在 retrieval **之前** 就在資料庫層過濾
   錯誤： retrieve(query, top_k=100) 然後再依 tenant 過濾
   正確： retrieve(query, top_k=10, filter={tenant_id: X})

2. Context isolation：絕不能在 LLM context 中混用不同 tenant 的資料

3. Cache isolation：所有 cache keys 都要帶 tenant 範圍
   cache_key = f'{tenant_id}:{query_hash}'

4. Embedding isolation：對安全要求最高的情境，可考慮 tenant-specific embedding spaces

5. Audit：所有操作都記錄 tenant context

我也會定期執行 isolation tests，使用 adversarial queries 來探測是否存在 cross-tenant leakage。」
```

---

<a id="pitfall-5-no-graceful-degradation"></a>
### 陷阱 5：沒有 Graceful Degradation

**會出什麼問題：**
當 LLM provider 掛掉、被 rate-limit 或回傳錯誤時，系統沒有任何 fallback。

**為什麼重要：**
LLM providers 會故障，rate limits 也會撞到。是否有失敗處理，是 production-ready 與 prototype 的分水嶺。

**面試官會注意到：**
- 沒提 fallback
- 沒有 retry strategy
- 依賴單一 provider

**較佳做法：**
```
「可靠性分層：

1. Retry with backoff：暫時性錯誤可重試
   - Exponential backoff with jitter
   - 最多 3 次

2. Fallback providers：主要 provider 失敗時，改用次要 provider
   - OpenAI → Anthropic → local model
   - 抽象化介面以便替換

3. Cached responses：對已知 queries 回傳快取結果
   - 重複問題用 exact match cache
   - 類似問題用 semantic cache

4. Graceful degradation：失敗時保留部分功能
   - Retrieval 失敗 → 回傳直接的 LLM 回應並加上 disclaimer
   - LLM 失敗 → 回傳相關 chunks，但不做 synthesis

5. Circuit breaker：當 provider 狀況不佳時快速失敗
   - 避免連鎖延遲問題」
```

---

<a id="technical-knowledge-pitfalls"></a>
## 技術知識陷阱

<a id="pitfall-6-confusing-embedding-and-generation-models"></a>
### 陷阱 6：混淆 Embedding 與 Generation Models

**會出什麼問題：**
候選人會把 embedding models 當成拿來生成文字，或把 generation 說成 retrieval。

**你需要知道：**
- **Embedding models：** 將文字映射成 vector。用於 search/retrieval。
- **Generation models：** 根據 prompt 產生文字。用於生成回應。

**兩者如何串接：**
RAG 會先用 embedding models 做 retrieval，再把取回的 chunks 傳給 generation model。

---

<a id="pitfall-7-misunderstanding-context-windows"></a>
### 陷阱 7：誤解 Context Window

**會出什麼問題：**
- 以為 128K context 代表有 128K tokens 的有效上下文
- 沒把 system prompt、retrieved chunks 與 conversation history 算進去
- 忽略「lost in the middle」現象

**你需要知道：**
- Context window 是上限，不是目標
- 中間內容的 attention 會衰退
- 實際有效的 context 通常比上限小很多

**更好的表述方式：**
```
「雖然 GPT-4o 支援 128K tokens，但我設計時會以小得多的有效 context 為目標：

- System prompt：約 500 tokens
- Retrieved context：3-5 個 chunks × 500 tokens = 1.5-2.5K
- Conversation history：最近 5 輪 × 300 tokens = 1.5K
- 輸出預留 buffer：約 2K

活躍 context 總量約 7K tokens，遠低於上限。

這樣可以讓模型專注在真正相關的資訊上，
並避免 Liu et al. 所描述的 lost-in-the-middle 問題。」
```

---

<a id="pitfall-8-not-understanding-token-economics"></a>
### 陷阱 8：不了解 Token Economics

**會出什麼問題：**
候選人在談功能時，卻不了解背後的成本影響。

**你需要知道：**
- 計價通常按 token，且 input 與 output 常有不同價格
- 對多數 providers 來說，output tokens 通常是 input tokens 的 2-4 倍價格
- Streaming 不會改變成本

**快速參考（2025 年 12 月，請確認最新價格）：**

| Model | Input/1M | Output/1M |
|-------|----------|-----------|
| GPT-4o | $2.50 | $10 |
| GPT-4o-mini | $0.15 | $0.60 |
| Claude 3.5 Sonnet | $3 | $15 |
| Claude 3.5 Haiku | $0.25 | $1.25 |
| Gemini 1.5 Pro | $1.25 | $5 |

**成本計算範例：**
```
10,000 queries/day
平均：2K input tokens、500 output tokens
Model: GPT-4o

每日成本 = 10K × (2K × $2.50/1M + 500 × $10/1M)
          = 10K × ($0.005 + $0.005)
          = 10K × $0.01
          = $100/day = $3K/month
```

---

<a id="pitfall-9-shallow-understanding-of-rag-components"></a>
### 陷阱 9：對 RAG 元件理解過淺

**會出什麼問題：**
候選人能列出元件（chunking、embedding、retrieval、generation），卻無法解釋每一層內部的取捨。

**對 chunks 應具備的深度：**
- 為什麼要 chunk？（Context limits、retrieval precision）
- Chunk size 有什麼取捨？（較小 = 更精準，較大 = 更多 context）
- Overlap 的目的？（避免邊界處遺失上下文）
- 何時該用 semantic chunking？（結構變化大的複雜文件）

**對 retrieval 應具備的深度：**
- 為什麼用 hybrid search？（dense 擅長語意，sparse 擅長關鍵字）
- 什麼是 reranking？（兩階段：先快速 recall，再精準排序）
- 沒有結果時怎麼辦？（fallback strategies）

---

<a id="pitfall-10-treating-prompts-as-magic"></a>
### 陷阱 10：把 Prompt 當魔法

**會出什麼問題：**
候選人輕描淡寫地說「然後我們 prompt 一下模型……」，卻沒有討論 prompt engineering。

**面試官想看到的是：**
- Prompt 結構（system、context、user）
- 指令是否清楚
- Output format 是否有明確規範
- 是否在適當時機使用 few-shot examples
- 是否防禦 edge cases

**較佳做法：**
```
「Generation prompt 會長這樣：

SYSTEM:
You are a support assistant for [Product]. Answer questions 
using ONLY the provided context. If the context does not 
contain the answer, say 'I don't have information about that.'
Always cite the source document.

CONTEXT:
[Retrieved chunks with source metadata]

USER:
[User's question]

我會明確指定 output format，並在複雜回應結構時使用 few-shot
examples。對這個 use case，我也會加入 negative examples，
示範在何時應該 abstain。」
```

---

<a id="communication-pitfalls"></a>
## 溝通陷阱

<a id="pitfall-11-monologuing-without-interaction"></a>
### 陷阱 11：只顧著長篇大論，沒有互動

**會出什麼問題：**
候選人連續講 10-15 分鐘，完全沒和面試官確認方向。

**為什麼重要：**
面試是對話，不是獨白。一直 monologue 會錯過面試官真正關心什麼的訊號。

**較佳做法：**
每 3-5 分鐘確認一次：
- 「你希望我更深入 retrieval，還是切到 generation？」
- 「在我談細節之前，這個架構方向合理嗎？」
- 「有沒有哪個元件是你希望我特別聚焦的？」

---

<a id="pitfall-12-not-leading-with-structure"></a>
### 陷阱 12：沒有先講結構

**會出什麼問題：**
候選人一開口就直接講，沒有先說明接下來會涵蓋哪些部分。

**為什麼重要：**
面試官心中都有自己的框架。如果你的回答無法對映到他們的預期，你就會顯得很亂。

**較佳做法：**
先給一個 roadmap：
```
「我會把答案分成四個部分：
1. 高層架構
2. 深入 RAG pipeline
3. Scaling 與 reliability
4. Evaluation 方法

先從高層架構開始……」
```

---

<a id="pitfall-13-technical-jargon-without-explanation"></a>
### 陷阱 13：丟技術術語卻不解釋

**會出什麼問題：**
候選人丟出像「PagedAttention」或「GQA」這種詞，卻沒有解釋。

**為什麼重要：**
如果面試官不知道這個詞，你會看起來像是在 name-dropping；如果他們知道，就可能追問你答不出來的細節。

**較佳做法：**
在引入術語時簡短說明：
```
「我會使用 vLLM，它實作了 PagedAttention。
這會用類似 virtual memory 的方式管理 KV cache，降低
fragmentation，並提高吞吐量。」
```

---

<a id="pitfall-14-defending-wrong-answers"></a>
### 陷阱 14：硬拗錯誤答案

**會出什麼問題：**
當面試官已暗示某個方向不對時，候選人不是重新思考，而是繼續硬拗。

**為什麼重要：**
固執是紅旗；願意接受指導、可被 coaching 很有價值。

**較佳做法：**
```
面試官：「那如果遇到……的情況呢？」
你：「這點提醒得很好，我剛剛沒有想到 [X]。
讓我調整一下我的做法……」
```

---

<a id="interview-strategy-pitfalls"></a>
## 面試策略陷阱

<a id="pitfall-15-solving-a-different-problem"></a>
### 陷阱 15：解了另一個問題

**會出什麼問題：**
候選人對某個技術太興奮，最後設計的是那個技術，而不是題目真正要求的系統。

**範例：**
明明被要求設計一個簡單的 Q&A 系統，候選人卻設計出帶 autonomous research capabilities 的複雜 multi-agent system。

**較佳做法：**
先依需求設計，再補充延伸：
```
「這個設計能滿足核心需求。如果未來要擴充成處理更複雜的 multi-step queries，
我們可以再加上 agent layer，但我不會一開始就從那裡出發。」
```

---

<a id="pitfall-16-not-managing-time"></a>
### 陷阱 16：不管理時間

**會出什麼問題：**
候選人花 20 分鐘講架構，結果沒有時間談 evaluation、reliability 或 scaling。

**較佳做法：**
明確分配時間：
- 澄清：3-5 分鐘
- 高層設計：5-7 分鐘
- Deep dives：10-15 分鐘
- Evaluation / reliability：5-7 分鐘
- 問題 / 收尾：3-5 分鐘

隨時看時間並調整。

---

<a id="pitfall-17-not-drawing"></a>
### 陷阱 17：不畫圖

**會出什麼問題：**
候選人口頭描述架構，卻沒有畫圖。

**為什麼重要：**
視覺化溝通更清楚，也能展現你可以和 stakeholders 溝通。

**較佳做法：**
一邊解釋一邊畫 boxes and arrows。標示要清楚，並在後續討論中持續把圖當參考。

---

<a id="ai-specific-pitfalls"></a>
## AI 特有陷阱

<a id="pitfall-18-treating-ai-components-as-black-boxes"></a>
### 陷阱 18：把 AI 元件當黑盒子

**會出什麼問題：**
候選人把「呼叫 LLM」當成單一步驟，卻不了解內部發生了什麼事。

**Senior 職級的期待：**
- 理解 prefill 與 decode phases
- 知道什麼會影響延遲（TTFT vs TPS）
- 理解 KV cache 的影響
- 知道 batching effects

---

<a id="pitfall-19-ignoring-hallucination-risk"></a>
### 陷阱 19：忽略 Hallucination 風險

**會出什麼問題：**
候選人設計的系統會盲目信任 LLM 輸出。

**為什麼重要：**
Hallucinations 是 LLM 的固有特性。Production 系統必須有對應處理。

**較佳做法：**
```
「Hallucination 緩解需要多層設計：

1. Retrieval grounding：只根據 context 作答
2. Citation enforcement：每個主張都要附來源
3. Abstention：適當時讓模型說『我不知道』
4. Output validation：檢查是否有不可能的主張
5. Confidence display：讓使用者知道何時該自行驗證」
```

---

<a id="pitfall-20-security-as-an-afterthought"></a>
### 陷阱 20：把安全性當成事後補救

**會出什麼問題：**
安全考量如果有提，通常也是最後才補。

**為什麼重要：**
AI 系統有新的攻擊面（prompt injection、data leakage）。安全性需要從設計一開始就納入。

**較佳做法：**
把安全性編進整體設計：
```
「在 retrieval layer，我會在資料庫層使用 metadata filtering，
以確保 tenant isolation。System prompt 會利用 instruction hierarchy
來抵抗 injection。輸出在到達使用者前，會先通過 content filter。」
```

---

<a id="checklists-for-self-review"></a>
## 自我檢查清單

<a id="before-the-interview"></a>
### 面試前

- [ ] 已複習 RAG 架構模式
- [ ] 知道目前模型價格的大致區間
- [ ] 能解釋 chunking strategies
- [ ] 理解 embedding 與 generation 的差異
- [ ] 知道常見 evaluation metrics
- [ ] 至少能討論一種 vector database
- [ ] 理解 multi-tenancy 挑戰
- [ ] 能討論 prompt engineering 技巧

<a id="during-the-interview"></a>
### 面試中

- [ ] 有先問澄清問題
- [ ] 有說明優先順序與取捨
- [ ] 有畫圖
- [ ] 有提到 evaluation 方法
- [ ] 有討論失效模式
- [ ] 有處理 security / isolation
- [ ] 有和面試官確認方向
- [ ] 有管理各段落時間

<a id="after-each-section"></a>
### 每一段之後

- [ ] 我有解釋「為什麼」，而不只是「做什麼」嗎？
- [ ] 我有提到取捨嗎？
- [ ] 我有在適當時使用具體數字嗎？
- [ ] 我有避免不必要的複雜度嗎？

---

*另請參考：[題庫](01-question-bank.md) | [答題框架](02-answer-frameworks.md) | [白板練習](04-whiteboard-exercises.md)*

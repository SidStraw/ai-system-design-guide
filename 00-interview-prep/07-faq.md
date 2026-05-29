<a id="frequently-asked-questions-ai-engineering-rag-and-agents"></a>
# 常見問題：AI Engineering、RAG 與 Agents

以簡短直接的方式回答大家最常問的現代 AI 系統設計問題。每個答案都會指出可深入閱讀該主題的章節。

<a id="table-of-contents"></a>
## 目錄

- [General：AI Engineering 角色](#general)
- [RAG 與 Retrieval](#rag)
- [Agents 與 Tool Use](#agents)
- [Models 與選型](#models)
- [Evaluation 與 Observability](#evaluation)
- [Inference 與成本](#inference)
- [Memory 與狀態](#memory)
- [Security 與 Safety](#security)

---

<a id="general"></a>
## General

<a id="what-is-an-ai-engineer"></a>
### 什麼是 AI engineer？

AI engineer 會在大型語言模型之上建構 production 系統。這個角色位在傳統軟體工程與機器學習研究之間：較少做模型訓練，更多是在既有模型周圍做系統設計。日常工作涵蓋 prompt 與 context engineering、retrieval pipeline、agent loop、evaluation harness，以及維持整體系統在線的基礎設施。參見 [AI Job Market Trends](06-job-market-trends-2026.md)。

<a id="what-is-the-difference-between-an-ai-engineer-and-an-ml-engineer"></a>
### AI engineer 與 ML engineer 有什麼差別？

ML engineer 負責訓練、fine-tune 並交付模型。AI engineer 則把現有模型（通常透過 API）組裝成產品。ML engineer 的時間主要花在 dataset 與 training loop；AI engineer 的時間主要花在 prompts、RAG、agents、evals 與 latency。對於同時做兩者的大公司，界線會變得模糊。參見 [Transition Guide](../TRANSITION_GUIDE.md)。

<a id="how-do-i-become-an-ai-engineer"></a>
### 我如何成為 AI engineer？

如果你已經能撰寫 production code，差距其實不大：學會 LLM 的行為方式、學會 RAG 與 agent patterns、學會 evaluation，並掌握一套 inference stack。[Transition Guide](../TRANSITION_GUIDE.md) 會把你目前的角色（backend、frontend、QA、PM、data、DevOps）對應到適合的 AI 職位，以及需要補齊的具體技能。

<a id="what-programming-language-should-i-learn-for-ai-engineering"></a>
### 做 AI engineering 應該學什麼程式語言？

Python 是 AI 工作的預設語言。TypeScript 是最常見的第二語言，因為 frontend 與 edge agent stack 常落在那裡。C# 與 Go 也會出現在 enterprise infrastructure 類職位中。大多數 production AI 程式碼看起來都像一般應用程式：HTTP client、queue consumer、database call，再加上對 model provider 的呼叫。

<a id="is-ai-engineering-a-good-career"></a>
### AI engineering 是好的職涯選擇嗎？

需求仍然強勁，薪資與資深軟體工程相當，在 frontier labs 通常更高。風險在於技術堆疊變化很快：去年還重要的 framework，今天可能已經 deprecated。能跨多次版本迭代持續累積的能力，是 evaluation、system design 與 grounded debugging。參見 [Job Market Trends](06-job-market-trends-2026.md)。

---

<a id="rag"></a>
## RAG

<a id="what-is-rag"></a>
### 什麼是 RAG？

Retrieval-Augmented Generation 是一種模式：在查詢時抓取外部 context（文件、資料列、程式碼、圖片），放進 LLM 的 prompt，讓模型能根據內容回答，而不是憑訓練資料 hallucinate。對於任何需要知道訓練截止點之外資訊的 LLM，這是最常見的 production pattern。參見 [RAG Fundamentals](../06-retrieval-systems/01-rag-fundamentals.md)。

<a id="how-does-rag-work"></a>
### RAG 如何運作？

使用者查詢會先被轉成搜尋請求，系統再從 knowledge store（vector DB、keyword index、graph，或其組合）中找出最相關的 chunks，對結果 rerank，最後把前幾名結果當作 context 傳給 LLM 生成答案。兩個失敗點分別是 retrieval（正確 chunk 沒有被取回）與 generation（模型忽略或誤用該 chunk）。大多數 RAG 失敗其實都是 retrieval failure。

<a id="is-rag-dead-because-of-long-context-windows"></a>
### 長 context window 出現後，RAG 已經過時了嗎？

沒有。即使 Claude Opus 4.7、Claude Sonnet 4.6、GPT-5.5 與 Gemini 3.1 Pro 都具備 1M-2M token 的 context window，RAG 在成本、延遲、新鮮度與語料規模上仍然更有優勢。企業資料集（SharePoint、log archives、code monorepos）會超過任何 context window。RAG 就像篩網，負責先找出值得放進高價值視窗的那 0.01% 資料。參見 [RAG vs Long Context](../06-retrieval-systems/14-production-rag-at-scale.md)。

<a id="what-is-the-difference-between-rag-and-fine-tuning"></a>
### RAG 與 fine-tuning 有什麼差別？

RAG 透過 context 在查詢時注入知識；fine-tuning 則把行為烘焙進權重中。經驗法則是：**RAG for facts，fine-tuning for form**。當知識會變動、需要 citations，或資料必須留在模型外部時，用 RAG；當你要穩定語氣、嚴格輸出格式，或降低重複任務的 latency 時，用 fine-tuning。參見 [Fine-Tuning Strategies](../03-training-and-adaptation/02-fine-tuning-strategies.md)。

<a id="what-is-the-best-vector-database"></a>
### 最好的 vector database 是哪個？

沒有單一最佳答案。Pinecone 在受管規模與 SLA 上表現最好。Qdrant 在 open-source 速度上領先（約 10M vectors 時 p99 約 12ms）。Weaviate 擁有最強的原生 hybrid（BM25 + dense + metadata 單次查詢整合）。一旦需要超過 50M vectors 的分散式規模，Milvus 往往是選擇。如果你本來就用 Postgres，且索引低於 10M vectors，pgvector 通常是正解。參見 [Vector Databases](../06-retrieval-systems/04-vector-databases.md)。

<a id="what-is-contextual-retrieval"></a>
### 什麼是 contextual retrieval？

Contextual retrieval 是 Anthropic 提出的一種技術：在 embedding 與索引前，先為每個 chunk 加上一段由 LLM 生成的短摘要，讓 chunk 自帶它在原文件中的位置語境。Anthropic 報告指出，搭配 hybrid search 時可讓 retrieval failure 降低 49%，若再加上 reranker 則可降低 67%。參見 [Contextual Retrieval](../06-retrieval-systems/10-contextual-retrieval.md)。

<a id="what-is-hybrid-search"></a>
### 什麼是 hybrid search？

Hybrid search 會把 sparse keyword retrieval（通常是 BM25）與 dense vector retrieval 結合，再把兩份排序結果融合成單一排名，通常使用 Reciprocal Rank Fusion。Sparse 端能抓到精確 token（產品代碼、函式名稱、罕見名詞）；dense 端能抓到同義詞與意圖。每一個現代 vector DB 幾乎都原生支援 hybrid。參見 [Hybrid Search](../06-retrieval-systems/05-hybrid-search.md)。

<a id="what-is-graphrag"></a>
### 什麼是 GraphRAG？

GraphRAG 會從 corpus 中抽取 entities 與 relationships，建立 knowledge graph，並透過 traversal 查詢，而不是只靠（或主要靠）vector similarity。對於**彙整型問題**（例如「總結這 50 份合約中的所有法律風險」），當向量式 RAG 只會回傳相關但彼此斷裂的 chunks 時，這種模式特別合適。Microsoft 的 LazyGraphRAG 將昂貴的 community summarization 延後到查詢時才做，從而降低 ingestion 成本。參見 [GraphRAG](../06-retrieval-systems/07-graph-rag.md)。

<a id="what-is-the-best-chunk-size-for-rag"></a>
### RAG 最佳 chunk size 是多少？

沒有放諸四海皆準的答案。對一般文字內容來說，300-500 token、50-token overlap 是合理的預設值。程式碼與結構化資料通常需要更大的 chunks（1000-2000 tokens）。真正更大的收益通常來自 **structure-aware chunking**（依標題、段落、code block 切分）、**contextual chunking**（前置摘要），以及 **hierarchical chunking**（索引小 chunk，但回傳父層 context）。參見 [Chunking Strategies](../06-retrieval-systems/02-chunking-strategies.md)。

---

<a id="agents"></a>
## Agents

<a id="what-is-an-ai-agent"></a>
### 什麼是 AI agent？

AI agent 是一種系統：LLM 決定下一步要做什麼、執行某個工具、觀察結果，再做下一次決策，如此循環。最簡單的 agent 就是 ReAct 模式：Thought → Action → Observation → repeat。現代 agents（Claude Opus 4.7、GPT-5.5 reasoning、DeepSeek-R2）則把 reasoning 步驟直接內建在模型中。參見 [Agent Fundamentals](../07-agentic-systems/01-agent-fundamentals.md)。

<a id="what-is-the-difference-between-an-agent-and-a-chatbot"></a>
### agent 與 chatbot 有什麼差別？

Chatbot 只是回應訊息；agent 會**在世界中採取行動**：執行程式碼、呼叫 API、讀檔、發送訊息、預約行程。這個差別很重要，因為行動很難回滾，所以 agent 需要完全不同層級的 guardrails、sandboxing 與 human-in-the-loop 模式。參見 [Agentic Security](../07-agentic-systems/09-agentic-security-and-sandboxing.md)。

<a id="what-is-mcp-model-context-protocol"></a>
### 什麼是 MCP（Model Context Protocol）？

MCP 是一種開放協定，讓 LLM 應用程式可以透過標準介面連接工具與資料來源。Anthropic 在 2024 年 11 月推出它；治理權於 2025 年 12 月轉移到 Linux Foundation 的 Agentic AI Foundation。採用已相當普遍：Anthropic、OpenAI、Google、Microsoft、AWS 都支援。截至 2026 年 5 月，已有超過 2,300 個公開 MCP servers。參見 [Tool Use and MCP](../07-agentic-systems/03-tool-use-and-mcp.md)。

<a id="what-is-the-best-agent-framework"></a>
### 最好的 agent framework 是哪個？

三大主流已能覆蓋多數 production 需求。**LangGraph**（圖式架構，於 2026 年初在 stars 數超越 CrewAI）是處理有狀態 multi-agent control flow 與 checkpointing 的預設選擇。**CrewAI**（現為 v1.13，使用於 60%+ 的 Fortune 500）適合角色導向的商業自動化。**Microsoft Agent Framework**（2026 年 2 月 RC 1.0，Q2 2026 GA）則是 AutoGen 與 Semantic Kernel 整合後，面向 enterprise .NET 與 Python 的後繼框架。參見 [Framework Selection Guide](../09-frameworks-and-tools/08-framework-selection-guide.md)。

<a id="what-is-agentic-rag"></a>
### 什麼是 agentic RAG？

Agentic RAG 以迴圈取代線性的 retrieve-then-generate pipeline：agent 會自己決定該抓什麼、判斷結果是否夠好，不夠就重新查詢。常見模式包括 Self-RAG（模型輸出 reflection tokens）、Corrective RAG（獨立 grader）、Adaptive RAG（classifier 決定深度），以及 multi-hop decomposition。若做三到四次迭代，每次查詢通常要預估 8-12 秒。參見 [Agentic RAG](../06-retrieval-systems/08-agentic-rag.md)。

<a id="how-do-computer-use-agents-work"></a>
### computer-use agents 如何運作？

Computer-use agent 會對桌面或瀏覽器截圖、決定滑鼠與鍵盤動作、執行它，再截下一張圖。三種 production 選項分別是 Claude Computer Use（透過截圖跨 OS 運作）、Google Gemini Computer Use（憑 DOM 感知對瀏覽器最佳化），以及 OpenAI Operator（專注於 web 任務）。Claude Sonnet 4.6 在 OSWorld-Verified 上達到 72.5%，比 2024 年 10 月發布時的 14.9% 大幅提升。參見 [Computer-Use Agents](../17-tool-use-and-computer-agents/04-computer-use-agents.md)。

---

<a id="models"></a>
## Models

<a id="what-is-the-best-llm-right-now"></a>
### 現在最好的 LLM 是哪個？

到 2026 年 5 月為止，沒有單一最好的模型。排行榜已經依任務分裂。**Claude Opus 4.7** 以 64.3% 領先 SWE-bench Pro 的複雜 coding 任務。**GPT-5.5** 以 82.7% 領先 Terminal-Bench 2.0 的 agentic terminal 工作。**Gemini 3.1 Pro** 以 94.3% 領先 GPQA Diamond 的科學推理。**Claude Sonnet 4.6** 則能以 Opus 4.7 約 40% 的價格，提供大約 90% 的品質。參見 [Model Taxonomy](../02-model-landscape/01-model-taxonomy.md)。

<a id="how-much-does-claude--gpt--gemini--deepseek-cost"></a>
### Claude / GPT / Gemini / DeepSeek 成本是多少？

定價每月都在變。以 2026 年 5 月來看，frontier-tier closed models 大約是每百萬 input token $3-15、每百萬 output token $15-75；若有 caching，重複 prefix 的成本可再降 75-90%。中階模型（Claude Sonnet 4.6、GPT-5.5-mini、Gemini 3.1 Flash）大約是每百萬 $0.30-3 / $1-15。**DeepSeek 重新定義了地板價**：V4 Pro 為每 1M $0.435 / $0.87（2026 年 5 月 22 日起 75% 折扣轉為永久），V4 Flash 則是每 1M $0.14 / $0.28，並提供 1M context window；對很多任務來說，大約比 closed frontier 便宜 10 倍。請務必交叉確認各 provider 的 pricing page。參見 [Pricing and Costs](../02-model-landscape/03-pricing-and-costs.md)。

<a id="what-is-the-difference-between-claude-opus-and-claude-sonnet"></a>
### Claude Opus 與 Claude Sonnet 有什麼差別？

Opus 是 Anthropic 的 frontier tier：最聰明、最慢、最昂貴。Sonnet 是 production workhorse：以約 40% 的價格提供約 90% 的 Opus 品質。Haiku 是高速 tier：便宜、低延遲，適合 routing 與 classification。正確模式通常是把簡單查詢導向 Haiku、中等查詢導向 Sonnet，只有最難的才導向 Opus。參見 [Model Selection](../02-model-landscape/04-model-selection-guide.md)。

<a id="should-i-use-an-open-source-model"></a>
### 我應該使用 open-source model 嗎？

如果你看重高流量下的單次查詢成本、data residency，或 fine-tuning 需求，答案是肯定的。Open-weight 模型的品質已縮小到距 closed frontier 約 5-15 分差距（Qwen3-Embedding-8B 領先 MTEB Multilingual，Llama 4 Maverick 與 DeepSeek V4 Pro 也已是有競爭力的 frontier 選擇）。但 tradeoff 在營運面：你必須自行負責 inference stack、GPU 帳單與 security patches。參見 [Model Landscape](../02-model-landscape/01-model-taxonomy.md)。

<a id="what-is-prompt-caching"></a>
### 什麼是 prompt caching？

Prompt caching 會讓固定 prompt prefix 的 KV cache 保持在 inference server 上是熱的，之後的呼叫只需為新增 token 付費。所有主流 provider 都支援它。Cache read 比重新計算的 token 便宜 75-90%。但代價是：cache write 通常貴 25%，所以只有在 TTL 內（通常 5 分鐘）至少重複使用同一 prefix 3-5 次時，caching 才划算。參見 [KV Cache and Context Caching](../04-inference-optimization/02-kv-cache-and-context-caching.md)。

---

<a id="evaluation"></a>
## Evaluation

<a id="how-do-you-evaluate-an-llm"></a>
### 你如何評估一個 LLM？

LLM evaluation 是分層進行的：快速迭代用 **reference-free metrics**（faithfulness、relevance、coherence，透過 LLM-as-judge）；回歸測試用 **golden test sets**；而像 QA 的 exact match、translation 的 BLEU、code 的 pass@k 等，則屬於 **task-specific metrics**。若特別是 RAG，請使用 RAG Triad：context relevance、faithfulness、answer relevance。參見 [LLM Evaluation](../14-evaluation-and-observability/01-llm-evaluation.md)。

<a id="what-is-llm-as-judge"></a>
### 什麼是 LLM-as-judge？

LLM-as-judge 是用一個 LLM 依照 rubric（correctness、helpfulness、safety）去評分另一個 LLM 的輸出。它能在人工評估無法擴展時提供規模化能力，但也有已知偏誤：position bias、verbosity bias、self-preference bias。標準做法是使用更強的模型作 judge（Claude Opus 4.7 或 GPT-5.5 reasoning）、隨機化位置，並用一小部分人工標註樣本做驗證。

<a id="what-is-the-best-llm-observability-tool"></a>
### 最好的 LLM observability 工具是哪個？

領先的平台包括 **Langfuse**（最好的 self-hosted open-source，於 2026 年 1 月被 ClickHouse 收購）、**Braintrust**（最適合 eval-driven CI/CD 與 quality gates）、**LangWatch**（最適合 agent simulation）、**LangSmith**（LangChain-native），以及 **Arize Phoenix**（OTel-native）。請依部署模式（SaaS vs self-hosted）、是否需要 CI/CD gating，以及你對特定 framework 的依賴程度來選擇。參見 [Observability](../14-evaluation-and-observability/02-observability.md)。

<a id="what-is-ragas"></a>
### 什麼是 RAGAS？

RAGAS（Retrieval Augmented Generation Assessment）是用於 RAG evaluation 的 Python library。它同時提供 reference-free metrics（faithfulness、answer relevance、context relevance）與 reference-based metrics（context recall、context precision），並透過 LLM-as-judge 計算。對任何 RAG eval pipeline 而言，它幾乎都是事實上的起點。參見 [RAG Evaluation](../06-retrieval-systems/13-rag-evaluation-patterns.md)。

---

<a id="inference"></a>
## Inference

<a id="what-is-vllm"></a>
### 什麼是 vLLM？

vLLM 是一個 open-source LLM inference engine，以開創 PagedAttention（以虛擬記憶體風格分配 KV cache）聞名。當工作負載是「Llama、Mistral、Qwen 或 DeepSeek，在 continuous batching 下執行」時，它是預設的開放式 inference engine。它也是主要引擎中最容易操作、修補最完整的一個。多模態部署必須使用 v0.18.2+，因為 2026 年 2 月有一個 CVE。參見 [Serving Infrastructure](../04-inference-optimization/06-serving-infrastructure.md)。

<a id="what-is-the-difference-between-vllm-and-sglang"></a>
### vLLM 與 SGLang 有什麼差別？

兩者都是 open-source inference engine。vLLM 的模型涵蓋面更廣，營運成熟度也更高。SGLang 則憑藉 async constrained decoding，在 structured-output 與 function-calling 工作負載上擁有約高 29% 的 throughput，並透過 RadixAttention 提供同級最佳的 prefix-cache reuse。重要警告是：截至 2026 年 5 月，SGLang 的 multimodal 路徑仍有未修補 CVE，因此 multimodal traffic 應改跑在 vLLM v0.18.2+。

<a id="what-is-tensorrt-llm"></a>
### 什麼是 TensorRT-LLM？

這是 NVIDIA 的 inference engine。在 H100/H200/B200 上，它可提供比 vLLM 與 TGI 高 2-4 倍的 throughput，但代價是需要 1-2 週的 setup，且完全綁定 NVIDIA。當你已投入 NVIDIA 容量，且 throughput 的提升能支付這筆營運稅時，它就是正確選擇。

<a id="how-do-you-optimize-llm-inference-cost"></a>
### 你如何優化 LLM inference 成本？

五個高槓桿作法：**model cascading**（簡單查詢導向小模型，困難查詢導向 frontier）、**prompt caching**（重複 prefix 省 75-90%）、**semantic caching**（相似查詢直接跳過 LLM 呼叫）、**quantization**（FP8 或 4-bit 權重，讓更多模型塞進單張 GPU），以及 **continuous batching**（vLLM/SGLang 在 iteration 層級做 batch）。這些方法加總後，經常能在不犧牲品質的情況下把 inference 成本壓低 10 倍。參見 [Cost Optimization](../04-inference-optimization/07-cost-optimization-playbook.md)。

<a id="what-is-speculative-decoding"></a>
### 什麼是 speculative decoding？

Speculative decoding 讓 LLM 每次 forward pass 可以生成多個 token：先由較便宜的「draft」模型（或主模型上的額外 heads，例如 Medusa）預測接下來幾個 token，再由目標模型在單次平行 pass 中一次驗證。收益是 wall-clock generation 可快 2-3 倍，且沒有品質損失。vLLM 與 TensorRT-LLM 都已內建支援。參見 [Speculative Decoding](../04-inference-optimization/03-speculative-decoding.md)。

---

<a id="memory"></a>
## Memory

<a id="what-is-the-best-ai-agent-memory-framework"></a>
### 最好的 AI agent memory framework 是哪個？

截至 2026 年 5 月，四個成熟選項為 **Mem0**（最全面的獨立 memory layer，在 LoCoMo 得分 92.5、在 LongMemEval 得分 94.4）、**Zep**（具時間感知的 production pipeline，原生支援對話摘要）、**Letta**（為長期運行 agents 提供類作業系統的 paging 與無界 memory），以及 **Cognee**（以 knowledge graph 為核心，適合 RAG-heavy workflow）。選擇方式依 use case：chatbot personalization → Mem0；大規模 production agent → Zep；長時程 task agent → Letta；KG-grounded RAG → Cognee。參見 [Agentic Memory](../08-memory-and-state/04-agentic-memory-mem0.md)。

<a id="what-is-the-difference-between-short-term-and-long-term-memory-in-agents"></a>
### agents 的短期記憶與長期記憶有什麼差別？

短期記憶存在於 LLM 的 context window 中（當前回合、tool outputs、scratchpad）。長期記憶則會跨 session 持久保存在 vector DB、graph 或 relational store 中。Memory architecture 還可再拆成：**episodic**（過去軌跡）、**semantic**（關於使用者／世界的抽取事實）、**procedural**（學到的技能、playbook）。對某項事實選錯層級會造成問題：若把只屬於某一個 session 的偏好升級成長期記憶，就會外洩到其他 session。參見 [Memory Architectures](../08-memory-and-state/01-memory-architectures.md)。

<a id="how-does-a-knowledge-graph-help-an-ai-agent"></a>
### knowledge graph 如何幫助 AI agent？

Knowledge graph 會明確儲存 entities 與 relationships（User → OWNER_OF → Project_A）。它能提供 agent 對結構化關係的**確定性** retrieval，這是 vector search 做不到的。最強的模式通常是混合式：vector search 先依相似度找出入口節點，再由 graph traversal 展開相關 context。這在 compliance、多跳推理，以及關係本身就是一等公民的領域（legal、biomedical、finance）中特別常見。參見 [Long-Term Memory](../08-memory-and-state/03-long-term-memory.md)。

---

<a id="security"></a>
## Security

<a id="what-is-prompt-injection"></a>
### 什麼是 prompt injection？

Prompt injection 是 LLM 時代版的 SQL injection：惡意內容藏在使用者輸入或檢索到的文件中，覆蓋 system instructions，讓模型做出本來不該做的事。**Direct injection** 出現在使用者 prompt 中；**indirect injection** 則藏在模型會讀取的文件裡（網頁、email、PDF）。OWASP LLM Top 10 將其列為頭號 LLM 風險。參見 [Prompt Injection Defense](../05-prompting-and-context/08-prompt-injection-defense.md)。

<a id="how-do-you-prevent-prompt-injection"></a>
### 你如何防止 prompt injection？

沒有銀彈：prompt injection 無法像 SQL 那樣被完全「escape」。Production 的防禦堆疊通常會結合：**input isolation**（用 XML tags 標示不可信內容）、**dual-LLM patterns**（主模型看到輸入前，先由小 guard model 判斷意圖）、**canary tokens**（偵測模型是否洩漏 system prompt）、**least-privilege tool scopes**，以及對破壞性 tool call 加上 **human-in-the-loop**。參見 [Agentic Security](../07-agentic-systems/09-agentic-security-and-sandboxing.md)。

<a id="what-is-owasp-llm-top-10"></a>
### 什麼是 OWASP LLM Top 10？

OWASP Top 10 for LLM Applications（v2.0，發布於 2025 年）是最權威的 LLM security risk 清單。前幾項包括：prompt injection、insecure output handling、training data poisoning、model denial of service、supply chain vulnerabilities、sensitive information disclosure、insecure plugin design、excessive agency、overreliance，以及 model theft。針對 agentic apps 的 2026 更新版，則把 goal hijacking、identity abuse 與 cascading failures 納入主要風險。參見 [LLM Security](../12-security-and-access/01-llm-security.md)。

<a id="what-is-sandboxing-in-ai-agents"></a>
### 什麼是 AI agents 中的 sandboxing？

Sandboxing 會把 agent 生成並執行的程式碼與主機系統隔離開來。標準模式是使用暫時性的 micro-VMs（E2B、Docker、Firecracker），在 10ms 內啟動、執行程式碼，再銷毀。沒有 sandboxing 時，遭到 prompt injection 的 agent 可能 `rm -rf /` 或外洩 secrets；有了 sandboxing，最糟的情況只會是毀掉一個可拋棄容器。

---

<a id="related-reading"></a>
## 延伸閱讀

- [Question Bank（110 senior 面試題）](01-question-bank.md)
- [Answer Frameworks](02-answer-frameworks.md)
- [Common Pitfalls](03-common-pitfalls.md)
- [Whiteboard Exercises](04-whiteboard-exercises.md)
- [AI Job Market Trends](06-job-market-trends-2026.md)

---

*有其他也應該收錄在這裡的問題嗎？歡迎在 repo 開 issue 或 PR。*

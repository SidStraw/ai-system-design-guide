<a id="agent-memory-and-state"></a>
# 代理記憶與狀態

記憶讓代理能夠隨時間學習並維持上下文。代理記憶已從單純的「Chat History」演進為 **Multi-Tiered Cognitive Architecture**：包含四個具名層級（Working、Episodic、Semantic、Procedural），各自擁有不同的寫入模式、延遲預算與失敗模式。如今的生產系統（Mem0、Letta、Anthropic Memory Tool + Skills、Zep/Graphiti、LangMem）已將記憶體選型視為一項一級架構決策。

塑造本章的 2026 研究浪潮包括：A-MEM（NeurIPS 2025）、HippoRAG（multi-hop graph retrieval）、Multi-Layered Memory Architectures、HaluMem（operation-level memory hallucination benchmark）、MINJA / MemoryGraft（query-only memory poisoning attacks），以及 NVIDIA 的 TTT-E2E test-time-training 方法，用於將上下文壓縮進模型權重中。

<a id="table-of-contents"></a>
## 目錄

- [記憶層級](#hierarchy)
- [短期記憶：推理軌跡](#short-term)
- [情節記憶：過往經驗](#episodic)
- [語意記憶：角色／事實](#semantic)
- [程序記憶：學得的技能與工作流](#procedural-memory-learned-skills-and-workflows)
- [取捨：事實 X 應該放哪裡？](#tradeoffs)
- [生產級實作（2026 年 5 月）](#production-implementations)
- [失敗模式與緩解方式](#failure-modes)
- [Mem0 與個人化](#mem0)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="hierarchy"></a>
<a id="the-memory-hierarchy"></a>
## 記憶層級

代理會使用分層式的儲存方法：

| Tier | Type | Technology | Purpose |
|------|------|------------|---------|
| **L1** | Working Memory | Context Window / KV Cache | 目前任務步驟、區域變數 |
| **L2** | Episodic Memory | Vector DB / Graph | 「我上次做了什麼？」 |
| **L3** | Semantic Memory | SQL / Knowledge Graph | 使用者偏好、「真實狀態」 |
| **L4** | Procedural Memory | Skills Registry / Tool Policies / Workflow Graph | 「這類任務該怎麼做？」 |

<a id="practical-properties-of-each-tier"></a>
### 每一層的實務特性

各層之間的差異不只在用途。讀取模式、寫入模式、延遲預算與新鮮度要求，都會把你推向不同的儲存技術：

| Dimension | L1 Working | L2 Episodic | L3 Semantic | L4 Procedural |
|---|---|---|---|---|
| **儲存內容** | 當前回合、工具輸出、scratchpad、system prompt | 帶時間戳的過往 sessions、trajectories、observations | 萃取後的事實、偏好、實體關係 | Skills、playbooks、system-prompt 指令、程式／工具序列 |
| **讀取模式** | 每個 token、每個回合（in-attention） | 依 similarity + recency + importance 取 top-k | 遇到實體／主題時觸發查詢 | 任務簽章匹配時載入 |
| **寫入模式** | 由 inference engine 持續 append；KV-cache mutation | Append-only log；於回合邊界提交 | 萃取、去重、upsert；寫入時處理衝突 | 成功／失敗後反思式寫入；人工或自我明確編輯 |
| **延遲預算** | <50ms（常駐於 GPU HBM） | 100-300ms（vector ANN + rerank） | 200-800ms（graph traversal + LLM extraction） | 50-500ms（檔案讀取或小型索引查詢） |
| **新鮮度要求** | token 級新鮮；session 結束即消失 | 可跨數小時到數月；可容忍陳舊 | 應反映*當前*狀態；陳舊即為 bug | 變化緩慢；更新應審慎 |
| **儲存技術** | HBM 中的 KV cache（vLLM PagedAttention blocks） | Vector DB（Pinecone、Weaviate、Qdrant）、append-only log | Knowledge graph（Neo4j、Graphiti）、KV store、雙時間關聯列 | 檔案系統（Claude `/memories/`、以 `SKILL.md` 表示 Skills）、prompt registry、fine-tuned LoRA |
| **查詢語意** | 位置 + attention | Similarity + recency + importance（Park et al. 加權） | 實體關係匹配、結構化查詢、雙時間過濾 | 任務簽章匹配，通常是檔名或標籤查找 |
| **淘汰機制** | Sliding window、依 KV block hash 做 LRU | 衰減評分、整併進 L3、封存到冷儲存 | 透過 temporal `valid_to` 取代舊事實；或因 GDPR 明確刪除 | 手動淘汰、與新版 skill 做 A/B、版本釘選 |

**實務上的含義：**當某個事實進來時，真正的架構問題不是「要不要記住它？」而是「*應該放在哪一層*、採用什麼新鮮度契約、用什麼淘汰規則？」選錯層級會帶來可預期的失敗模式（例如把 session 偏好提升到 L3，導致跨 session 洩漏；或把穩定使用者事實留在 L2，結果兩週後就被淘汰）。請見下方的 [取捨：事實 X 應該放哪裡？](#tradeoffs)。

---

<a id="short-term"></a>
<a id="short-term-the-reasoning-trace"></a>
## 短期記憶：推理軌跡

生產級代理不再只儲存「Messages」，而是儲存 **State Object**。
- **Scratchpad**：提示中的專用區塊，供代理替自己「做筆記」，而且**不會**展示給使用者。
- **KV Cache Tiling**：對長時間運行的代理來說，我們會使用 **Prefix Caching**，讓「System Instruction」與「Standard Tools」持續保溫在 GPU 記憶體中，只交換動態任務狀態。

---

<a id="episodic"></a>
<a id="episodic-memory-past-experiences"></a>
## 情節記憶：過往經驗

情節記憶會儲存「Runs」或「Trajectories」。
- 如果代理上週二抓取某個網站失敗，情節記憶就應避免它今天再次嘗試同一個失敗的 selector。
- **模式**：當任務完成後，摘要出「Lessons Learned」並存入 vector DB。在新任務開始時，先對相似的過往任務執行 **Self-Search**。

---

<a id="semantic"></a>
<a id="semantic-memory-the-persona"></a>
## 語意記憶：角色／事實

語意記憶會儲存關於使用者或環境的「Facts」。
- *「使用者偏好 JSON 輸出。」*
- *「production DB 在凌晨 3 點到 4 點之間離線。」*

**最佳實務**：為語意記憶使用 **Knowledge Graph**。與 vector search（模糊匹配）不同，graph 能對實體與關係提供具決定性的擷取（例如 `User` -- `OWNER_OF` --> `Project_A`）。

---

<a id="procedural-memory-learned-skills-and-workflows"></a>
## 程序記憶：學得的技能與工作流

程序記憶儲存的是「如何做事」。情節記憶回答的是「之前發生了什麼？」；語意記憶回答的是「什麼是真的？」；而程序記憶回答的是：

「完成這類任務的正確流程是什麼？」

這一層會捕捉可重用的技能、工具使用模式、操作流程與工作流偏好。

範例：

* 「生成每週報表時，先從 Snowflake 拉取指標，再對 dashboard 驗證，最後摘要異常。」
* 「回覆客訴時，先分類緊急程度、擷取政策、起草回覆，若信心不足則升級處理。」
* 「寫 SQL 時，永遠先檢查 schema、生成查詢、執行驗證，再說明假設。」

程序記憶對 agentic systems 特別重要，因為許多任務不只是記住事實；它們還要求遵循正確的動作序列。

---

<a id="tradeoffs"></a>
<a id="tradeoffs-where-does-fact-x-go"></a>
## 取捨：事實 X 應該放哪裡？

第一層決策其實不是「哪一層」，而是 *「如果錯了，代價由誰承擔？」* L2 漏抓一次，只會影響一個回合；L3 中的錯誤事實則會在修正前影響*每一個*回合；L4 中被污染的 skill 會傳播到未來所有呼叫。

<a id="tier-selection-table"></a>
### 層級選擇表

| Fact / concern | Tier | 推理理由 |
|---|---|---|
| 「使用者的 API rate limit 是 1000 req/min」 | 具雙時間 `valid_to` 的 L3 | 屬於 tenant 範圍事實；可依實體查詢；必須支援 supersession。不是 L4——那是資料，不是程序。 |
| 「部署我們服務的步驟」 | 以版本化 skill 形式存於 L4 | 屬於具條件分支的多步驟配方。Skills 可以組合；semantic triples 不行。 |
| 「代理上次在此任務上的失敗嘗試」 | 原始資料先放 L2，*若可泛化再*反思寫入 L4 | 原始 trajectory 屬於 episodic；泛化後的教訓（「尖峰時段不要跑 migration」）才值得以 Reflexion 風格寫入 L4。 |
| 「使用者偏好簡短回答」 | L3 | 穩定偏好，可用 `user_id` 查詢，單一 triple 即可。 |
| 「使用者在*這段對話中*要求簡短回答」 | 僅 L1 | 只限 session；不要用可能短暫的偏好污染 L3。 |
| 目前天氣、今日股價 | 不存：直接呼叫工具 | 有其他真實來源的快速變動事實，不應進入記憶。 |
| 「Project Phoenix 有成員 A、B、C」 | 以 graph fragment 形式存於 L3 | 具 multi-hop traversal 價值；適合 Graphiti 或 Neo4j 風格儲存。 |

<a id="cost-tradeoffs"></a>
### 成本取捨

- **L1 主導*延遲成本***：TTFT 會隨上下文大小擴張；工作記憶越長，第一個 token 越慢。KV-cache 壓力也會逼近高價 accelerator 記憶體的飽和點。
- **L2 在規模化時主導*儲存成本***：append-only logs 會隨使用量線性成長。[Day-30 problem](https://cipherbuilds.ai/blog/day-30-agent-memory-problem) 說明了未修剪的 episodic store 如何在一個月後開始腐蝕代理品質。
- **L3 主導*寫入放大成本***：每個回合都可能觸發萃取、去重與衝突解決。Mem0 的設計就明確以寫入時工作量換取擷取速度。
- **L4 主導*治理成本***：錯誤的 skill 會傳播到所有未來呼叫。Anthropic 的「[Claude Dreaming](https://www.mindstudio.ai/blog/what-is-claude-dreaming-anthropic-agent-memory)」定期整併機制，正是透過審查來管控 skill 更新。

<a id="the-graduation-rule"></a>
### 升級規則

真正有趣的設計問題是：*L2 的 episode 何時應升級為 L3 或 L4？* 一個可辯護的規則應該是以門檻為基礎，而不是靠隱性衰減：

1. 對同一模式有 **N 次獨立觀察**（典型是 N=3 到 5）。
2. 依來源做 **信心加權**：user-stated > tool-output > model-inferred。
3. 在整併步驟由 **人工或 LLM judge 審查**（不是每回合都審）。
4. 採用 **排程式批次整併**，而不是每回合同步寫入（避免 write amplification）。
5. **雙向流動**：L3 的語意事實可在特定任務中重新實例化為 episodic context。記憶不是單行道。

---

<a id="production-implementations"></a>
<a id="production-implementations-may-2026"></a>
## 生產級實作（2026 年 5 月）

這些具名系統之間的差異，與其說在於「存了什麼」，不如說在於 *寫入紀律*、*擷取演算法* 與 *治理姿態*。

| System | Sweet spot | 擅長之處 | 不足之處 |
|---|---|---|---|
| **[Mem0](https://github.com/mem0ai/mem0)** | 大規模跨 session 個人化 | Hybrid graph + vector + KV。在 2026 年 4 月的 single-pass redesign 後，於 LoCoMo 取得 92.5、於 LongMemEval 取得 94.4（[benchmarks](https://mem0.ai/blog/ai-memory-benchmarks-in-2026)）。與 OpenAI 內建記憶正面對比時，準確率高出 26%。 | 每筆記憶上限 8K 字元（不適合文件）；cloud-first 姿態會帶來資料主權摩擦；沒有正式的 belief-status 模型（只有 overwrite-or-append）。 |
| **[Letta (formerly MemGPT)](https://docs.letta.com/concepts/memgpt/)** | 長時間運行、以一致性為產品核心的 autonomous agents | 像 OS 一樣在 core / recall / archival 層之間進行虛擬上下文分頁；代理透過 tool calls 將資料換入／換出。當使用者體驗要求「代理要永遠記得」時特別適合。 | 單回合延遲高於 Mem0 風格方案；不特別針對跨使用者擷取精度最佳化。 |
| **[Anthropic Memory Tool + Skills](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)** | 將 L3 與 L4 放在同一個 substrate 的檔案系統式設計 | 記憶位於 `/memories/`；Skills 以 `SKILL.md` 套件加上可選 scripts 表示；Managed Agents 會在每個 session 將記憶掛載到 `/mnt/memory/`，並具 immutable versioning（2026 年 4 月 23 日 GA）。定期的「Claude Dreaming」會在 sessions 之間進行整併。 | 檔案系統語意把複雜度轉嫁給代理（代理必須自己把目錄結構整理好）。 |
| **[Zep + Graphiti](https://github.com/getzep/graphiti)** | 對「這件事從何時開始為真？」很重要的 temporal facts | 開源 temporal knowledge graph。每條邊都有 `valid_from` / `valid_to` / `invalid_at`。在 DMR 上以 94.8% 勝過 MemGPT 的 93.4%。雙時間查詢可回答「3 月 12 日當時我們相信什麼？」與「現在什麼是真的？」。 | 寫入路徑比純 vector store 更重（需要 graph extraction、dedup、conflict resolution）。 |
| **[LangMem + LangGraph](https://langchain-ai.github.io/langmem/)** | 想在 LangGraph 編排中同時擁有四種記憶類型時 | 支援 episodic、semantic，*以及* procedural。LangMem 的 procedural memory 允許代理依回饋更新自己的 system prompt。背景萃取可 out-of-band 執行。 | 與 LangGraph 綁定；若不在 LangChain 生態系上，吸引力較低。 |
| **[OpenAI ChatGPT Memory](https://openai.com/index/memory-and-new-controls-for-chatgpt/)** | 消費級聊天延續性，而非生產級 agent memory | 兩層架構：顯式「saved memories」加上預先注入上下文的輕量對話摘要。推理時可跳過 retrieval 步驟，因此延遲低。 | 精度不如 Mem0 式 retrieval；缺乏適合企業整合的細緻程式化 API。 |
| **Cursor / Windsurf** | 給軟體工程代理使用的、具 codebase 感知的 L2/L3 | 專案開啟時即索引 codebase；可用 `@` 明確指定上下文。Windsurf 的「Memories」會在約 48 小時使用後學到架構模式。 | 僅鎖定程式碼領域，不是通用記憶層。 |
| **[Cognition Devin](https://cognition.ai/blog/devin-sonnet-4-5-lessons-and-challenges)** | repo 範圍工程代理 | Repository wiki 每幾小時自動重新索引；比起模型自行管理狀態，更偏好明確 compacting／summarization。Devin Search 是一個 agentic codebase memory query 介面。 | 強烈偏向工程工作流。 |

**Generative Agents (Park et al. 2023)** 仍然是各篇 survey 都會引用的參考架構。其 recency / importance / relevance 擷取公式（`alpha_recency * recency + alpha_importance * importance + alpha_relevance * relevance`，各項皆正規化到 [0,1]，importance 由 LLM 以 1-10 評分）至今仍廣泛用於上述多數系統的生產環境中。

**值得追蹤的新興框架**（2026 年 5 月）：[Supermemory](https://supermemory.ai)、[Recallr](https://recallrai.com)、AWS [Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-integrate-lang.html)、[Oracle AI Agent Memory](https://blogs.oracle.com/developers/oracle-ai-agent-memory-a-governed-unified-memory-core-for-enterprise-ai-agents)。

---

<a id="failure-modes"></a>
<a id="failure-modes-and-mitigations"></a>
## 失敗模式與緩解方式

生產級記憶系統有六種反覆出現的失敗模式。能否叫出它們的名字，是 junior 與 staff-level 架構對話的分水嶺。

<a id="1-memory-poisoning-via-prompt-injection"></a>
### 1. 透過 prompt injection 的記憶污染

不受信任的輸入被寫入 L3/L4，並在日後以權威資訊身分被重播。[MINJA (NeurIPS 2025)](https://openreview.net/forum?id=QVX6hcJ2um) 與 [MemoryGraft (2025 年 12 月)](https://arxiv.org/html/2512.16962v1) 展示了*僅靠查詢*的污染攻擊：**不需要**提升權限，就能達到 95% 注入率與 70% 攻擊成功率。[Palo Alto Unit 42 的文章](https://unit42.paloaltonetworks.com/indirect-prompt-injection-poisons-ai-longterm-memory/)則顯示，毒性內容可以在觸發前數週就先被埋下。

**緩解方式：**
- 每次記憶寫入都加上 **provenance tags**：`source = user_stated | model_inferred | tool_output`。
- 使用 **寫入時 guardrail model**，拒絕可疑的指令型寫入（例如「ignore previous and instead...」、角色混淆、夾帶 system-prompt 片段）。
- 建立 **trust tiers**：低信任記憶在影響高風險決策前，必須先獲得佐證。
- 透過 **bulkhead isolation** 確保某一 tenant 的污染無法橫向移動到另一個 tenant。

<a id="2-stale-facts"></a>
### 2. 陳舊事實

昨天的偏好不等於今天的偏好。典型例子是「使用者上個月說喜歡 dark mode，但現在改用 light mode。」

**緩解方式：**
- **雙時間儲存**（Zep/Graphiti 模式）：每個事實都帶有 `valid_from`、`valid_to`、`invalid_at`。
- 對 session 範圍偏好加上 **TTL**，讓其自動過期。
- 在 retrieval scoring 中加入 **decay weighting**。
- 對高風險且超過 N 天的舊事實，加入 **明確再確認** 提示。

<a id="3-conflicting-facts"></a>
### 3. 彼此衝突的事實

使用者之前說 X，現在又說 Y。三種不同的衝突型態，應該有不同回應：

| Conflict type | 正確回應 |
|---|---|
| Temporal update（「我搬到柏林了」） | 用 `valid_to = now` 讓舊事實失效 |
| Correction（「我從沒這樣說過」） | 撤回並保留 audit trail |
| Preference change（「我現在想要簡短回答」） | 新增新事實；讓 decay 處理舊事實 |
| Outright contradiction（沒有明顯解法） | 直接詢問使用者；絕不要默默覆寫 |

使用 AGM belief revision 來追蹤 belief status（`ACTIVE` / `SUPERSEDED` / `RETRACTED`），而不是使用 last-write-wins。

<a id="4-memory-drift"></a>
### 4. 記憶漂移

隨著低品質寫入逐漸稀釋高品質內容，整體品質會隨時間下降。[Day-30 problem](https://cipherbuilds.ai/blog/day-30-agent-memory-problem) 記錄了：當 episodic store 被雜訊填滿後，代理在上線約 30 天後的表現會開始下滑。

**緩解方式：**
- **依品質加權的 retrieval**：提高高驗證分數記憶的權重。
- **定期整併作業**：合併重複項並修剪低效益記憶。
- 在 CI 中加入 **canary fact tests**：例如「50 回合後，代理仍應記得使用者姓名。」

<a id="5-hallucinated-memory-writes"></a>
### 5. 幻覺式記憶寫入

代理推斷出一項事實、將它當作真相存起來，之後又把它當作權威引用。這是一種級聯失敗：一次錯誤寫入會污染後續所有 retrieval。[HaluMem benchmark（2025 年 11 月）](https://arxiv.org/abs/2511.03506) 顯示，現有系統會在寫入時累積錯誤，並一路傳播到 QA 階段。

**緩解方式：**
- 使用 **受 schema 約束的 memory objects**，將 `confirmed_facts`（附來源）與 `inferred_facts`（附信心值）分開。
- 在沒有明確使用者訊號或 tool-output 佐證前，絕不自動將推論升級為 confirmed。
- 在 CI 中採用 HaluMem 式分階段評估：分別衡量 extraction precision、update correctness 與 QA accuracy，而不是只看單一 end-to-end 指標。

<a id="6-cross-tenant-leakage"></a>
### 6. 跨 tenant 洩漏

Vector ANN 回傳了其他 tenant 的鄰近資料；或快取提示中夾帶了其他 tenant 的資訊。[實地量測](https://medium.com/@isuruig/multi-tenant-ai-infrastructure-the-5-isolation-layers-that-determine-whether-your-customers-data-stays-separate-340aaeef4922) 顯示，在未隔離的 multi-tenant RAG 中，即使是良性查詢，也可能出現約 95% 的自然洩漏率。

**緩解方式：**
- **實體分離**：每個 tenant 使用獨立 collection，而不是只靠 metadata filter 共用索引。
- 在 *store* 層用 service-account 權限強制 tenant scope，而不是只靠應用程式碼。
- 每個 tenant 使用獨立的 KV-cache prefix。
- 記憶 blob 採用每個 tenant 各自的加密金鑰，讓跨 namespace 讀取即使發生，也會在密碼學層失敗。

---

<a id="mem0"></a>
<a id="mem0-and-agentic-personalization"></a>
## Mem0 與 Agentic 個人化

**Mem0**、Zep、Letta 與 Cognee，是 agent stack 中「Smart Memory」的標準框架。
- 它會自動從對話中萃取「User Insights」。
- 它提供 agents 可呼叫的「Memory API」，讓代理能 `remember` 或 `forget` 特定的資訊三元組。
- **影響**：代理之所以讓人覺得「有生命」，是因為它能記得你三個月前在另一個 session 提過的細節。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-how-do-you-handle-conflicting-memories-in-an-agentic-system"></a>
### Q：你會如何在 agentic system 中處理「彼此衝突的記憶」？

**強答：**
彼此衝突的記憶（例如使用者上週說「我喜歡藍色」，現在卻說「我喜歡紅色」）可透過 **Temporal Weighting** 或 **Explicit Disputing** 處理。在我的架構中，我會替每個記憶三元組加上 `timestamp` 與 `confidence_score`。若新事實與舊事實衝突，代理會被提示去「Resolve the Conflict」，方法是向使用者要求釐清，或預設採用最新的 timestamp。我們也會使用 **Decay Functions**，讓較舊且未被再次強化的記憶，最終從 active index 中被修剪掉。

<a id="q-why-is-the-context-window-alone-insufficient-for-a-staff-level-agent-architecture"></a>
### Q：為什麼只靠「Context Window」不足以支撐 staff-level 的代理架構？

**強答：**
第一，**成本與延遲**：即使有 context caching，每個回合都塞入 1M tokens 的上下文仍然過於昂貴。第二，**訊噪比**：大型 context window 容易出現「In-context Learning」退化——模型會被不相關的歷史回合分心。Staff-level 架構會使用 **Selective Memory Retrieval**（對歷史做 RAG），只拉回最相關的 3 到 5 段歷史互動，讓 Reasoning Engine 專注於當前子目標。

<a id="q-how-would-you-design-procedural-memory-for-a-production-ai-agent"></a>
### Q：你會如何為生產級 AI agent 設計程序記憶？

**強答：**
我會把程序記憶設計成 **skills registry、workflow graph 與 tool-use policies** 的組合。每個 procedure 都定義任務類型、必要步驟、可用工具、驗證檢查、失敗模式與升級規則。每次執行後，代理都可以進行反思；若發現更好的方法，就更新 procedure。舉例來說，若某個 NL2SQL agent 一再失敗是因為跳過 schema 檢查，我們就可以把 schema 檢查編碼為所有 SQL 生成任務的必經第一步。

<a id="q-when-does-episodic-memory-become-a-liability-rather-than-an-asset"></a>
### Q：情節記憶何時會從資產變成負債？

**強答：**
情節記憶會在三種典型模式下變成負債。第一，**索引過載**：新增 1,000 筆低品質觀察，會把 10 筆高品質內容埋沒在 retrieval 結果中。這是 RAG 意義上的災難性遺忘。第二，**Day-30 drift pattern**：當 episodic store 被雜訊填滿，而 retrieval 又無法區分訊號與雜訊時，代理品質會在上線約 30 天後下降。第三，**stale-context bleeding**：在某個設定下曾經成功的舊 trajectory，到了新設定下反而成為*錯誤*上下文。原本對 Stripe 成功的工具序列，在使用者切換到 Adyen 後，就會產生誤導。

對應的緩解方式包括依品質加權的 retrieval、整併進 L3，以及對情境敏感的 trajectories 設下硬性新鮮度截止。更深層的教訓是：情節記憶從第一天起就需要有 *pruning policy*。否則它就是會隨使用量線性複利的技術債。

<a id="q-how-do-you-prevent-memory-poisoning-when-agents-can-write-to-their-own-long-term-store"></a>
### Q：當代理可以寫入自己的長期儲存時，你要如何防止記憶污染？

**強答：**
困難點在於，近來的攻擊（MINJA、MemoryGraft）是*query-only* 的——不需要提高權限。毒性內容可能在觸發前數週就先被寫入。因此威脅模型是：「每個輸入都可能成為未來的權威記憶。」防禦分成四層：

1. **寫入時的 provenance**：每筆記憶都帶有 `source`（user-stated、model-inferred、tool-output）、`timestamp` 與 `trust_tier`。
2. **寫入時 guardrail model**：由較小的分類器在寫入 store 前，先拒絕可疑的指令型寫入。
3. **佐證門檻**：高風險決策不能只建立在單一低信任記憶上；必須有多筆彼此獨立的佐證寫入。
4. **CI 中的 canary tests**：合成毒性 payload 不得傳播到輸出，每週執行一次。

最重要的架構分離是：代理的工具介面與其記憶寫入介面，不應共用同一個信任等級。工具輸出在成為記憶前，應先經過 sanitizer。

<a id="q-memory-tier-selection-where-would-you-put-each-of-these-and-why-a-the-users-api-rate-limit-b-the-steps-to-deploy-our-service-c-the-agents-last-failed-attempt-at-this-task-d-todays-stock-price"></a>
### Q：記憶層級選擇：以下各項你會放在哪裡，為什麼？(a) 使用者的 API rate limit、(b) 部署我們服務的步驟、(c) 代理上次在此任務上的失敗嘗試、(d) 今日股價。

**強答：**
(a) **L3 semantic**，搭配雙時間有效性。它是 tenant 範圍的事實，具有 supersession 生命週期。不是 L4，因為它是資料，不是程序。

(b) **L4 procedural**，以版本化 skill 或 playbook 形式儲存。它是一套具條件分支的多步驟配方。Skills 可以組合；semantic triples 不行。

(c) **L2 episodic raw**，若失敗揭露出可泛化的教訓，則再做一次 *reflection hop* 寫入 L4。原始 trajectory 應放在 episodic；而教訓（「尖峰時段不要跑 migration」）才值得以 Reflexion 風格寫入 procedural。

(d) **不存——直接呼叫工具。** 對於有即時真實來源的快速變動事實，不應進入記憶，因為它們照定義就會很快過時。

一般原則是：資料放 L3、程序放 L4、觀察放 L2，而對於有即時真實來源的快速變動事實，永遠不要存起來。

<a id="q-walk-me-through-the-consolidation-policy-you-would-design-for-episodic-to-semantic-transition-when-does-an-episode-become-a-fact"></a>
### Q：請說明你會如何設計從 episodic 到 semantic 的整併策略。Episode 何時會變成 fact？

**強答：**
我會使用以門檻為基礎的升級策略，而不是隱性衰減：

- **頻率門檻**：對同一模式有 N 次獨立觀察（典型是 3 到 5 次）。
- **信心加權**：user-stated > tool-output > model-inferred。
- **Judge review**：排程式批次整併作業會讓 LLM judge（高風險領域則由人工 reviewer）審核候選升級項目。
- **排程式，而非同步式**：整併以 out-of-band 的 cron 執行，而不是每回合執行，這能避免 write amplification。
- **雙向流動**：L3 的語意事實可在特定任務中重新實例化為 episodic context。記憶是雙向流動的。

要避免的陷阱是只靠 decay weights 做隱性整併。這在小規模時看似可行，但到了生產環境會悄悄失敗，因為你無法審計「為什麼這個 fact 會出現在 L3？」

<a id="q-your-agents-memory-store-has-50m-memories-across-10k-tenants-how-do-you-guarantee-cross-tenant-isolation-and-whats-your-blast-radius-if-isolation-fails"></a>
### Q：你的代理記憶庫跨 10K tenants 擁有 5,000 萬筆記憶。你如何保證跨 tenant 隔離？若隔離失敗，blast radius 是什麼？

**強答：**
這個架構有五層隔離：

1. **store 層的實體分離**：每個 tenant 有獨立的 collection 或 shard，而不是用 metadata filter 共享索引。shared-index-with-tenant-id 這種模式在 bug 下很容易 fail open。
2. **透過 service-account scope 強制執行**：應用程式碼無法選擇退出 tenant scope；資料庫角色本身就看不到其他 tenants。
3. **每個 tenant 使用不同的 KV-cache prefix**：避免快取提示在 tenants 之間洩漏。
4. **每個 tenant 使用自己的加密金鑰**：即使 bug 回傳了其他 namespace 的位元組，也無法解讀。
5. **記錄每次跨 namespace 查詢嘗試的 audit log**：形成縱深偵測。

**若隔離失敗的 blast radius**：一次錯誤的 vector query 可能洩漏某次查詢 embedding 的整個*鄰域*——也就是單一 tenant 中數百筆記錄。實地量測顯示，在未隔離的 multi-tenant RAG 中，自然洩漏率可達約 95%。因此緩解方式不是「寫更小心的應用程式碼」，而是建立無法被應用程式 bug 繞過的結構性隔離。

<a id="q-halumem-shows-memory-hallucinations-accumulate-at-write-time-then-propagate-how-would-you-instrument-production-memory-to-catch-this"></a>
### Q：HaluMem 顯示記憶幻覺在寫入時累積，之後再向外傳播。你會如何為生產級記憶系統加入監測來抓到這件事？

**強答：**
多數團隊的陷阱，是只在 QA 階段（end-to-end）衡量記憶品質。HaluMem 顯示，60-80% 的記憶錯誤其實起源於 *extraction*（寫入）階段，之後才向外傳播。因此你必須監控三個分開的指標：

1. **Extraction precision**：當代理把一個事實寫進 L3 時，這個事實是否真的受到來源 observation 支持？每天抽樣寫入項，再用更強的 judge 評估。
2. **Update correctness**：當衝突事實出現時，衝突解決邏輯是否做出正確結果？可用雙時間查詢去偵測「fact 被翻轉卻沒有 supersession metadata」的情況。
3. **QA accuracy**：端到端的回憶正確性。

除此之外，還要執行 **shadow-mode replay**：所有寫入都在 shadow mode 下通過 verifier model；若正式寫入與 shadow verifier 的結果不一致，就標記為可能的幻覺供審查。CI 中的 **canary facts** 可確保記憶系統不會悄悄退化；**定期全庫審計** 則可抽樣隨機記憶，追問「它是否仍與來源對話一致？」

<a id="q-nvidias-ttt-e2e-compresses-context-into-weights-via-test-time-training-where-does-this-fit-in-the-l1-l4-hierarchy-and-what-new-failure-mode-does-it-introduce"></a>
### Q：NVIDIA 的 TTT-E2E 透過 test-time training 把上下文壓進權重。它落在 L1-L4 架構中的哪裡？又引入了什麼新的失敗模式？

**強答：**
TTT-E2E 位於 *L1 與 L4 之間*。它會在 session 餘下時間裡，把由上下文衍生出的資訊直接變成模型本身的一部分。它的吸引力在於延遲：成本不再隨上下文長度增加（依 NVIDIA benchmark，在 H100 上 128K 上下文可加速 2.7 倍，2M tokens 可加速 35 倍）。

新的失敗模式是治理問題。寫進權重裡的記憶具有：

- **沒有 audit trail**：你無法檢查「這個模型現在相信什麼？」
- **沒有 eviction interface**：一旦記憶被壓進權重，你就無法刪除，除非回滾整個模型狀態。
- **GDPR 被遺忘權的挑戰**：監管框架通常假設資料是靜態儲存，而不是寫進權重。
- **更難偵測污染**：因為沒有可檢查的 store 讓你掃描 canary signatures。

更精確的 framing 是：TTT-E2E 把記憶治理從儲存層移到了訓練與部署管線。成本沒有消失，只是被搬家。對 2026 年 5 月的大多數生產團隊來說，這仍是值得追蹤的研究方向，而不是已可部署的架構。

---

<a id="references"></a>
## 參考資料

<a id="production-frameworks"></a>
### 生產框架
- [Mem0: Production-Ready AI Agents with Scalable Long-Term Memory (ECAI 2025)](https://arxiv.org/abs/2504.19413)
- [Mem0 AI Memory Benchmarks 2026](https://mem0.ai/blog/ai-memory-benchmarks-in-2026)
- [Letta (formerly MemGPT) documentation](https://docs.letta.com/concepts/memgpt/)
- [MemGPT: Towards LLMs as Operating Systems (arXiv 2310.08560)](https://arxiv.org/abs/2310.08560)
- [Anthropic Memory Tool documentation](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- [Anthropic Claude Sonnet 4.6 Skills announcement](https://www.anthropic.com/news/claude-sonnet-4-6)
- [Claude Dreaming: scheduled memory consolidation](https://www.mindstudio.ai/blog/what-is-claude-dreaming-anthropic-agent-memory)
- [Zep: Temporal Knowledge Graph Architecture (arXiv 2501.13956)](https://arxiv.org/abs/2501.13956)
- [Graphiti GitHub](https://github.com/getzep/graphiti)
- [LangMem documentation](https://langchain-ai.github.io/langmem/)
- [OpenAI Memory and new controls for ChatGPT](https://openai.com/index/memory-and-new-controls-for-chatgpt/)
- [Cognition: Rebuilding Devin for Claude Sonnet 4.6](https://cognition.ai/blog/devin-sonnet-4-5-lessons-and-challenges)

<a id="research-2023-2026"></a>
### 研究（2023-2026）
- [Generative Agents: Interactive Simulacra of Human Behavior (Park et al. 2023)](https://arxiv.org/abs/2304.03442)
- [Reflexion: Language Agents with Verbal Reinforcement Learning (Shinn et al. 2023)](https://arxiv.org/abs/2303.11366)
- [HippoRAG: Neurobiologically Inspired Long-Term Memory (Gutierrez et al. 2024)](https://arxiv.org/abs/2405.14831)
- [A-MEM: Agentic Memory for LLM Agents (Xu et al. NeurIPS 2025)](https://arxiv.org/abs/2502.12110)
- [Multi-Layered Memory Architectures (arXiv 2603.29194, March 2026)](https://arxiv.org/abs/2603.29194)
- [Memp: Exploring Agent Procedural Memory (Aug 2025)](https://arxiv.org/html/2508.06433v2)
- [LEGOMem: Modular Procedural Memory for Multi-agent LLM (Oct 2025)](https://arxiv.org/pdf/2510.04851)
- [Rethinking Memory Mechanisms of Foundation Agents (Feb 2026 survey)](https://arxiv.org/abs/2602.06052)
- [Position: Episodic Memory is the Missing Piece (arXiv 2502.06975)](https://arxiv.org/pdf/2502.06975)

<a id="safety-poisoning-and-hallucinations"></a>
### 安全、污染與幻覺
- [HaluMem: Operation-Level Memory Hallucination Benchmark (Nov 2025)](https://arxiv.org/abs/2511.03506)
- [MINJA Memory Injection Attack (NeurIPS 2025)](https://openreview.net/forum?id=QVX6hcJ2um)
- [MemoryGraft Persistent Memory Compromise (Dec 2025)](https://arxiv.org/html/2512.16962v1)
- [Palo Alto Unit 42: Indirect prompt injection poisons AI long-term memory](https://unit42.paloaltonetworks.com/indirect-prompt-injection-poisons-ai-longterm-memory/)
- [Multi-tenant AI Infrastructure: 5 Isolation Layers](https://medium.com/@isuruig/multi-tenant-ai-infrastructure-the-5-isolation-layers-that-determine-whether-your-customers-data-stays-separate-340aaeef4922)
- [The Day-30 Problem: agent memory drift](https://cipherbuilds.ai/blog/day-30-agent-memory-problem)

<a id="infrastructure"></a>
### 基礎設施
- [NVIDIA TTT-E2E: Reimagining LLM Memory (May 2026)](https://developer.nvidia.com/blog/reimagining-llm-memory-using-context-as-training-data-unlocks-models-that-learn-at-test-time/)
- [vLLM PagedAttention](https://docs.vllm.ai/en/latest/design/paged_attention/)
- [Anthropic Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

---

*下一篇：[Planning and Decomposition](06-planning-and-decomposition.md)*

<a id="ai-job-market-trends---may-2026"></a>
# AI 就業市場趨勢 - 2026 年 5 月

> **最後驗證：2026 年 5 月 17 日。** 本章整理的是 AI 招聘市場此刻真正發生的事——企業刊登的職稱、篩選的技能、薪酬區間，以及你會遇到的面試形式。資料來源涵蓋 2026 年 4–5 月的 100+ 公開職缺、招聘報告與 recruiter 訊號。

本章適合規劃下一步的工程師、建立評估標準的 hiring manager，以及進行組織設計決策的 engineering leader。它可與 [TRANSITION_GUIDE.md](../TRANSITION_GUIDE.md)（如何轉職到 AI 職位）及 [Question Bank](01-question-bank.md)（該準備什麼）搭配閱讀。

---

<a id="table-of-contents"></a>
## 目錄

- [三個最重要的市場變化](#the-three-headline-shifts)
- [2026 年的職位分類](#role-taxonomy-in-2026)
- [不同職涯層級所需技能](#skills-by-career-level)
- [職缺實際要求什麼](#what-job-listings-actually-require)
- [薪酬現實](#compensation-reality)
- [地理與產業分布](#geographic--industry-distribution)
- [面試流程模式](#interview-process-patterns)
- [值得關注的新興職位](#emerging-roles-to-watch)
- [策略重點](#strategic-takeaways)
- [參考資料](#references)

---

<a id="the-three-headline-shifts"></a>
## 三個最重要的市場變化

如果你只讀這一章的一部分，請先牢記以下三點。

<a id="1-the-market-is-paradoxically-hot-and-cold"></a>
### 1. 市場同時很熱，也很冷。

2026 年 Q1 出現約 52,050 個科技業裁員名額（Oracle 30K、Amazon、Meta、Dell），是自 2023 年以來裁員最多的第一季（[Kore1](https://www.kore1.com/tech-layoffs-2026/); [Tom's Hardware](https://www.tomshardware.com/tech-industry/tech-industry-lays-off-nearly-80-000-employees-in-the-first-quarter-of-2026-almost-50-percent-of-affected-positions-cut-due-to-ai)）。與此同時，AI 職缺 QoQ 成長 +8.9%、YoY 成長 +4.8%，且約有 275,000 個 AI 職位仍未補滿（[Allwork.space](https://allwork.space/2026/05/ai-hiring-is-rising-even-as-tech-layoffs-surge-140/)）。初階／入門工程師受衝擊最深——例行 codegen、QA 測試、基礎 frontend 工作被裁得最兇。資深與專精型 AI 職位則仍具韌性。

**含義：** 2026 年的「科技業招聘」與「AI 招聘」不是同一個故事。如果你的職涯仍以通用型中階 SWE 為主，你感受到的是冷市場；如果你是專精 AI 的資深人才，你處在賣方市場。

<a id="2-the-title-is-collapsing-the-work-is-fragmenting"></a>
### 2. 職稱正在收斂，但工作內容正在分化。

現在大多數公司都以「AI Engineer」作為總稱，但實際進入角色後，很快就會再細分成 RAG、agents、evals、fine-tuning 或 platform work。[「多數 AI 職稱將在接下來 18 個月收斂為 'AI Engineer'；只有 frontier labs 仍會保留帶有光環的職稱」](https://www.ivanturkovic.com/2026/04/24/ai-job-titles-2026-naming-chaos/)。獨立的「Prompt Engineer」職稱已幾乎從主要職缺網站消失——這項技能還在，但職稱本身已消失（[PE Collective](https://pecollective.com/blog/is-prompt-engineering-a-real-career/); [Medium - Prompt Engineering Is Dead 2026](https://medium.com/write-a-catalyst/prompt-engineering-is-dead-2026-ai-systems-engineering-7acdbbcb2160)）。

**含義：** 如果你現在還在招「Prompt Engineer」，你已經落後 18 個月。請定義真正的問題（eval 嚴謹度？agent debugging？customer-facing tuning？）並針對該特定角色招聘。

<a id="3-forward-deployed-engineer-is-the-breakout-role-of-2026"></a>
### 3. Forward Deployed Engineer 是 2026 年最亮眼的新職位。

FDE 在 2025 年中期於 frontier labs 還不是明確類別。到了 2026 年 5 月，OpenAI、Anthropic 與 Google 都已招聘數百人。Google／Box CEO 公開稱它為「科技業需求最旺的職位」（[Fast Company](https://www.fastcompany.com/91541878/google-box-ceos-say-this-is-the-most-in-demand-job-in-tech); [Hashnode FDE guide](https://hashnode.com/blog/a-complete-2026-guide-to-the-forward-deployed-engineer)）。中高階的 TC 已穩定在 $350-550K。

**含義：** Frontier AI 的買方（Fortune 500、政府、biotech）把現場工程支援視為合約交付的一部分。FDE 之所以存在，不是因為它是交付軟體最有效率的方法，而是因為買方願意為它付費。

---

<a id="role-taxonomy-in-2026"></a>
## 2026 年的職位分類

<a id="established-titles-still-hiring-strongly"></a>
### 已建立的職稱（仍然強力招聘中）

| Title | Description | Where it's posted |
|-------|-------------|-------------------|
| **AI Engineer** | 事實上的通用 AI 職稱。其他職稱正逐漸收斂到它之下。 | 普遍存在——大多數職缺 |
| **LLM Engineer** | 以 transformer fine-tuning、RAG、agents 為核心。與 ML Engineer 有所區別。 | 中大型公司；[iSmart LLM JD 2026](https://www.ismartrecruit.com/job-descriptions/llm-engineer) |
| **ML Engineer / ML+AI Software Engineer** | 經典的訓練與部署角色。 | [levels.fyi ML/AI focus](https://www.levels.fyi/t/software-engineer/focus/ml-ai) |
| **Applied AI Engineer** | 在 frontier labs 中嵌入客戶場景的變體。 | [Anthropic Applied AI](https://job-boards.greenhouse.io/anthropic/jobs/5116274008) |
| **Member of Technical Staff (MTS)** | 刻意模糊的職稱，淡化研究與工程之間的界線。 | OpenAI、Anthropic、Thinking Machines、Mistral（[Scout AI on MTS](https://scoutnow.ai/blog/rebirth-member-of-technical-staff)） |
| **AI Research Engineer / Research Scientist** | 僅見於 frontier labs；偏好 PhD。 | [Sundeep Teki - AI Research Eng 2026](https://www.sundeepteki.org/advice/the-ultimate-ai-research-engineer-interview-guide-cracking-openai-anthropic-google-deepmind-top-ai-labs) |
| **AI Solutions Architect** | 在 enterprise 領域需求很重。 | EY、Caterpillar、Deloitte（[EY listing](https://careers.ey.com/ey/job/Amsterdam-AI-Solution-Architect-1083-HP/1258705801/)） |
| **AI Platform Engineer** | 負責內部 LLM-ops platform。 | [Augment Code spec 2026](https://www.augmentcode.com/guides/ai-platform-engineering-leader-job-spec) |
| **AI Engineering Manager** | 單一角色中薪資最高；median $293.5K（[AI Pulse benchmarks](https://theaimarketpulse.com/salaries/)）。 | 在 scale-up 以上普遍存在 |
| **AI Product Manager** | 幾乎所有 B2B SaaS 都需要。 | 普遍存在（[Aakash Gupta](https://www.aakashg.com/product-manager-requirements/)） |
| **AI Technical Program Manager (TPM)** | 細分方向包括「Responsible AI TPM」「AI Infrastructure TPM」「GenAI Customer Performance TPM」 | Microsoft、AMD、Together AI |

<a id="new-titles-since-2025"></a>
### 2025 年以來的新職稱

| Title | Why it emerged | Where it's posted |
|-------|----------------|-------------------|
| **Forward Deployed Engineer (FDE)** | Frontier AI 買方要求現場工程支援作為交付內容。 | OpenAI、Anthropic、Google（[Anthropic FDE](https://job-boards.greenhouse.io/anthropic/jobs/4985877008)） |
| **AI Evaluation Engineer** | Eval 工作已成熟為獨立學科。 | OpenAI（[Applied Evals](https://openai.com/careers/software-engineer-applied-evals-san-francisco/)、[Frontier Evals](https://openai.com/careers/research-engineer-frontier-evals-and-environments-san-francisco/)）、Apple、Scale AI、Distyl、Apex |
| **Agentic Systems Engineer / AI Agent Engineer** | Agents 已成為獨立的 engineering surface。 | Teradata、GE Vernova、Deloitte、OpenAI（[Agent Infrastructure SWE](https://openai.com/careers/software-engineer-agent-infrastructure-san-francisco/)） |
| **AI Reliability Engineer** | Production AI 需要類似 SRE 的紀律；與傳統 SRE 不同。 | Anthropic（[Staff/Sr AI Reliability](https://www.anthropic.com/jobs)）；AI SRE 正由 Resolve.ai、Rootly 等定義成類別。 |
| **AI Security Engineer / LLM Red Team Specialist** | Prompt-injection 防禦與 jailbreak 研究已成為獨立學科。 | Life360（[Principal AI Security Engineer](https://www.remoterocketship.com/us/company/life360/jobs/principal-ai-security-engineer-ai-native-platform-united-states-remote/)）；[Practical DevSecOps](https://www.practical-devsecops.com/emerging-ai-security-roles/) 列出 10 個新興 AI security 職位。 |
| **MCP Engineer / MCP Software Engineer** | MCP 的採用讓 server development 成為專門領域。 | Descope（[MCP SWE](https://careers.descope.com/p/fe57f6224769-mcp-model-context-protocol-software-engineer)） |
| **AI Operator / Computer-Use Specialist** | 與 OpenAI Operator 和 Claude Cowork 相關。 | $75-120K 的 specialist tier（[Coasty](https://coasty.ai/blog/best-computer-use-platform-2026-20260402)） |

<a id="roles-disappearing-or-consolidating"></a>
### 正在消失或整併的角色

- **Prompt Engineer（獨立職稱）：** 這個職稱正在消失，但相關技能已成為基本門檻。
- **Distillation Engineer：** 更常以 fine-tuning／inference engineer 職缺中的*職責*出現，而非獨立大量招聘的職稱。

---

<a id="skills-by-career-level"></a>
## 不同職涯層級所需技能

<a id="l4-l5-mid-level-ic-3-5-yrs"></a>
### L4-L5（中階 IC，3-5 年）

- Python production proficiency —— **71% 的 AI 職缺**都要求（[Second Talent](https://www.secondtalent.com/resources/most-in-demand-ai-engineering-skills-and-salary-ranges/)）
- 至少實作過一個主流 LLM provider SDK（OpenAI、Anthropic、Bedrock）與一個 orchestration framework——最常見的是 LangChain/LangGraph（占 34.3% 的 agentic AI 職缺；[Agentic Engineering Jobs](https://agentic-engineering-jobs.com/langchain-job-market-2026)）
- Vector DB 基礎：Pinecone、Weaviate、pgvector——特定工具經驗幾週內可補，概念理解更重要
- RAG：chunking、hybrid search、BM25、reranking、retrieval evals
- Containerization：Docker（15.4%）、Kubernetes（17.6%）
- Cloud：AWS（32.9%）、Azure（26%）

<a id="l6-l7-senior--staff"></a>
### L6-L7（Senior / Staff）

- 具備端到端交付 production LLM systems 的經驗——「有實際交付真實系統的產業經驗，比學術資歷更具訊號」
- 能處理 vector indexes、GPU memory、agent state 的 multi-tenant isolation
- Eval frameworks（LangSmith / Langfuse / Braintrust）；eval-gated CI/CD
- Fine-tuning / LoRA / QLoRA / RLHF
- 成本最佳化——token budget、model routing、caching
- 「應把 LLM、vector stores 與 RAG 視為標準 system design 的一部分，而非小眾專業」([Design Gurus](https://designgurus.substack.com/p/system-design-interviews-changed))

<a id="l8-principal--leadership-ic"></a>
### L8+（Principal / Leadership IC）

- 擁有 agent orchestration layers、model routing、支援全體工程團隊的 LLMOps platforms
- 為非決定性系統建立 runtime governance
- 為 SOC 2 / HIPAA / EU AI Act 合規做架構設計——在 AI Act 第 27 條下觸發 DPIA + FRIA
- 「定義技術願景並擴展工程團隊，比單純 coding 能力更重要」

<a id="manager-track-em--director"></a>
### Manager 軌（EM / Director）

- AI Engineering Manager median $293.5K —— 單一角色中最高薪（[AI Pulse](https://theaimarketpulse.com/salaries/)）
- 現在的 hiring rubric 更看重：「你能不能把這個人放進一個有 PM 和 junior engineer 的房間裡，讓他推進技術方向而不把事情搞亂」——7 位受訪 hiring managers 中有 5 位如此表示（[Design Gurus](https://designgurus.substack.com/p/system-design-interviews-changed)）
- 在 frontier labs，mission alignment 與 safety judgment 權重很高（[Anthropic EM guide](https://www.gethireready.com/interview-guides/engineering-manager-anthropic)）

<a id="pm-track-ai-pm--ai-tpm"></a>
### PM 軌（AI PM / AI TPM）

- 「AI 已是新的基本盤，不再只是加分技能」
- 4+ 年 PM 經驗，最好具 B2B SaaS 或 AI-driven product 背景
- 關鍵：符合「technical fluency + product rigor」門檻的資深 AI PM 候選人不到 4 分之 1（[Aakash Gupta](https://www.aakashg.com/product-manager-requirements/)）
- 「能展示可運作 prototype 的候選人，表現優於只能描述 prototype 的人」

---

<a id="what-job-listings-actually-require"></a>
## 職缺實際要求什麼

<a id="must-have-called-out-as-required-across-100-postings"></a>
### Must-Have（100+ 份職缺中反覆列為必備）

- Python production code，3+ 年
- LLM API integration（OpenAI / Anthropic / Bedrock）
- RAG pipeline 經驗，包括 vector DB、chunking、retrieval evals
- Production-grade observability 與 eval pipelines
- Cloud + Kubernetes + IaC
- Agent debugging / multi-step workflow tracing
- 對 security 敏感職位而言，prompt injection / jailbreak defense

<a id="nice-to-have-explicitly-listed-as-plus-or-bonus"></a>
### Nice-to-Have（明列為「plus」或「bonus」）

- 論文或 OSS 貢獻；對應用型職位來說，3-5 個可運作專案的作品集比一篇 paper 更有力
- CUDA / GPU-level optimization —— 在 NVIDIA／frontier labs 是 must-have，其他地方多半只是加分
- Distillation / model compression
- Distributed inference 經驗
- Java/C++ 用於 legacy enterprise integration
- 超越 RLHF 的 reinforcement learning

<a id="top-tech-stack-in-listings-may-2026"></a>
### 職缺中最常見的技術棧（2026 年 5 月）

依出現頻率排序：

1. **Python** —— 占所有 AI 職缺的 71%
2. **PyTorch / JAX** —— 在 frontier labs 幾乎是標配
3. **LangChain / LangGraph** —— 占 agentic 職缺的 34.3%，排名第一的 framework
4. **LlamaIndex** —— 在 38% 的 LangChain 職缺中共同出現
5. **AWS (32.9%) / Azure (26%) / GCP / Vertex / Bedrock**
6. **Kubernetes (17.6%) + Docker (15.4%)**
7. **Vector DBs:** Pinecone、Weaviate、Qdrant、Chroma、pgvector
8. **MCP (Model Context Protocol)** —— 在尖端團隊已是[「基本要求」](https://medium.com/@adnanmasood/the-rise-of-model-context-protocol-mcp-skills-5f0d6a1c3579)
9. **Observability:** LangSmith、Langfuse、Braintrust、Arize
10. **Inference engines:** vLLM、SGLang、TensorRT-LLM
11. **Terraform / Helm / Ray / Kubeflow / MLflow / Feast** —— 內部 platform stack
12. **Provider SDKs:** OpenAI Agents SDK、Claude SDK、Vercel AI SDK、Mastra、Pydantic AI

<a id="by-company-tier"></a>
### 依公司層級區分

- **Frontier labs**（Anthropic、OpenAI、xAI）：PyTorch/JAX、vLLM/custom inference、internal evals、MCP servers、CUDA/GPU-level optimization
- **Scale-ups**（Cursor、Harvey、Sierra、Decagon、Glean、Perplexity）：TypeScript + Python 混合、LangGraph / OpenAI Agents SDK、Pinecone/pgvector、LangSmith/Braintrust evals
- **Enterprises**（Deloitte、EY、Caterpillar、Citi）：Azure 比重高、Bedrock、LangChain、重視 governance/MLOps 與 on-prem 能力

<a id="non-technical-requirements"></a>
### 非技術要求

- **Communication / cross-functional collaboration** —— 對 senior+ 已是基本門檻
- **Customer-facing skills** —— 對 FDE 職位至關重要；Anthropic 明確要求 3+ 年「technical, customer-facing role」經驗
- **OSS contributions** —— Anthropic 明言：[「若你做過有趣的獨立研究、寫過有洞見的部落格文章，或對 open-source software 做出重大貢獻，請把它放在履歷最上方」](https://www.sundeepteki.org/advice/how-to-get-hired-at-openai-anthropic-and-google-deepmind-in-2026)
- **Publications** —— AI Research Engineer 必備；但 Anthropic 技術團隊只有約 50% 擁有 PhD
- **Mission alignment** —— Anthropic 會透過 Behavioral and Values round 明確篩選
- **Regulatory experience:** 對 enterprise 而言是 SOC 2 / HIPAA / FedRAMP；對 EU 營運則需要熟悉 EU AI Act（FRIA/DPIA）
- **Security clearance** —— Lockheed 與聯邦周邊職位必備

---

<a id="compensation-reality"></a>
## 薪酬現實

> 僅列出公開資料來源的區間。請用 [levels.fyi](https://www.levels.fyi/) 驗證最新數據。除非另有說明，所有數字皆為 USD。

| Tier / Company | Level | Total Comp |
|---|---|---|
| **Anthropic (SF)** | Senior SWE | $316K base / $563K TC |
| **Anthropic (SF)** | Lead SWE | $332K base / $785K TC |
| **OpenAI (SF)** | All SWE | $251K – $1.28M+ TC |
| **OpenAI (SF)** | L5 SWE | $336K base + $774K stock = $1.15M TC |
| **OpenAI MTS / Research Scientist** | - | $245K – $685K base |
| **Cursor (Anysphere)** | SWE | $850K – $1.28M TC |
| **Sierra** | SWE | $200K – $460K TC; median $450K |
| **Thinking Machines Lab** | All eng | $450K – $500K base (Q1 H-1B filings) |
| Google AI Engineer | L3-L6 | $183K – $583K TC; median $280K |
| Microsoft AI Engineer | All | $238K – $355K+ TC; median $282K |
| **US National AI Engineer** | Entry (0-2y) | $90-135K base / $110-160K TC |
| **US National AI Engineer** | Mid (3-5y) | $140-210K base / $170-260K TC |
| **US National AI Engineer** | Senior (6-9y) | $180-280K base / $220-350K+ TC |
| **US National AI Engineer** | Staff/Principal (10+y) | $250-400K+ base / $350-600K+ TC |
| RAG Engineer Senior | - | $195-290K base; $400K+ TC at frontier |
| LLM Fine-Tuning Specialist | - | $195K-$350K |
| AI Security Engineer | - | $152-210K |
| LLM Red Team Specialist | - | $160-230K |
| **AI Engineering Manager** | - | $293.5K median (highest-paying single role) |
| AI Product Manager | - | $141K – $250K (median $159K) |
| **Agentic AI Architect** | - | $260K – $420K base |
| London (Quant fund / FAANG) | Senior ML | £140-180K base; £200K+ TC |
| London (Google DeepMind) | Senior | £110-155K base + RSU |
| Berlin / Germany | Senior | €95-130K |
| **Bangalore (Top GCC / AI-first)** | Senior | ₹1-2 Cr TC |
| Bangalore (Fresh PhD / top MS) | Entry | ₹22-32 LPA |
| Singapore | Avg | S$221,200 |
| Singapore (Principal/Lead) | 10+y | S$323,505 |

**Sources:** [levels.fyi Anthropic](https://www.levels.fyi/companies/anthropic/salaries/software-engineer), [OpenAI](https://www.levels.fyi/companies/openai/salaries/software-engineer), [Cursor](https://www.levels.fyi/companies/cursor/salaries/software-engineer), [Sierra](https://www.levels.fyi/companies/sierra/salaries/software-engineer), [Pin AI Comp Guide 2026](https://www.pin.com/blog/ai-compensation-salary-guide/), [Kore1 salary guide](https://www.kore1.com/ai-engineer-salary-guide/), [AI Pulse benchmarks](https://theaimarketpulse.com/salaries/), [Career Check London 2026](https://www.careercheck.io/blog/ml-engineer-salary-london-2026), [Zen van Riel Europe](https://zenvanriel.com/job/ai-engineer-salary-europe/), [Scaler India](https://www.scaler.com/topics/ai-ml-engineer-salary-complete-guide/).

<a id="compensation-insight"></a>
### 薪酬洞察

Frontier lab 的 MTS 薪酬（Anthropic/OpenAI median 約 $600-795K）與 enterprise AI engineering（中階約 $170-260K）之間的差距達 **3-5 倍**。選擇公司層級時，務必看清楚這一點。

---

<a id="geographic--industry-distribution"></a>
## 地理與產業分布

- **集中度：** 65%+ 的 AI engineers 位於 SF + NYC
- **雙層市場：** Indeed Hiring Lab 指出，約 95% 的招聘公司**尚未**發布 AI 職缺——採用仍集中在最大型公司（[Indeed Hiring Lab Jan 2026](https://www.hiringlab.org/2026/01/16/ai-adoption-accelerating-still-concentrated-among-largest-firms/)）
- **Enterprise adoption：** 截至 2026 年 Q1，72% 的 enterprises 至少已有一個 AI workload 在 production 上線（[Medha Cloud](https://medhacloud.com/blog/ai-adoption-statistics-2026)）
- **Consulting boom：** BCG 報告指出，2025 年 144 億美元營收中的 25%（36 億美元）來自 AI consulting（[Metaintro BCG](https://www.metaintro.com/blog/bcg-25-percent-ai-revenue-consulting-jobs-2026)）
- **International hiring** 年增 82%；67% 的公司提供 relocation package
- **Remote-friendly：** LangChain 生態系 35.2% remote、48.4% hybrid、16.4% 完全 onsite
- **Indeed AI Tracker：** 2025 年 12 月，所有職缺中有 4.2% 提及 AI——在整體招聘疲弱下仍持續成長

---

<a id="interview-process-patterns"></a>
## 面試流程模式

2026 年 5 月 AI-native 公司的標準流程：

1. **Recruiter screen**（30 分鐘）—— culture/mission + comp + visa
2. **Technical phone screen**（60-90 分鐘）—— 實務 coding、production 風格
3. **Take-home**（48 小時 - 3 天）—— LangChain、Mistral、Eightfold 常見；會要求做一個小型 RAG/agent system。[「考的不是你能不能做出來，而是你怎麼做——code quality、evals、error handling」](https://github.com/alexeygrigorev/ai-engineering-field-guide/blob/main/interview/01-interview-process.md)
4. **Onsite/virtual loop**（4-6 小時）：coding round + AI system design + project deep dive + behavioral。[「純白板回合幾乎消失了，連 Google 的形式現在也更偏協作」](https://designgurus.substack.com/p/system-design-interviews-changed)
5. **Hiring manager / values round** —— Anthropic 明確設有此輪

<a id="ai-role-specifics"></a>
### AI 職位特有重點

- **System design rounds** 現在會期待你理解 LLM infra、GPU scheduling、vector stores、RAG、eval-gated CI、cost/latency tradeoffs
- **AI-assisted coding rounds** 在 Meta、Canva、Google、Microsoft、Sierra、Cursor 都已明確允許 AI 工具（Cursor、Copilot、Claude）——評估的是 prompt 能力與輸出驗證能力
- **Take-home transparency：** 加上一段「AI audit note」——你在哪些地方用了 AI、改了什麼、為什麼這樣改。透明比偷偷使用更好
- **Sierra：** 僅接受在 SF 或 NY 辦公室現場面試；採「Plan + Build + Present」兩小時 agent assessment，沒有演算法回合
- **Cursor：** 8 小時 take-home，使用他們自家產品、文件有限並附 Slack channel——評估的是 product sense、autonomy 與 system design
- **Anthropic：** 「聽起來像前一天晚上才寫出來的答案，是不好的訊號」

<a id="frontier-lab-specifics"></a>
### Frontier lab 特有重點

- **Anthropic：** 90 分鐘、4 個層次、難度逐步提高的 coding problem，測試你是否能寫出乾淨、模組化、可吸收新需求的程式碼。另有明確的 values round。
- **OpenAI：** 「Design the OpenAI Playground」——要求你設計 wireframes + API + DB schema 來處理 thread/message history；同時考慮 multi-tenant 的安全雲端 IDE。
- **Mistral（Paris）：** 5 輪流程，不提供 remote，且另設一輪「LLM theory」，涵蓋 transformer internals 與 alignment。

---

<a id="emerging-roles-to-watch"></a>
## 值得關注的新興職位

這些職位在 2026 年 5 月成長最快——若你在規劃未來 12 個月的職涯方向，值得押注。

<a id="forward-deployed-engineer-fde"></a>
### Forward Deployed Engineer（FDE）
- **Why:** Frontier AI 的買方（Fortune 500、政府、biotech）把現場工程支援視為合約交付的一部分
- **Comp:** Frontier labs 的中高階為 $350-550K
- **Skills:** RAG、fine-tuning、distillation、MCP、customer-facing communication，以及在客戶現場做 evals
- **Where:** OpenAI、Anthropic、Google、ElevenLabs、Cohere、Mistral、各類 scale-up

<a id="ai-evaluation-engineer"></a>
### AI Evaluation Engineer
- **Why:** Eval 已成熟為獨立學科；production 需要 eval-gated CI/CD
- **Comp:** contractor 為 $100-110/hr；frontier labs 全職為 $200-400K
- **Skills:** LLM-as-judge calibration、error analysis methodology、statistical correction、dataset curation、regression detection
- **Where:** OpenAI（Applied Evals、Frontier Evals）、Apple、Scale AI、Distyl、Apex

<a id="agentic-systems-engineer"></a>
### Agentic Systems Engineer
- **Why:** Multi-agent 與 tool use 已成為第一級系統工程問題
- **Comp:** 一般區間約 $84-250K；agentic AI architect 為 $260-420K
- **Skills:** LangGraph / multi-agent orchestration、MCP、A2A protocol、agent debugging、tool design、sandbox security
- **Where:** Teradata、GE Vernova、Deloitte、OpenAI（Agent Infrastructure）

<a id="ai-reliability-engineer"></a>
### AI Reliability Engineer
- **Why:** Production AI 對非決定性系統需要類 SRE 的紀律
- **Comp:** Frontier labs 的 senior 約 $250-400K（Anthropic 有 Staff/Sr 職缺）
- **Skills:** AI agents 的 incident response、runaway-loop containment、cost anomaly detection、multi-provider fallback
- **Where:** Anthropic；「AI SRE」類別正由 Resolve.ai、Rootly 等定義中

<a id="ai-security-engineer--llm-red-team-specialist"></a>
### AI Security Engineer / LLM Red Team Specialist
- **Why:** 在 2026 年 5 月 AI security 轉折點（Mythos disclosure、Daybreak、MDASH、首個真實世界 AI-built zero-day）後，prompt injection 與 jailbreak 研究成為獨立學科
- **Comp:** 依專長不同，約 $152-230K
- **Skills:** Indirect prompt injection defense、jailbreak research、constitutional classifiers、model supply-chain trust、MCP threat modeling
- **Where:** Life360、frontier labs、重視 security 的 enterprises

<a id="mcp-engineer"></a>
### MCP Engineer
- **Why:** MCP 生態成熟，讓 server development 成為獨立專業
- **Skills:** MCP server design（HTTP/STDIO）、OAuth resource server pattern、agent-card signing、MCP security
- **Where:** Descope、與 Anthropic 對齊的 scale-ups，以及 Fortune 500 的內部平台

---

<a id="strategic-takeaways"></a>
## 策略重點

對於規劃下一步的 **engineers**：

1. **把自己定位成專家，而不是「Prompt Engineer」。** 選一個領域（evals、agents、RAG、FDE、MLOps）並做深。
2. **能運作的作品集 > paper。** 交付 3-5 個具備 evals 與 observability 的 production-grade 專案。對應用型職位而言，Anthropic、OpenAI 與 scale-ups 都比 publications 更看重這個。
3. **FDE 是高槓桿角色。** 如果你能同時具備技術深度與 customer-facing communication，在 frontier labs 中，除了創辦人／獨角獸的 staff equity 外，FDE 幾乎是市場頂端薪酬。
4. **市場正在分裂。** 通用型中階 SWE 工作正在被砍。資深 AI specialists 則處於賣方市場。請據此規劃你的路徑。

對於建立 rubric 的 **hiring managers**：

1. **針對具體問題招人，而不是只寫「AI Engineer」。** 如果你寫的是通用 AI Engineer JD，你收到的也會是通用型候選人。
2. **先評估實際交付過的系統。** 能模擬你真實工作負載的 take-home（例如：為你的領域做一個小型 RAG agent），比演算法謎題更有預測力。
3. **AI-assisted coding rounds 已成標準。** 觀察候選人如何 prompt + validate 模型輸出，比禁止使用 AI 更有資訊量。
4. **Comp banding 很重要。** Frontier labs 的薪酬正在對低兩個層級的公司造成留才壓力。若你是 enterprise 在招 AI 人才，請至少用本地市場水準再加上 senior+ 約 15-25% 的 AI premium 來校準。

對於做組織設計的 **engineering leaders**：

1. **讓角色對應工作，而不是對應頭銜。** 「AI Engineer」是你的總稱；其內部要明確命名專業化方向（RAG lead、agent lead、eval lead、platform lead）。
2. **Eval Engineer 是真正獨立的角色。** 不要讓 feature engineer 同時擁有自己想改善的 metric。測量與交付應分離。
3. **FDE 只有在客戶 ARR 高於約 $500K 時才划算。** 低於此門檻，請用 solutions engineering；高於此門檻，FDE 才能透過客戶專屬工程真正賺回薪資。
4. **AI Reliability Engineer 是你還沒意識到自己需要的角色。** 當你的第一個 agent 凌晨 3 點開始無限循環，在 loop guard 觸發前燒掉 $50K API 成本時，你會希望 6 個月前就有這個角色。

---

<a id="references"></a>
## 參考資料

本章資料來自截至 2026 年 5 月 17 日的 100+ 公開職缺、招聘報告與 recruiter 訊號。核心來源如下：

<a id="hiring-market-reports"></a>
### 招聘市場報告
- [Ivan Turkovic - AI Job Titles 2026: A CTO's Guide to the Naming Chaos](https://www.ivanturkovic.com/2026/04/24/ai-job-titles-2026-naming-chaos/)
- [Kore1 - AI Engineer Salary Guide 2026](https://www.kore1.com/ai-engineer-salary-guide/)
- [Kore1 - Tech Layoffs Q1 2026](https://www.kore1.com/tech-layoffs-2026/)
- [Pin - AI Compensation Benchmarks 2026](https://www.pin.com/blog/ai-compensation-salary-guide/)
- [Allwork.space - AI Hiring Rising vs Layoffs](https://allwork.space/2026/05/ai-hiring-is-rising-even-as-tech-layoffs-surge-140/)
- [Tom's Hardware - Q1 2026 Layoffs](https://www.tomshardware.com/tech-industry/tech-industry-lays-off-nearly-80-000-employees-in-the-first-quarter-of-2026-almost-50-percent-of-affected-positions-cut-due-to-ai)
- [Indeed Hiring Lab - Jan 2026 AI in Postings](https://www.hiringlab.org/2026/01/22/january-labor-market-update-jobs-mentioning-ai-are-growing-amid-broader-hiring-weakness/)
- [Indeed Hiring Lab - AI Adoption Concentration](https://www.hiringlab.org/2026/01/16/ai-adoption-accelerating-still-concentrated-among-largest-firms/)
- [Second Talent - Top 10 In-Demand AI Engineering Skills](https://www.secondtalent.com/resources/most-in-demand-ai-engineering-skills-and-salary-ranges/)
- [World Economic Forum - AI Added 1.3M Jobs](https://www.weforum.org/stories/2026/01/ai-has-already-added-1-3-million-new-jobs-according-to-linkedin-data/)
- [AI Pulse - AI & ML Engineer Salary Benchmarks 2026](https://theaimarketpulse.com/salaries/)
- [Agentic Engineering Jobs - LangChain Market 2026](https://agentic-engineering-jobs.com/langchain-job-market-2026)

<a id="compensation-data"></a>
### 薪酬資料
- [levels.fyi - Anthropic](https://www.levels.fyi/companies/anthropic/salaries/software-engineer)
- [levels.fyi - OpenAI](https://www.levels.fyi/companies/openai/salaries/software-engineer)
- [levels.fyi - Cursor](https://www.levels.fyi/companies/cursor/salaries/software-engineer)
- [levels.fyi - Sierra](https://www.levels.fyi/companies/sierra/salaries/software-engineer)
- [levels.fyi - Google AI](https://www.levels.fyi/companies/google/salaries/software-engineer/title/ai-engineer)
- [levels.fyi - Microsoft AI](https://www.levels.fyi/companies/microsoft/salaries/software-engineer/title/ai-engineer)
- [Entrepreneur - OpenAI Salaries (Federal Filing)](https://www.entrepreneur.com/business-news/how-much-openai-employees-make-salaries-685000)
- [Career Check - ML Engineer Salary London 2026](https://www.careercheck.io/blog/ml-engineer-salary-london-2026)
- [Zen van Riel - AI Engineer Salary Europe](https://zenvanriel.com/job/ai-engineer-salary-europe/)
- [Scaler - AI/ML Engineer Salary India](https://www.scaler.com/topics/ai-ml-engineer-salary-complete-guide/)
- [Morgan McKinley - Singapore AI/ML Engineer](https://www.morganmckinley.com/sg/salary-guide/data/ai-ml-engineer/singapore)

<a id="frontier-lab-career-sources"></a>
### Frontier lab 職涯資料來源
- [Anthropic - Careers](https://www.anthropic.com/careers)
- [Anthropic - Forward Deployed Engineer](https://job-boards.greenhouse.io/anthropic/jobs/4985877008)
- [Anthropic - Applied AI Engineer](https://job-boards.greenhouse.io/anthropic/jobs/5116274008)
- [OpenAI Careers](https://openai.com/careers/search/)
- [OpenAI - Applied Evals](https://openai.com/careers/software-engineer-applied-evals-san-francisco/)
- [OpenAI - Frontier Evals & Environments](https://openai.com/careers/research-engineer-frontier-evals-and-environments-san-francisco/)
- [OpenAI - Agent Infrastructure SWE](https://openai.com/careers/software-engineer-agent-infrastructure-san-francisco/)
- [Sundeep Teki - How to Get Hired at OpenAI/Anthropic/DeepMind 2026](https://www.sundeepteki.org/advice/how-to-get-hired-at-openai-anthropic-and-google-deepmind-in-2026)
- [Sundeep Teki - AI Research Engineer Interview Guide](https://www.sundeepteki.org/advice/the-ultimate-ai-research-engineer-interview-guide-cracking-openai-anthropic-google-deepmind-top-ai-labs)
- [Sundeep Teki - FDE Interviews](https://www.sundeepteki.org/advice/the-definitive-guide-to-forward-deployed-engineer-interviews-in-2026)
- [Hashnode - Complete 2026 Guide to FDE](https://hashnode.com/blog/a-complete-2026-guide-to-the-forward-deployed-engineer)

<a id="interview-process-sources"></a>
### 面試流程資料來源
- [Design Gurus - System Design Interviews Changed in 2026](https://designgurus.substack.com/p/system-design-interviews-changed)
- [IGotAnOffer - Anthropic Interview Process](https://igotanoffer.com/en/advice/anthropic-interview-process)
- [Jobright - Anthropic Technical Interview 2026](https://jobright.ai/blog/anthropic-technical-interview-questions-complete-guide-2026/)
- [Sierra - The AI-Native Interview](https://sierra.ai/blog/the-ai-native-interview)
- [Alexey Grigorev - AI Engineering Field Guide (Interview Process)](https://github.com/alexeygrigorev/ai-engineering-field-guide/blob/main/interview/01-interview-process.md)
- [interviewing.io - Meta AI-Assisted Coding Interview](https://interviewing.io/blog/how-to-use-ai-in-meta-s-ai-assisted-coding-interview-with-real-prompts-and-examples)
- [Exponent - OpenAI System Design 2026](https://www.tryexponent.com/blog/openai-system-design-interview)
- [Exponent - Anthropic System Design 2026](https://www.tryexponent.com/blog/anthropic-system-design-interview)

<a id="emerging-role-coverage"></a>
### 新興角色報導
- [AI Career Lab - Agentic-AI Job Guide 2026](https://theaicareerlab.com/blog/agentic-ai-jobs-guide-2026)
- [Practical DevSecOps - Top 10 Emerging AI Security Roles](https://www.practical-devsecops.com/emerging-ai-security-roles/)
- [Fast Company - Google/Box CEOs: FDE most in-demand](https://www.fastcompany.com/91541878/google-box-ceos-say-this-is-the-most-in-demand-job-in-tech)
- [Computerworld - FDE career emerging from AI shift](https://www.computerworld.com/article/4171867/heres-one-career-emerging-from-the-ai-shift-forward-deployed-engineers.html)
- [Rootly - AI SRE Guide 2026](https://rootly.com/ai-sre-guide)
- [Resolve.ai - What is an AI SRE](https://resolve.ai/glossary/what-is-ai-sre)
- [Medium - Rise of MCP Skills](https://medium.com/@adnanmasood/the-rise-of-model-context-protocol-mcp-skills-5f0d6a1c3579)

<a id="compliance--regulation"></a>
### 合規與法規
- [EU AI Act Implementation Timeline](https://artificialintelligenceact.eu/implementation-timeline/)
- [Secure Privacy - EU AI Act 2026 Compliance](https://secureprivacy.ai/blog/eu-ai-act-2026-compliance)
- [Augment Code - EU AI Act 2026 Guide](https://www.augmentcode.com/guides/eu-ai-act-2026)

---

*另見：[Question Bank](01-question-bank.md) | [Answer Frameworks](02-answer-frameworks.md) | [Behavioral for AI Roles](05-behavioral-for-ai-roles.md) | [Role Transition Guide](../TRANSITION_GUIDE.md)*

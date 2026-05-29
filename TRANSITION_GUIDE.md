<a id="transitioning-to-ai-engineering-roles"></a>
# 🔄 轉職到 AI 工程職位

> 為工程師、PM、QA 與管理者轉向 AI 聚焦職位所寫的具體角色指南。
> **不是泛泛而談的建議。每條路徑都對應到真實技能、真實 repo 章節，以及真實課程。**

---

<a id="who-this-guide-is-for"></a>
## 這份指南適合誰

如果你目前是軟體工程師、QA、PM、EM 或資料工程師，並想轉往 AI 相關職位，這份指南會把你**現有的技能**對應到特定 AI 職位，告訴你**具體還缺哪些能力**，並指出這個 repo 與哪些課程最適合補上這些缺口。

---

<a id="the-ai-role-landscape"></a>
## AI 職位版圖

在選擇路徑之前，先了解目標職位實際上是什麼：

```
┌─────────────────────────────────────────────────────────────────┐
│                    AI ROLE LANDSCAPE                            │
│                                                                 │
│  ┌──────────────────────┐    ┌──────────────────────────────┐  │
│  │   APPLICATION LAYER  │    │     INFRASTRUCTURE LAYER     │  │
│  │                      │    │                              │  │
│  │  LLM App Engineer    │    │  MLOps / AI Infra Engineer   │  │
│  │  AI Product Engineer │    │  AI Platform Engineer        │  │
│  │  Agentic Systems Eng │    │  AI Reliability Engineer     │  │
│  └──────────────────────┘    └──────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────┐    ┌──────────────────────────────┐  │
│  │    QUALITY LAYER     │    │     LEADERSHIP LAYER         │  │
│  │                      │    │                              │  │
│  │  AI Eval Engineer    │    │  AI Product Manager          │  │
│  │  AI Quality Engineer │    │  AI Engineering Manager      │  │
│  │  Red Team Analyst    │    │  AI Program Manager          │  │
│  └──────────────────────┘    └──────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────┐    ┌──────────────────────────────┐  │
│  │    RESEARCH LAYER    │    │     SPECIALIST LAYER         │  │
│  │                      │    │                              │  │
│  │  Applied AI Scientist│    │  Agentic Coding Specialist   │  │
│  │  Fine-tuning Engineer│    │  RAG Architect               │  │
│  │  Alignment Researcher│    │  AI Safety Engineer          │  │
│  └──────────────────────┘    └──────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

<a id="transition-paths-by-current-role"></a>
## 依目前職務區分的轉職路徑

---

<a id="1--backend-engineer--ai-engineering"></a>
### 1. 🖥️ 後端工程師 → AI 工程

**為什麼後端是最好的起點：** 你已經理解 API、延遲、資料庫、分散式系統與正式環境可靠性。AI 應用全都需要這些能力。你的落差主要是領域知識，而不是工程基本功。

#### 目標職位

```
Backend Engineer
      │
      ├──► LLM Application Engineer       (most common transition, 3–6 months)
      ├──► Agentic Systems Engineer        (3–9 months)
      ├──► AI Infrastructure / MLOps Eng  (6–12 months, needs GPU/serving knowledge)
      └──► RAG Architect                  (4–8 months)
```

#### 技能缺口分析

| 你已具備 | 需要補上的缺口 | 優先級 |
|-----------------|--------------|----------|
| REST API 設計 | LLM API 整合模式 | 🔴 高 |
| 資料庫設計 | 向量資料庫（Qdrant、Pinecone、Weaviate） | 🔴 高 |
| 非同步 / 串流 | 串流式 LLM 回應、token streaming | 🔴 高 |
| Auth 與 multi-tenancy | 多租戶 RAG 隔離 | 🔴 高 |
| 快取（Redis、CDN） | Prompt caching、semantic caching | 🟡 中 |
| 監控（Prometheus） | LLM observability（traces、evals） | 🟡 中 |
| CI/CD | LLMOps pipeline、模型版本管理 | 🟡 中 |
| N/A | Embedding 模型與向量數學 | 🟡 中 |
| N/A | Prompt engineering 基礎 | 🟡 中 |
| N/A | RAG pipeline 架構 | 🟡 中 |
| N/A | Agent framework（LangGraph、CrewAI） | 🟢 較低 |
| N/A | Fine-tuning 概念（LoRA、RLHF） | 🟢 較低 |

#### 你的 90 天計畫

**第 1 個月：LLM 整合**
- 學習 OpenAI / Anthropic API（streaming、function calling、structured output）
- 建一個簡單的 RAG 系統：PDF ingestion → Qdrant → LLM response
- 閱讀這個 repo：[01-foundations](01-foundations/)、[02-model-landscape](02-model-landscape/)、[05-prompting-and-context](05-prompting-and-context/)
- 課程：*ChatGPT Prompt Engineering for Developers*（DeepLearning.AI，免費）

**第 2 個月：正式環境模式**
- 為你的 RAG 系統加入 multi-tenant isolation
- 加入 LangSmith 或 Langfuse tracing  
- 實作 prompt caching 來節省成本
- 閱讀這個 repo：[06-retrieval-systems](06-retrieval-systems/)、[12-security-and-access](12-security-and-access/)、[08-memory-and-state](08-memory-and-state/)
- 課程：*Building and Evaluating Advanced RAG*（DeepLearning.AI，免費）

**第 3 個月：Agentic Systems**
- 用工具（web search、code execution）建一個 LangGraph agent
- 用 RAGAS 或 Phoenix 加入 evaluation pipeline
- 在具備成本控制與 rate limiting 的前提下部署
- 閱讀這個 repo：[07-agentic-systems](07-agentic-systems/)、[09-frameworks-and-tools](09-frameworks-and-tools/)、[14-evaluation-and-observability](14-evaluation-and-observability/)
- 課程：*AI Agents in LangGraph*（DeepLearning.AI，免費）

#### 作品集專案點子
- 具備存取控制的多租戶文件問答服務
- 會在 GitHub PR 發表評論的 agentic code reviewer
- 帶有 eval pipeline 的 RAG 驅動內部知識庫

---

<a id="2--frontend-engineer--ai-product-engineering"></a>
### 2. 🎨 前端工程師 → AI 產品工程

**為什麼這條轉職路徑可行：** 前端工程師理解 UX、即時 UI 更新與使用者行為。AI 產品的成敗往往取決於 UX——串流回應、漸進式渲染、loading state、回饋蒐集。你的技能比你想像中更有價值。

#### 目標職位

```
Frontend Engineer
      │
      ├──► AI Product Engineer         (3–6 months — highest demand)
      ├──► AI UX Engineer              (3–6 months, UX focus)
      └──► Full-Stack LLM Engineer     (6–9 months, add backend LLM skills)
```

#### 技能缺口分析

| 你已具備 | 需要補上的缺口 | 優先級 |
|-----------------|--------------|----------|
| 串流 UI（SSE、WebSocket） | LLM token streaming 整合 | 🔴 高 |
| 狀態管理 | 對話狀態、session memory | 🔴 高 |
| 使用者回饋模式 | AI 回饋蒐集（thumbs、ratings） | 🔴 高 |
| 表單驗證 | Prompt 輸入驗證與 sanitization | 🟡 中 |
| 非同步錯誤處理 | LLM timeout、fallback、retry 模式 | 🟡 中 |
| A/B testing | LLM A/B 測試與 variant tracking | 🟡 中 |
| N/A | 基本 prompt engineering | 🟡 中 |
| N/A | LLM API 整合（至少一個 provider） | 🟡 中 |
| N/A | 理解 context window | 🟡 中 |
| N/A | 基本 RAG 概念（是什麼、為什麼要用） | 🟢 較低 |

#### 你的 90 天計畫

**第 1 個月：把 LLM 整合進 UI**
- 建立一個串流聊天介面（Next.js + Vercel AI SDK）
- 實作完善的 loading states、逐 token 渲染與 error boundaries
- 加入回饋元件（讚 / 倒讚、重新生成按鈕）
- 課程：*ChatGPT Prompt Engineering for Developers*（DeepLearning.AI，免費）

**第 2 個月：AI 的 UX 模式**
- 用 session state 實作對話記憶
- 為 RAG 回應加入 citation rendering
- 為你的團隊做一個 prompt playground UI
- 閱讀這個 repo：[08-memory-and-state](08-memory-and-state/)、[05-prompting-and-context](05-prompting-and-context/)

**第 3 個月：整合評估**
- 為 UI 加上回饋訊號蒐集
- 將回饋接到 Langfuse 或 LangSmith 專案
- 在兩個 prompt variant 之間跑一個基本 A/B 測試
- 閱讀這個 repo：[14-evaluation-and-observability](14-evaluation-and-observability/)
- 課程：*Evaluating and Debugging Generative AI*（DeepLearning.AI + W&B，免費）

#### 作品集專案點子
- 支援 AI 建議與行內引用的串流文件編輯器
- 具持久化上下文的多步驟 AI 表單精靈
- 顯示各功能品質指標的 AI 回饋儀表板

---

<a id="3--qa-engineer--ai-eval-engineer"></a>
### 3. 🧪 QA 工程師 → AI Eval 工程師

**為什麼 QA 是最被低估的路徑：** AI evaluation 本質上就是一種新的 QA。手動測試案例設計、邊界情境思維、避免回歸——這些正是 AI 系統最需要的能力。差別在於工具不同，而且你需要調整對非決定性輸出的思維方式。

#### 目標職位

```
QA Engineer
      │
      ├──► AI Eval Engineer            (3–6 months — best fit, fast transition)
      ├──► AI Quality Engineer         (3–6 months)
      └──► Red Team Analyst            (6–9 months, security focus)
```

#### 技能缺口分析

| 你已具備 | 需要補上的缺口 | 優先級 |
|-----------------|--------------|----------|
| 測試案例設計 | Eval dataset 建立（dimensional sampling） | 🔴 高 |
| Regression testing 思維 | 以 eval suite 作為 CI 品質關卡 | 🔴 高 |
| Bug 回報 | Error analysis 方法論（open/axial coding） | 🔴 高 |
| 測試自動化 | LLM-as-judge evaluator 自動化 | 🔴 高 |
| 非功能測試 | Hallucination、bias、toxicity 偵測 | �� 中 |
| User acceptance testing | Human annotation workflow | 🟡 中 |
| N/A | Tracing 與 observability 設定 | 🟡 中 |
| N/A | RAGAS 指標（faithfulness、relevance、recall） | 🟡 中 |
| N/A | 基本 prompt engineering | 🟢 較低 |
| N/A | 用於 eval pipeline 的 Python scripting | 🟢 較低 |

#### 你的 90 天計畫

**第 1 個月：Error Analysis 基礎**
- 在任何 LLM 應用上（你自己的或 open source 的都行）設好 Langfuse 或 Phoenix tracing
- 做 3 輪手動 error analysis：每輪檢視 50 筆 traces、寫筆記、分類
- 閱讀 repo 中的 evals 配套指南：
  - [AI Evals: Comprehensive Study Guide](ai_evals_comprehensive_study_guide.md)
  - [AI Evals: LangWatch + Langfuse Guide](ai_evals_complete_guide_langwatch_langfuse.md)
- 必讀課程章節：*Error Analysis: The Secret Sauce*（收錄於 evals 指南第 3 章）

**第 2 個月：打造 Evaluator**
- 寫 3 個 code-based evaluator（JSON schema 檢查、格式驗證器、regex-based）
- 寫 1 個具備 Train/Dev/Test calibration 的 LLM-as-judge evaluator
- 導入 `judgy` 做統計偏誤校正
- 閱讀這個 repo：[14-evaluation-and-observability](14-evaluation-and-observability/)
- 課程：*Quality and Safety for LLM Applications*（DeepLearning.AI + WhyLabs，免費）

**第 3 個月：CI/CD 整合**
- 把 evaluator 接進 GitHub Actions workflow——每個 PR 都跑 eval
- 定義品質門檻（faithfulness > 0.85、format pass rate > 0.99）
- 建立每週 eval report dashboard
- 課程：*Evals for AI*（Maven，Hamel + Shreya——付費，但很值得拿來做職涯轉換）

#### 作品集專案點子
- 為公開 LLM 應用打造 open-source eval suite
- 部落格文章：〈我是如何用 QA 方法論抓出 LLM 失敗模式〉
- 結合 LangSmith + GitHub Actions 的 eval pipeline 範本 repo

---

<a id="4--product-manager--ai-product-manager"></a>
### 4. 📋 產品經理 → AI 產品經理

**為什麼 PM 有獨特優勢：** AI 產品失敗通常不是因為模型不好，而是因為產品決策錯了（問題定義錯、eval criteria 錯、success metrics 錯）。真正理解 AI failure modes 的 PM 非常稀缺，因此也非常搶手。

#### 目標職位

```
Product Manager
      │
      ├──► AI Product Manager           (3–6 months — direct analog)
      ├──► AI Program Manager           (3–6 months, coordination focus)
      └──► Head of AI Product           (9–18 months, leadership path)
```

#### 技能缺口分析

| 你已具備 | 需要補上的缺口 | 優先級 |
|-----------------|--------------|----------|
| 使用者研究 | 把 error analysis 當成 voice-of-customer | 🔴 高 |
| 成功指標定義 | AI 專屬指標（faithfulness、completion rate） | 🔴 高 |
| 路線圖優先排序 | 根據 eval data 排序 failure modes | 🔴 高 |
| A/B testing | LLM A/B 測試設計（prompt variants、models） | 🔴 高 |
| 利害關係人溝通 | 向合作方說明 AI 限制 | 🟡 中 |
| PRD 撰寫 | AI 系統能力文件與限制文件 | 🟡 中 |
| N/A | 高層次理解 LLM 如何運作（不需寫程式） | 🟡 中 |
| N/A | RAG pipeline 概念 | 🟡 中 |
| N/A | Tracing / observability 工具（Langfuse UI） | 🟡 中 |
| N/A | Prompt engineering 基礎 | 🟢 較低 |

#### 你的 90 天計畫

**第 1 個月：建立技術詞彙**
- 閱讀這個 repo 的基礎章節，而且**不要**跳過直接看程式碼：
  - [01-foundations](01-foundations/)——從概念上理解 transformer
  - [02-model-landscape](02-model-landscape/)——了解有哪些模型以及成本如何
  - [GLOSSARY.md](GLOSSARY.md)——學會這套詞彙
- 課程：*AI for Everyone*（Coursera，Andrew Ng，免費）——專為非技術職位設計

**第 2 個月：親自主導 Error Analysis**
- 請工程團隊架設 Langfuse 或 LangSmith
- 親自檢視產品中 100+ 筆 traces——做筆記、找出模式
- 與團隊進行 error analysis session，由你來帶 failure mode 分類
- 閱讀這個 repo：[14-evaluation-and-observability](14-evaluation-and-observability/)
- 閱讀：[AI Evals Comprehensive Study Guide](ai_evals_comprehensive_study_guide.md) 第 3 章（Error Analysis）

**第 3 個月：定義你的 Eval 策略**
- 為你的產品撰寫一份「AI Quality Spec」：定義每個功能何謂做好
- 與工程師合作，為這些標準加上 eval instrumentation
- 為下一季設定包含 AI quality gates 的成功指標（不只看使用者成長）
- 課程：*Evals for AI*（Maven，Hamel + Shreya——明確為 PM 設計）

#### 讓你成為突出 AI PM 的技能
- 你親自看過 traces（大多數 PM 會把這件事委派出去）
- 你能用量化方式，而不只是定性方式，定義 failure modes
- 你能說清楚品質改善的成本（prompt 調整 vs. 模型升級 vs. fine-tuning）
- 你理解 RAG、fine-tuning 與 prompt engineering 的差異，也知道各自適合什麼情境

---

<a id="5--engineering-manager--ai-engineering-manager"></a>
### 5. 👨‍💼 工程經理 → AI 工程經理

**EM 的轉變在於領導力升級：** AI 技術素養是必要條件，但還不夠。真正的關鍵轉變，是管理非決定性系統、管理在沒有 ground truth 下仍要評估品質的團隊，以及面對一個每 3–6 個月就快速改變的領域。

#### 目標職位

```
Engineering Manager
      │
      ├──► AI Engineering Manager       (6–12 months)
      ├──► Director of AI Engineering   (12–24 months)
      └──► VP of AI / Head of AI        (18–36 months)
```

#### 成為 AI EM 之後有什麼不同

| 傳統 EM | AI EM 新增的責任 |
|----------------|-----------------|
| Sprint planning | 由 eval 驅動的迭代循環 |
| PR review 標準 | 以 eval suite 作為新的「tests pass」門檻 |
| 招募 backend/frontend | 招募具備 LLM、vector search、evals 專長的人才 |
| 對故障做 incident response | 對品質回退做 incident response |
| 以 feature flags 規劃 roadmap | 以模型升級風險規劃 roadmap |
| 以交付表現做績效評估 | 績效評估需納入 AI 品質 ownership |

#### 你的 90 天計畫

**第 1 個月：技術深度**
- 閱讀整個 [09-frameworks-and-tools](09-frameworks-and-tools/)，理解工具版圖
- 閱讀 [09-claude-code.md](09-frameworks-and-tools/09-claude-code.md) 與 [10-opencoderguide.md](09-frameworks-and-tools/10-opencoderguide.md)——你會管理使用這些工具的團隊
- 理解成本：閱讀 [02-model-landscape/03-pricing-and-costs.md](02-model-landscape/03-pricing-and-costs.md)
- 課程：*Generative AI with LLMs*（Coursera，DeepLearning.AI）——提供足夠深度讓你主導技術討論

**第 2 個月：流程與團隊設計**
- 重新定義團隊的「done」，把 eval gates 納入其中
- 建立 eval 文化：每週 trace review、在 retros 中納入品質指標
- 定義你的 AI incident runbook：當 hallucination rate 飆升時該怎麼辦？
- 閱讀這個 repo：[13-reliability-and-safety](13-reliability-and-safety/)、[14-evaluation-and-observability](14-evaluation-and-observability/)

**第 3 個月：策略與招募**
- 為團隊定義 AI skills matrix：誰會什麼、還缺什麼
- 建立 AI 工程師面試 rubric（可用 [00-interview-prep](00-interview-prep/) 作為來源）
- 為下一季設定團隊層級的 AI quality OKRs
- 課程：*CS294 LLM Agents*（Berkeley，免費）——讓你有足夠深度參與策略討論

---

<a id="6--devops--platform-engineer--mlops--ai-infrastructure-engineer"></a>
### 6. 🛠️ DevOps / 平台工程師 → MLOps / AI 基礎設施工程師

**為什麼平台工程師在這裡會表現出色：** Kubernetes、CI/CD、observability、成本管理、SLA——這些你都做過。AI 特有的新加成是 GPU scheduling、model serving 與 LLMOps pipeline。

#### 目標職位

```
DevOps / Platform Engineer
      │
      ├──► MLOps Engineer               (3–6 months)
      ├──► AI Infrastructure Engineer   (6–9 months)
      └──► AI Platform Engineer         (9–12 months)
```

#### 技能缺口分析

| 你已具備 | 需要補上的缺口 | 優先級 |
|-----------------|--------------|----------|
| Container orchestration（K8s） | GPU node pools、NVIDIA device plugins | 🔴 高 |
| CI/CD pipelines | LLMOps pipeline（model eval、deployment gates） | 🔴 高 |
| Observability stacks | LLM 專屬指標（token throughput、TTFT） | 🔴 高 |
| 成本管理 | GPU 成本最佳化、training 用 spot instances | 🔴 高 |
| Secret management | 多個 LLM provider 的 API key rotation | 🟡 中 |
| N/A | 自架模型服務的 vLLM / TGI | 🟡 中 |
| N/A | Model versioning 與 registry | 🟡 中 |
| N/A | Quantization 基礎（GPTQ、AWQ、GGUF） | 🟡 中 |
| N/A | 為了理解你在服務什麼而需要的基本 prompt engineering | 🟢 較低 |

#### 你的 90 天計畫

**第 1 個月：LLM Serving**
- 在本機部署 vLLM，提供 Llama 3.3 7B 或 Qwen2.5-Coder 服務
- 加入 Prometheus metrics：tokens/sec、latency P50/P95/P99、queue depth
- 根據 request queue 設定 auto-scaling
- 閱讀這個 repo：[04-inference-optimization](04-inference-optimization/)、[11-infrastructure-and-mlops](11-infrastructure-and-mlops/)
- 課程：*Efficiently Serving LLMs*（DeepLearning.AI + Predibase，免費）

**第 2 個月：LLMOps Pipeline**
- 設好 LangSmith 或 Langfuse 來蒐集 traces
- 建立 CI/CD 品質關卡：模型部署前先跑 eval suite
- 實作 prompt version control（Langfuse prompt registry 或 DSPy）
- 閱讀這個 repo：[14-evaluation-and-observability](14-evaluation-and-observability/)

**第 3 個月：規模與成本**
- 比較在目標流量下，自架與 API 的成本（使用 repo 內的定價指南）
- 依模型、團隊、功能建立成本儀表板
- 實作平順的 multi-provider failover
- 課程：*ML Engineering for Production (MLOps)*（Coursera，DeepLearning.AI）

---

<a id="7--data-engineer--ai-data--feature-engineer"></a>
### 7. 📊 資料工程師 → AI 資料 / 特徵工程師

**為什麼資料工程師至關重要：** 訓練資料才是 AI 系統的競爭護城河。資料 pipeline、品質與新鮮度，往往比模型架構更能決定表現。你的技能可以立刻派上用場。

#### 目標職位

```
Data Engineer
      │
      ├──► AI Data Engineer             (2–4 months — fastest transition)
      ├──► Embedding Pipeline Engineer  (3–6 months)
      └──► Fine-tuning Data Specialist  (4–8 months)
```

#### 技能缺口分析

| 你已具備 | 需要補上的缺口 | 優先級 |
|-----------------|--------------|----------|
| ETL pipeline | 用於 RAG 的文件 ingestion pipeline | 🔴 高 |
| 資料品質檢查 | Eval dataset 品質驗證 | 🔴 高 |
| Schema 設計 | 向量資料庫的 metadata schema | 🔴 高 |
| Streaming pipeline | 即時 embedding 與索引更新 | 🟡 中 |
| N/A | Embedding 模型選擇與 batching | 🟡 中 |
| N/A | 向量資料庫操作（upsert、filter、ANN search） | 🟡 中 |
| N/A | 不同文件類型的 chunking 策略 | 🟡 中 |
| N/A | Fine-tuning annotation pipeline 設計 | 🟢 較低 |
| N/A | RLHF preference data format | 🟢 較低 |

#### 你的 90 天計畫

**第 1 個月：RAG 資料 Pipeline**
- 建立 ingestion pipeline：PDF/HTML/DOCX → chunked → embedded → Qdrant
- 加入資料品質關卡：最小 chunk 大小、去重、語言偵測
- 實作增量同步：只重新 embed 有變更的文件
- 閱讀這個 repo：[06-retrieval-systems/02-chunking-strategies.md](06-retrieval-systems/02-chunking-strategies.md)、[10-document-processing](10-document-processing/)

**第 2 個月：Eval Dataset 工程**
- 用 dimensional sampling 建立測試資料集（參考 evals 指南）
- 用 Label Studio 或 Argilla 建立 human annotation pipeline
- 追蹤 inter-annotator agreement，淘汰低品質標註
- 閱讀：[AI Evals Comprehensive Study Guide](ai_evals_comprehensive_study_guide.md) 第 12 章（Human Annotation）
- 課程：*Finetuning Large Language Models*（DeepLearning.AI，免費）

**第 3 個月：進階資料工程**
- 建一條能把 production traces 轉成 fine-tuning examples 的 pipeline
- 實作 embedding drift detection：當文件分布偏移時發出警示
- 在你的領域資料上 benchmark 3 個 embedding 模型
- 閱讀這個 repo：[03-training-and-adaptation](03-training-and-adaptation/)

---

<a id="role-comparison-overview"></a>
## 📊 職位比較總覽

```
Role            Months to   Avg Salary    Best Suited For
                First Role   (US, 2026)
────────────────────────────────────────────────────────
Backend         3–6 mo       $170–220K    LLM App / Agentic Engineering
Frontend        3–6 mo       $150–190K    AI Product / UX Engineering
QA              3–6 mo       $140–180K    AI Eval / Quality Engineering
PM              3–6 mo       $160–200K    AI Product Management
DevOps          3–6 mo       $170–220K    MLOps / AI Platform
Data Eng        2–4 mo       $165–210K    RAG Data, Fine-tuning Data
EM              6–12 mo      $200–280K    AI Engineering Manager
```

*薪資為依據 Levels.fyi 與 LinkedIn 資料整理的美國市場估算（2026 年 5 月）。實際範圍會因公司、地點與資歷而有很大差異。*

---

<a id="which-repo-sections-map-to-what"></a>
## 🗺️ Repo 各章節對應什麼主題

當你準備好深入某個方向時，可用這張表：

| 主題 | Repo 章節 | 原因 |
|-------|-------------|-----|
| LLM 如何運作 | [01-foundations](01-foundations/) | 其他一切的基礎 |
| 該用哪個模型 | [02-model-landscape](02-model-landscape/) | 模型選型是日常決策 |
| Fine-tuning | [03-training-and-adaptation](03-training-and-adaptation/) | 適合 fine-tuning data specialist 路線 |
| GPU serving / vLLM | [04-inference-optimization](04-inference-optimization/) | MLOps / 平台路線 |
| Prompt engineering | [05-prompting-and-context](05-prompting-and-context/) | 每個人都需要 |
| RAG pipeline | [06-retrieval-systems](06-retrieval-systems/) | 後端 / Data Eng 路線 |
| Agentic systems | [07-agentic-systems](07-agentic-systems/) | 後端 / 資深 AI 工程路線 |
| Memory & state | [08-memory-and-state](08-memory-and-state/) | 所有打造 agents 的工程師 |
| LangGraph、CrewAI、Claude Code | [09-frameworks-and-tools](09-frameworks-and-tools/) | 實務上的工具選型 |
| 文件解析 | [10-document-processing](10-document-processing/) | Data Eng / RAG 路線 |
| GPU infra、LLMOps | [11-infrastructure-and-mlops](11-infrastructure-and-mlops/) | DevOps / 平台路線 |
| 多租戶安全 | [12-security-and-access](12-security-and-access/) | 後端 / PM 路線 |
| Guardrails、red teaming | [13-reliability-and-safety](13-reliability-and-safety/) | QA / Red Team 路線 |
| RAGAS、LangSmith、evals | [14-evaluation-and-observability](14-evaluation-and-observability/) | QA / PM / 所有職位 |
| 設計模式 | [15-ai-design-patterns](15-ai-design-patterns/) | 資深等級準備 |
| 案例研究 | [16-case-studies](16-case-studies/) | 面試準備、參考設計 |
| Evals 深入指南 | [AI Evals Comprehensive Guide](ai_evals_comprehensive_study_guide.md) | QA / PM 路線 |
| AI Evals（LangWatch） | [AI Evals LangWatch Guide](ai_evals_complete_guide_langwatch_langfuse.md) | QA / Eval Eng 路線 |
| 面試準備 | [00-interview-prep](00-interview-prep/) | 所有職位 |
| 課程 | [COURSES.md](COURSES.md) | 所有職位 |

---

<a id="recommended-starter-courses-by-role"></a>
## 📚 依職務推薦的入門課程

> 完整細節請見 [COURSES.md](COURSES.md)

| 你的職務 | 第一門課 | 第二門課 | 第三門課 |
|-----------|-------------|---------------|--------------|
| **Backend** | ChatGPT Prompt Engineering for Devs（DL.AI，免費） | Building & Evaluating RAG（DL.AI，免費） | AI Agents in LangGraph（DL.AI，免費） |
| **Frontend** | ChatGPT Prompt Engineering for Devs（DL.AI，免費） | Building Systems with ChatGPT API（DL.AI，免費） | Evaluating & Debugging GenAI（DL.AI + W&B，免費） |
| **QA** | 本 repo 的 AI Evals Guide（免費） | Quality & Safety for LLM Apps（DL.AI，免費） | Evals for AI – Maven（Hamel + Shreya，付費） |
| **PM** | AI for Everyone（Coursera，免費） | AI Evals Guide 第 3 章（免費） | Evals for AI – Maven（Hamel + Shreya，付費） |
| **DevOps** | Efficiently Serving LLMs（DL.AI，免費） | Evaluating & Debugging GenAI（DL.AI + W&B，免費） | ML Engineering for Production（Coursera） |
| **Data Eng** | Building & Evaluating RAG（DL.AI，免費） | Finetuning LLMs（DL.AI，免費） | 本 repo 的 AI Evals Guide（免費） |
| **EM** | Generative AI with LLMs（Coursera） | AI Agents in LangGraph（DL.AI，免費） | CS294 LLM Agents（Berkeley，免費） |

*DL.AI = DeepLearning.AI*

---

<a id="common-mistakes-to-avoid"></a>
## 常見錯誤要避免

1. **跳過基礎知識** —— 在理解 embedding 是什麼之前就跳去用 LangChain，只會產出你無法除錯的 cargo-cult 程式碼。

2. **先做再評估** —— 沒有衡量品質的方法前，什麼都不要上線。在寫第一個 prompt 前就先定義好 eval criteria。

3. **抄 prompt 卻不理解原因** —— Prompt 是工程決策。要知道每個元素為什麼存在。

4. **太晚才關心成本** —— 每次 API 呼叫都有價格。從第一天起就建立成本追蹤。請參見 [02-model-landscape/03-pricing-and-costs.md](02-model-landscape/03-pricing-and-costs.md)。

5. **以為瓶頸一定在模型** —— 在大多數正式環境 AI 系統中，瓶頸通常是 retrieval 品質、prompt 設計或資料品質。模型很少是真正的問題。

6. **在正式環境用 model version string 的「latest」** —— 請固定到精確版本。靜默的模型更新會毀掉你的產品。

7. **過度 agent 化** —— 一開始就做 5-agent 系統，明明一個 prompt 寫得好的單次呼叫就夠用。先從簡單開始，只在真的需要時才增加複雜度。

---

<a id="how-to-get-hired"></a>
## 如何成功拿到工作

**公開打造作品。** AI 工程就業市場獎勵的是可被看見的實作成果：

1. **GitHub 作品集** —— 1 個完整打磨過的端到端專案，勝過 10 個玩具專案
2. **寫一篇部落格文章** —— 描述你真正解決的一個問題，以及你怎麼解（error analysis、eval pipeline、RAG latency 修復）
3. **貢獻 open source** —— OpenHands、LlamaIndex、DSPy、RAGAS。就算只是文件 PR，也會讓人注意到你。
4. **使用這個 repo 的面試準備** —— [00-interview-prep/01-question-bank.md](00-interview-prep/01-question-bank.md) 有 80 題附強答案

**面試時你該怎麼說：**
- 講出具體決策："I chose Qdrant over Pinecone because of X"（而不是 "I built a RAG system"）
- 說出你實際遇過哪些 failure modes，以及你怎麼修掉它們
- 至少把一個 benchmark 背熟（SWE-bench、RAGAS 分數、你 serving setup 的 TTFT）
- 展現你思考的不只有功能，還有 eval 與成本

---

*屬於 [AI System Design Guide](README.md) 的一部分——由 [ombharatiya](https://github.com/ombharatiya) 維護*

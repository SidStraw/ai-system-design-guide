<a id="recommended-ai-courses--learning-paths"></a>
# 🎓 推薦的 AI 課程與學習路徑

為 AI 工程師、ML 實務者與產品團隊整理的 **可靠、值得信賴且內容最新** 的線上課程清單。此處每門課程皆已於 **2026 年 5 月** 驗證，沒有灌水內容，也沒有過時的 MOOC。

---

<a id="table-of-contents"></a>
## 目錄

- [基礎：LLMs 與 Transformers](#foundation)
- [RAG 流程管線](#rag)
- [Agentic AI 與多代理系統](#agents)
- [上下文與記憶體管理](#context-memory)
- [AI 評估與可觀測性](#evals)
- [提示工程與上下文工程](#prompting)
- [微調與適配](#finetuning)
- [推論最佳化與 MLOps](#mlops)
- [AI 安全與 Guardrails](#safety)
- [Coding Agents 與開發者 AI 工具](#coding-agents)
- [給 PM 與非工程人員](#pm-track)
- [YouTube 頻道與免費內容](#free)
- [學習路徑建議](#paths)

---

<a id="foundation-llms--transformers"></a>
## 基礎：LLMs 與 Transformers <a name="foundation"></a>

| 課程 | 提供者 | 費用 | 為何值得信賴 |
|--------|----------|------|-----------------|
| **[Neural Networks: Zero to Hero](https://www.youtube.com/playlist?list=PLAqhIrjkxbuWI23v9cThsA9GvCAUhRvKZ)** | Andrej Karpathy (YouTube) | 免費 | 由 OpenAI／Tesla 傳奇人物打造、從零開始的權威系列。會從頭建出 GPT。 |
| **[CS324: Large Language Models](https://stanford-cs324.github.io/winter2022/)** | Stanford | 免費 | Stanford 等級的課程講義，涵蓋 LLM 基礎、scaling laws、alignment。 |
| **[Generative AI with LLMs](https://www.coursera.org/learn/generative-ai-with-llms)** | DeepLearning.AI + AWS (Coursera) | 約 $50 | LLM 的實作入門，涵蓋訓練、fine-tuning、RLHF。由 Andrew Ng 團隊製作。 |
| **[Practical Deep Learning for Coders](https://course.fast.ai/)** | fast.ai | 免費 | 自下而上、以程式為先的方法。最適合透過實作學習的工程師。 |

---

<a id="rag-pipelines"></a>
## RAG 流程管線 <a name="rag"></a>

| 課程 | 提供者 | 費用 | 為何值得信賴 |
|--------|----------|------|-----------------|
| **[Building and Evaluating Advanced RAG](https://www.deeplearning.ai/short-courses/building-evaluating-advanced-rag/)** | DeepLearning.AI + LlamaIndex | 免費 | 涵蓋 sentence-window retrieval、auto-merging，以及用 TruLens 進行 RAG 評估。 |
| **[Vector Databases: from Embeddings to Applications](https://www.deeplearning.ai/short-courses/vector-databases-embeddings-applications/)** | DeepLearning.AI + Weaviate | 免費 | 以實作方式帶你了解 embeddings、vector stores 與 hybrid search。 |
| **[Building RAG Agents with LLMs](https://courses.nvidia.com/courses/course-v1:DLI+S-FX-15+V1/)** | NVIDIA Deep Learning Institute | 免費 | 企業等級的 NVIDIA NIM RAG 課程。涵蓋 chunking、reranking、evaluation。 |
| **[LlamaIndex — Documentation: Learning](https://docs.llamaindex.ai/en/stable/understanding/)** | LlamaIndex | 免費 | 官方 LlamaIndex 學習路徑——想精通深度 RAG 流程管線的最佳資源。 |
| **[RAG Fundamentals (Haystack)](https://haystack.deepset.ai/tutorials)** | deepset / Haystack | 免費 | 使用 Haystack framework 進行 pipeline-based RAG 的實作教學。 |

---

<a id="agentic-ai--multi-agent-systems"></a>
## Agentic AI 與多代理系統 <a name="agents"></a>

| 課程 | 提供者 | 費用 | 為何值得信賴 |
|--------|----------|------|-----------------|
| **[AI Agents in LangGraph](https://www.deeplearning.ai/short-courses/ai-agents-in-langgraph/)** | DeepLearning.AI + LangChain | 免費 | 由 LangGraph 創作者親自講授。涵蓋 ReAct、persistence、human-in-the-loop、多代理。 |
| **[Multi AI Agent Systems with crewAI](https://www.deeplearning.ai/short-courses/multi-ai-agent-systems-with-crewai/)** | DeepLearning.AI + crewAI | 免費 | 官方 CrewAI 課程。涵蓋 Crews、Flows 與真實商業自動化案例。 |
| **[Building Agentic RAG with LlamaIndex](https://www.deeplearning.ai/short-courses/building-agentic-rag-with-llamaindex/)** | DeepLearning.AI + LlamaIndex | 免費 | 涵蓋 routing、tool-calling agents 與多文件的 agentic retrieval。 |
| **[Functions, Tools and Agents with LangChain](https://www.deeplearning.ai/short-courses/functions-tools-agents-langchain/)** | DeepLearning.AI + LangChain | 免費 | 介紹 tool-calling、OpenAI function calling，以及從零開始建構。 |
| **[Developing AI Agents using AutoGen](https://www.deeplearning.ai/short-courses/ai-agentic-design-patterns-with-autogen/)** | DeepLearning.AI + Microsoft | 免費 | AutoGen 多代理模式。涵蓋 debate、tool-use 與 code execution agents。 |
| **[CS294/194-196: LLM Agents (Berkeley)](https://rdi.berkeley.edu/llm-agents/f24)** | UC Berkeley | 免費 | 研究所等級的 LLM agents 課程。涵蓋 memory、planning、safety、evaluation。 |

---

<a id="context--memory-management"></a>
## 上下文與記憶體管理 <a name="context-memory"></a>

| 課程 | 提供者 | 費用 | 為何值得信賴 |
|--------|----------|------|-----------------|
| **[Building Systems with the ChatGPT API](https://www.deeplearning.ai/short-courses/building-systems-with-chatgpt/)** | DeepLearning.AI + OpenAI | 免費 | 涵蓋多輪對話狀態、上下文管理與 moderation chains。 |
| **[Prompt Engineering with Llama 2](https://www.deeplearning.ai/short-courses/prompt-engineering-with-llama-2/)** | DeepLearning.AI + Meta | 免費 | 展示 context window 的取捨，以及如何在 Llama 2 中管理 system prompt。 |
| **[Reasoning with o1](https://www.deeplearning.ai/short-courses/reasoning-with-o1/)** | DeepLearning.AI + OpenAI | 免費 | 深入介紹 o1 reasoning、budget tokens 與 thinking modes。可直接套用到 o3 與 Claude 3.7 Extended Thinking。 |
| **[Mem0 Documentation](https://docs.mem0.ai/)** | Mem0 (official) | 免費 | 生產環境代理中 multi-tier memory 的權威參考。 |

---

<a id="ai-evaluations--observability"></a>
## AI 評估與可觀測性 <a name="evals"></a>

| 課程 | 提供者 | 費用 | 為何值得信賴 |
|--------|----------|------|-----------------|
| **[Evals for AI: Maven Course](https://maven.com/hamel-shreya/evals-for-ai)** | Hamel Husain & Shreya Shankar (Maven) | 付費（約 $400） | 業界公認的 AI evals 黃金標準。已在數十家公司投入生產使用。我們 repo 中的 evals 指南也以此課程為基礎。 |
| **[Evaluating and Debugging Generative AI](https://www.deeplearning.ai/short-courses/evaluating-debugging-generative-ai/)** | DeepLearning.AI + W&B | 免費 | 涵蓋 tracing、以 W&B Weave 進行評估，以及 experiment tracking。 |
| **[Quality and Safety for LLM Applications](https://www.deeplearning.ai/short-courses/quality-safety-llm-applications/)** | DeepLearning.AI + WhyLabs | 免費 | 涵蓋 hallucination detection、toxicity、bias evaluation 與 drift monitoring。 |
| **[LangSmith Evaluation Tutorials](https://docs.smith.langchain.com/evaluation)** | LangChain | 免費 | 若你使用 LangChain 生態系，官方 LangSmith 文件就是最好的實作型 eval 參考。 |
| **[Phoenix + Langfuse official docs](https://docs.arize.com/phoenix)** | Arize Phoenix | 免費 | 使用 Phoenix 進行開源 evals 的實作教學。 |

> 📖 也請參考這個 repo 的配套指南：
> - [AI Evals：完整學習指南（Phoenix + Langfuse）](ai_evals_comprehensive_study_guide.md)
> - [AI Evals：LangWatch + Langfuse 指南](ai_evals_complete_guide_langwatch_langfuse.md)

---

<a id="prompt-engineering--context-engineering"></a>
## 提示工程與上下文工程 <a name="prompting"></a>

| 課程 | 提供者 | 費用 | 為何值得信賴 |
|--------|----------|------|-----------------|
| **[ChatGPT Prompt Engineering for Developers](https://www.deeplearning.ai/short-courses/chatgpt-prompt-engineering-for-developers/)** | DeepLearning.AI + OpenAI | 免費 | 提示工程的基礎課程。由 Isa Fulford 與 Andrew Ng 講授。 |
| **[Prompting Fundamentals (Anthropic)](https://www.anthropic.com/learn)** | Anthropic | 免費 | 直接來自 Claude 團隊。涵蓋 prompt design、XML tags、chain-of-thought。 |
| **[DSPy: Building Optimizable Pipelines](https://github.com/stanfordnlp/dspy)** | Stanford NLP (GitHub) | 免費 | 雖然不是正式課程，但 DSPy repo 中的 notebooks 是學習程式化 prompting 的最佳方式。 |
| **[Prompt Engineering Guide](https://www.promptingguide.ai/)** | DAIR.AI | 免費 | 由社群維護的完整參考資料，涵蓋各種主流 prompting 技術。 |

---

<a id="fine-tuning--adaptation"></a>
## 微調與適配 <a name="finetuning"></a>

| 課程 | 提供者 | 費用 | 為何值得信賴 |
|--------|----------|------|-----------------|
| **[Finetuning Large Language Models](https://www.deeplearning.ai/short-courses/finetuning-large-language-models/)** | DeepLearning.AI + Lamini | 免費 | 涵蓋 LoRA、完整 fine-tuning、資料集準備與 evaluation。精簡且實用。 |
| **[Reinforcement Learning from Human Feedback](https://www.deeplearning.ai/short-courses/reinforcement-learning-from-human-feedback/)** | DeepLearning.AI + AWS | 免費 | 深入介紹 RLHF：reward models、PPO、preference datasets。 |
| **[Hugging Face NLP Course](https://huggingface.co/learn/nlp-course/chapter1/1)** | Hugging Face | 免費 | 使用 HF 生態系（Trainer、PEFT 等）進行 transformer fine-tuning 的最佳免費課程。 |
| **[How Diffusion Models Work](https://www.deeplearning.ai/short-courses/how-diffusion-models-work/)** | DeepLearning.AI | 免費 | 適合影像模型 fine-tuning（stable diffusion、LoRA for images）。 |

---

<a id="inference-optimization--mlops"></a>
## 推論最佳化與 MLOps <a name="mlops"></a>

| 課程 | 提供者 | 費用 | 為何值得信賴 |
|--------|----------|------|-----------------|
| **[ML Engineering for Production (MLOps)](https://www.coursera.org/specializations/machine-learning-engineering-for-production-mlops)** | DeepLearning.AI (Coursera) | 付費 | 關於生產環境 ML 的 4 門課專項課程：deployment、monitoring、pipelines。 |
| **[Efficiently Serving LLMs](https://www.deeplearning.ai/short-courses/efficiently-serving-llms/)** | DeepLearning.AI + Predibase | 免費 | 涵蓋 vLLM、PagedAttention、quantization、LoRA serving。正是本指南所涵蓋的主題。 |
| **[vLLM Documentation & Tutorial](https://docs.vllm.ai/en/latest/)** | vLLM | 免費 | 官方 vLLM 文件是高吞吐 serving 最新、最完整的參考資料。 |

---

<a id="ai-safety--guardrails"></a>
## AI 安全與 Guardrails <a name="safety"></a>

| 課程 | 提供者 | 費用 | 為何值得信賴 |
|--------|----------|------|-----------------|
| **[Red Teaming LLM Applications](https://www.deeplearning.ai/short-courses/red-teaming-llm-applications/)** | DeepLearning.AI + Giskard | 免費 | 實作 red teaming、prompt injection、jailbreak detection 與 bias testing。 |
| **[AI Safety Fundamentals](https://aisafetyfundamentals.com/alignment-fast-track/)** | BlueDot Impact | 免費 | 最受信賴的免費 AI alignment 與 safety 課程，Anthropic、DeepMind 的專業人士也會使用。 |
| **[NVIDIA AI Red Team (NEMO Guardrails)](https://github.com/NVIDIA/NeMo-Guardrails)** | NVIDIA | 免費 | 使用 NeMo Guardrails 建立生產級 guardrails 的實作 notebooks。 |

---

<a id="coding-agents--developer-ai-tools"></a>
## Coding Agents 與開發者 AI 工具 <a name="coding-agents"></a>

| 課程 | 提供者 | 費用 | 為何值得信賴 |
|--------|----------|------|-----------------|
| **[Claude Code — Official Docs](https://docs.anthropic.com/en/home)** | Anthropic | 免費 | Claude Code 最權威的起點。涵蓋 CLAUDE.md、SDK 與 permissions。 |
| **[Building Code Agents (Hugging Face)](https://huggingface.co/learn/agents-course/unit1/introduction)** | Hugging Face | 免費 | Hugging Face 官方 agents 課程——包含建構可執行程式碼代理的單元。 |
| **[Introduction to OpenHands](https://github.com/All-Hands-AI/OpenHands/blob/main/docs/getting-started.md)** | All-Hands AI | 免費 | OpenHands 自主 coding agent 的官方入門指南。 |

> 📖 也請參考這個 repo 的指南：[Claude Code Guide](09-frameworks-and-tools/09-claude-code.md) 與 [OpenCoder Landscape](09-frameworks-and-tools/10-opencoderguide.md)

---

<a id="for-pms--non-engineers"></a>
## 給 PM 與非工程人員 <a name="pm-track"></a>

這些內容不需要 Python 經驗：

| 課程 | 提供者 | 費用 | 為何值得推薦 |
|--------|----------|------|--------------|
| **[AI for Everyone](https://www.coursera.org/learn/ai-for-everyone)** | DeepLearning.AI (Coursera) | 免費 | Andrew Ng 為非技術角色設計的課程。涵蓋 AI 能做／不能做什麼，以及專案領導。 |
| **[Prompt Engineering for Everyone](https://learnprompting.org/)** | Learn Prompting | 免費 | 為非工程人員撰寫、用白話文說明的 prompt engineering 指南。 |
| **[Evals for AI (Maven)](https://maven.com/hamel-shreya/evals-for-ai)** | Hamel Husain & Shreya Shankar | 付費 | 雖然課程中有程式碼，但它是為 PM 與 QA 設計，不只適合工程師。非常推薦。 |
| **[AI Product Management](https://www.productschool.com/blog/product-management/ai-product-manager/)** | Product School | 免費（部落格） | 提供 PM 建構 AI 驅動產品的實用指南。 |
| **[Google: Introduction to Generative AI](https://cloud.google.com/learn/training/machinelearning-ai)** | Google Cloud Skills Boost | 免費 | 以無程式碼方式介紹 generative AI、LLMs 與 responsible AI。 |

---

<a id="youtube-channels--free-content"></a>
## YouTube 頻道與免費內容 <a name="free"></a>

| 頻道 / 資源 | 焦點 | 為何值得追蹤 |
|-------------------|-------|------------|
| **[Andrej Karpathy](https://www.youtube.com/@AndrejKarpathy)** | 基礎、transformers | 最清楚解釋 LLM 實際運作方式的人之一 |
| **[Yannic Kilcher](https://www.youtube.com/@YannicKilcher)** | 論文解讀 | 清楚帶你走過最新 ML 研究論文 |
| **[Aleksa Gordić - The AI Epiphany](https://www.youtube.com/@TheAIEpiphany)** | 論文解讀 | 深入且技術導向的論文拆解 |
| **[AI Jason](https://www.youtube.com/@AIJasonZ)** | Agents、LangChain、實務應用 | 非常適合入門 agentic frameworks 的影片 |
| **[Sam Witteveen](https://www.youtube.com/@samwitteveenai)** | Gemini、RAG、agents | 最優秀的實戰型 AI YouTuber 之一 |
| **[Matt Wolfe](https://www.youtube.com/@mreflow)** | AI 新聞、產品展示 | 最適合追蹤最新 AI 新聞與工具 |
| **[Hamel Husain (blog)](https://hamel.dev/)** | Evals、生產環境 AI、LLMs | 由 evals maven 課程作者分享的真實生產經驗 |
| **[Simon Willison (blog)](https://simonwillison.net/)** | LLM 新聞、工具、coding | 最值得信賴的每日 AI 新聞來源 |
| **[The Latent Space podcast](https://www.latent.space/)** | 技術型 AI 訪談 | 最好的技術型 AI podcast——與研究者深入對談 |
| **[Lex Fridman Podcast](https://lexfridman.com/podcast/)** | 廣泛的 AI/ML 訪談 | 與頂尖 AI 研究者進行長篇訪談 |

---

<a id="learning-path-suggestions"></a>
## 學習路徑建議 <a name="paths"></a>

<a id="path-im-new-to-ai-and-want-to-build-things-fast"></a>
### 🛤️ 路徑：「我是 AI 新手，想快速做出作品」

```
第 1 週：Prompt Engineering for Developers (DeepLearning.AI) — 免費，2 小時
第 2 週：Building Systems with ChatGPT API (DeepLearning.AI) — 免費，2 小時
第 3 週：Building and Evaluating Advanced RAG (DeepLearning.AI) — 免費，2 小時
第 4 週：AI Agents in LangGraph (DeepLearning.AI) — 免費，4 小時
第 2 個月：挑一個真實專案，把這份指南當作參考
```

<a id="path-i-want-to-understand-llms-deeply"></a>
### 🛤️ 路徑：「我想深入理解 LLMs」

```
第 1-3 週：Neural Networks: Zero to Hero (Karpathy) — 免費，12+ 小時
第 4-6 週：CS324 Stanford LLMs — 免費，30+ 小時
第 2 個月：Generative AI with LLMs (Coursera DeepLearning.AI)
第 3 個月：CS294 LLM Agents (Berkeley)
```

<a id="path-i-want-to-build-production-ready-ai-evaluation"></a>
### 🛤️ 路徑：「我想建立可上線的 AI 評估能力」

```
第 1 週：Evaluating and Debugging Generative AI (DeepLearning.AI + W&B) — 免費
第 2 週：本 repo 的 evals 指南（Phoenix/Langfuse）— 免費  ← 從這裡開始
第 3-4 週：Quality and Safety for LLM Applications (DeepLearning.AI)
第 2 個月：Evals for AI (Maven, Hamel + Shreya) — 付費，但很值得
```

<a id="path-im-a-pm-learning-to-contribute-to-ai-product-quality"></a>
### 🛤️ 路徑：「我是 PM，想學習如何提升 AI 產品品質」

```
第 1 週：AI for Everyone (Coursera) — 免費
第 2 週：Prompt Engineering for Everyone (learnprompting.org) — 免費
第 3 週：本 repo 的 AI Evals 指南 — 免費（特別是第 1-3 章的錯誤分析）
第 2 個月：Evals for AI (Maven) — 付費，含 PM 路線
```

<a id="path-i-want-to-deploy-coding-agents-in-my-team"></a>
### 🛤️ 路徑：「我想在團隊中部署 coding agents」

```
第 1 天：Claude Code docs (anthropic.com) — 免費
第 1 週：本 repo 的 Claude Code Guide + OpenCoder Landscape Guide
第 2 週：Building Code Agents (Hugging Face) — 免費
第 1 個月：在 CI 中讓 Claude Code 跑一個真實專案
```

---

<a id="how-to-stay-current"></a>
## 如何保持跟上最新進展

AI 變化很快。除了課程之外，以下習慣能幫助你持續掌握最新動態：

1. **追蹤 Simon Willison 的部落格** — 每日更新、值得信賴的 AI 新聞摘要
2. **閱讀 Anthropic + OpenAI 的 release notes** — 第一手資料永遠勝過二手整理
3. **收聽 Latent Space podcast** — 最佳的技術深度
4. **參與 open source** — OpenHands、LlamaIndex、DSPy——真正的學習發生在 PR 中
5. **替這個 repo 按星** — 我們會隨著產業變化持續更新它 ⭐

---

*由 [Om Bharatiya](https://github.com/ombharatiya) 維護。歡迎透過 PR 新增更多課程！*

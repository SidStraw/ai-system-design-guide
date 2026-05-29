<a id="ai-system-design-glossary"></a>
# AI 系統設計詞彙表

本指南中關鍵術語的快速參考。

---

<a id="a"></a>
## A

**Agentic Coding** — LLM 自主編輯檔案、執行 shell 命令、撰寫測試並反覆迭代，直到完成一項程式設計任務。代表例子包括 Claude Code、OpenHands 與 Cline。

**Agentic System** — 使用工具自主規劃並執行多步驟任務的 LLM 應用程式。

**Attention Mechanism** — 讓模型能專注於輸入中相關部分的神經網路元件。Self-attention 會將每個 token 與其他所有 token 進行比較。

**ABAC (Attribute-Based Access Control)** — 根據使用者、資源與環境的屬性，而非固定角色，來進行存取控制。

---

<a id="b"></a>
## B

**Batching** — 將多個請求一併處理，以提升 GPU 使用率。Continuous batching 會在其他請求仍在生成時加入新請求。

**BM25** — 傳統的關鍵字排序演算法。常與向量搜尋結合，形成 hybrid retrieval。

**Budget Tokens** — Extended Thinking（Claude）或 reasoning（o3）的可設定運算預算。預算越高 → 內部推理步驟越多 → 準確率與成本越高。

---

<a id="c"></a>
## C

**Chain-of-Thought (CoT)** — 在最終答案前引導逐步推理的 prompting 技術。

**Chunking** — 將文件拆分成較小片段，以便進行 embedding 與 retrieval。常見策略包括 fixed-size、semantic 與 hierarchical。

**Claude Code** — Anthropic 的終端機原生自主程式設計代理。它使用 bash、text_editor 與 computer 工具，在整個專案中讀取、編輯並執行程式碼。透過 `CLAUDE.md` manifest 檔案控制。

**Cline** — 開源的 VS Code 擴充套件，透過工具使用提供自主 AI 程式設計能力（檔案編輯、終端機、瀏覽器）。原生支援 MCP。

**Computer-Use** — 一種模型能力（Claude 3.5+ 原生支援），可透過模擬滑鼠點擊、鍵盤輸入與截圖來控制 GUI。可用於瀏覽器與桌面自動化。

**Context7** — 一個 MCP server，可在執行時抓取最新的函式庫文件，解決程式設計代理的「訓練資料過時」問題。

**Context Window** — LLM 在單次請求中可處理的最大 token 數。範圍從 4K 到 1M+ tokens。

**Cosine Similarity** — 衡量兩個向量相似度的方法。是比較 embeddings 的標準指標。

**Cursor** — AI-native IDE（基於 VS Code 分支），具備深度模型整合，可進行程式碼補全、agentic 編輯與多檔案脈絡感知。

---

<a id="d"></a>
## D

**DPO (Direct Preference Optimization)** — 一種 fine-tuning 方法，可直接根據偏好資料最佳化，而不需要獨立的 reward model。

**DSPy** — 透過可最佳化模組而非手動 prompts 來程式化使用 LLM 的框架。

---

<a id="e"></a>
## E

**Embedding** — 文字的稠密向量表示。用於 semantic search 與相似度比較。

**Ensemble** — 結合多個模型輸出以提升可靠性。包含 voting、debate 與 mixture-of-agents。

**Extended Thinking** — Claude 的（3.7+）內部推理模式，模型會先進行 scratchpad 推理，再產生回應。可透過 `thinking.budget_tokens` 設定。預設不會顯示給終端使用者。

---

<a id="f"></a>
## F

**Few-Shot Prompting** — 在 prompt 中加入範例，以引導模型行為。

**Fine-Tuning** — 以任務特定資料對預先訓練模型進行訓練，以提升表現。

**Function Calling** — LLM 輸出結構化工具呼叫，而非純文字的能力。

---

<a id="g"></a>
## G

**Guardrails** — 用於防止有害或離題回應的輸入／輸出驗證機制。

**Grounding** — 將 LLM 回應連結到事實來源，以減少 hallucination。

**Grok 4.3** - xAI 的前沿推理模型。在推理基準上可與 GPT-5.5、Claude Opus 4.7 與 Gemini 3.1 Pro 競爭。可透過 xAI API 與 X 內部使用。

---

<a id="h"></a>
## H

**Hallucination** — 模型生成看似合理但事實上不正確的資訊。

**HNSW (Hierarchical Navigable Small World)** — 用於向量資料庫近似最近鄰搜尋的圖結構演算法。

**Human-in-the-Loop (HITL)** — 讓人類對 AI 輸出進行監督、核准或修正的模式。

---

<a id="i"></a>
## I

**In-Context Learning** — 模型根據 prompt 中的範例適應任務，而不需更新權重。

**Inference** — 執行已訓練模型以產生預測／輸出。

---

<a id="j"></a>
## J

**JSON Mode** — 可保證輸出為有效 JSON 結構的 LLM 輸出模式（舊版）。在較新的 API 中已被 **Structured Outputs** 取代。

---

<a id="k"></a>
## K

**KV Cache** — 來自 attention 計算的快取 key-value 配對。可實現高效率的 autoregressive 生成。

---

<a id="l"></a>
## L

**LangChain** — 用於建構 LLM 應用程式的框架，提供 chains、agents 與 integrations。

**LlamaIndex** — 專注於文件處理與 retrieval 的資料框架，供 LLM 應用程式使用。

**LiveCodeBench** — 在競技程式設計平台的真實世界問題上評估 coding models 的 benchmark。對生產環境程式設計任務而言，比 HumanEval 更可靠。

**LoRA (Low-Rank Adaptation)** — 一種 parameter-efficient fine-tuning 方法，訓練小型 adapter matrices，而非完整模型權重。

**LLM-as-Judge** — 使用一個 LLM 來評估另一個 LLM 的輸出。

---

<a id="m"></a>
## M

**MCP (Model Context Protocol)** - 用於與 LLM 進行標準化工具／資源整合的開放協定。由 Anthropic 於 2024 年 11 月推出；治理於 2025 年 12 月移交 Linux Foundation 的 Agentic AI Foundation；已被 Anthropic、OpenAI、Google、Microsoft、AWS 採用。Version 2.0（2026 年 3 月核准）新增 Streamable HTTP transport 與 OAuth 2.1 auth。

**Mixture of Agents (MoA)** — 一種 ensemble 模式，由多個 agents 共同產出綜合回應。

**Multi-Tenancy** — 在共享基礎設施上服務多個客戶，同時保持資料隔離。

---

<a id="o"></a>
## O

**o3** — OpenAI 的高運算推理模型（於 2025 年 1 月發布）。使用內部 chain-of-thought 分配測試時運算資源。提供標準版與「mini」版本。擅長數學、程式碼與科學。

**OCR (Optical Character Recognition)** — 從影像或掃描文件中擷取文字。

**OpenHands** — 開源的自主軟體工程代理（前身為 OpenDevin）。支援多種後端 LLM，並在 Docker sandbox 中執行。

---

<a id="p"></a>
## P

**Prompt Caching** — 針對重複的 prompt 前綴重用 KV cache。Anthropic（cache_control）、Google（implicit）以及部分 OpenAI endpoints 原生提供。對於長且固定的前綴，可降低 60–90% 成本。

**Prompt Injection** — 惡意輸入操控 LLM 行為的攻擊手法。

**Prefix Caching** — 在多個請求之間，針對共同的 prompt 前綴重用 KV cache。

---

<a id="q"></a>
## Q

**QLoRA** — 將 LoRA 與 4-bit quantization 結合，以實現記憶體效率更高的 fine-tuning。

**Quantization** — 降低模型精度（例如從 FP16 到 INT4），以減少記憶體用量並提升速度。

---

<a id="r"></a>
## R

**RAG (Retrieval-Augmented Generation)** — 一種會先擷取相關文件，為 LLM 生成提供脈絡的模式。

**RBAC (Role-Based Access Control)** — 根據具備預定義權限的使用者角色來進行存取控制。

**ReAct** — 在 Reasoning 與 Acting 步驟間交替進行的 agent 模式。

**Reranking** — 用於提升 retrieval 精準度的第二階段評分。Cross-encoders 的準確度高於 bi-encoders。

**RLHF (Reinforcement Learning from Human Feedback)** — 透過人類偏好來對齊模型行為的訓練方法。

---

<a id="s"></a>
## S

**Self-Consistency** — 取樣多條推理路徑，並選擇最常見答案的方法。

**Semantic Search** — 依據語意而非關鍵字尋找文件，使用 embeddings 進行搜尋。

**Speculative Decoding** — 先由小型 draft model 提議 tokens，再由大型模型驗證。

**Structured Outputs** — OpenAI（以及 Anthropic 的 tool-mode）提供的能力，可保證模型輸出符合指定的 JSON Schema。比舊版 JSON mode 更嚴格。

**SWE-bench Verified** — 自主軟體工程的業界標準 benchmark。衡量模型解決真實 GitHub issues 的能力。頂尖模型（Claude 3.7、o3）得分可達 50–70%+。

**System Prompt** — 為 LLM 對話設定脈絡與行為的指示。

---

<a id="t"></a>
## T

**Temperature** — 控制 LLM 輸出隨機性的參數。數值越低 = 越具決定性。

**Token** — 文字處理的基本單位。在英文中，大約相當於 0.75 個單字或 4 個字元。

**Tool Use** — LLM 呼叫外部函式／API 的能力。

**Transformer** — 以 self-attention 為基礎的神經網路架構，是現代 LLM 的基礎。

---

<a id="v"></a>
## V

**Vector Database** — 專為儲存與搜尋高維向量（embeddings）而最佳化的資料庫。

---

<a id="w"></a>
## W

**Windsurf** — AI-native IDE（由 Codeium 推出），具備緊密的 agentic 整合。使用「Flows」—— 可重現的 agentic 序列。是 Cursor 的替代方案。

---

<a id="z"></a>
## Z

**Zero-Shot** — 不提供範例、僅依賴模型既有知識的 prompting 方式。

---

*另請參閱：[PATTERNS.md](PATTERNS.md) 以取得設計模式快速參考*

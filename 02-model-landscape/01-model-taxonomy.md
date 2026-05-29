<a id="model-taxonomy"></a>
# 模型分類總覽

本章提供截至 **2026 年 5 月** 的模型版圖完整指南，涵蓋模型家族、能力，以及用於正式環境系統的選型標準。

> **最後驗證：2026 年 5 月 25 日。** 模型版圖變化非常快速。請務必再次核對供應商定價頁面與發行說明。
>
> **2026 年 5 月：自 4 月更新以來的新變化：** OpenAI GPT-5.5（4 月 23 日）與 GPT-5.5 Instant（5 月 5 日，成為 ChatGPT 預設）；Claude Opus 4.7（4 月 16 日，在 Bedrock／Vertex／Foundry 正式可用）；Claude Mythos Preview（限制存取；僅限 Project Glasswing 合作夥伴）；Google Gemma 4（4 月 2 日，Apache 2.0）與 Gemini 3.2 Flash（5 月 5 日低調推出）；DeepSeek V4 Pro 與 V4 Flash（4 月 24 日；V4 Pro 75% 折扣於 5 月 22 日改為 **永久**，6 月 1 日起新牌價為每 1M $0.435／$0.87）；Moonshot Kimi K2.6（4 月 20 日，1T MoE / 32B active）；Alibaba Qwen 3.6 Plus / 3.6-35B-A3B / 3.6 Max-Preview；Mistral Medium 3.5（4 月 29 日，整合 chat／reasoning／coding／vision）；Meta Muse Spark（4 月 8 日，Meta 首個 closed-weight 模型）；Llama 4 Behemoth 因能力疑慮，發布延後至 2026 年秋季之後。SWE-bench Verified 榜首：Claude Mythos Preview 93.9%；ARC-AGI-2 榜首：GPT-5.5 85.0%。

<a id="table-of-contents"></a>
## 目錄

- [模型類別](#model-categories)
- [前沿模型](#frontier-models)
- [推理模型](#reasoning-models)
- [開源模型](#open-source-models)
- [專門模型](#specialized-models)
- [Embedding 模型](#embedding-models)
- [模型選型框架與語意路由](#model-selection-framework)
- [主權 AI 與資料駐留](#sovereign-ai-and-data-residency)
- [能力比較](#capability-comparison)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="model-categories"></a>
## 模型類別

<a id="by-capability-level-april-2026-reality"></a>
### 依能力層級區分（2026 年 4 月現況）

| 層級 | 特徵 | 範例 | 使用情境 |
|------|------|------|----------|
| **Frontier** | 最先進的推理能力與 agentic 掌控力 | Claude Opus 4.6, GPT-5.4, Gemini 3.1 Pro, Grok 4 | 複雜推理、coding、正式環境 agents |
| **Fast/Efficient** | 低於 200ms、成本最佳化 | Gemini 3.1 Flash, GPT-5.4-mini, Claude Haiku 4.5 | 高流量串流、UI、即時場景 |
| **Battle-Tested** | 成熟、廣泛部署、穩定 | Claude Sonnet 4.6, GPT-4o, Gemini 2.5 Pro | 企業正式工作負載 |
| **Small/Edge** | 私有、邊緣、特化 | Llama 4 Scout, Mistral Small 4, Phi-4 | 本地隱私、裝置端、MoE 高效率 |
| **Reasoning-Heavy** | 延伸的內部 CoT | GPT-5.4 Pro, DeepSeek-R1, Claude Opus 4.6 (thinking) | 數學、程式除錯、多步邏輯 |

<a id="by-reasoning-mode-20252026"></a>
### 依推理模式區分（2025–2026）

| 模式 | 能力 | 模型 | 使用情境 |
|------|------|------|----------|
| **Standard** | 快速、直覺式回應 | GPT-5.4-mini, Claude Sonnet 4.6 | 對話、簡單擷取 |
| **Extended Thinking** | 輸出前先進行內部 scratchpad CoT | Claude Opus 4.6, GPT-5.4 Pro, DeepSeek-R1 | 數學、程式除錯、規劃 |
| **Hybrid** | 使用者可控制推理深度 | Claude Opus 4.6, GPT-5.4 | 複雜度可變的任務 |

---

<a id="frontier-models-aprilmay-2026"></a>
<a id="frontier-models"></a>
## 前沿模型（2026 年 4–5 月）

<a id="claude-opus-47-anthropic---may-2026-new"></a>
### Claude Opus 4.7（Anthropic）- 2026 年 5 月新版本

| 屬性 | 數值 |
|-----------|-------|
| Context Window | 1M tokens |
| Max Output | 128K tokens |
| Input Cost | $5.00 / 1M tokens（與 4.6 相同） |
| Output Cost | $25.00 / 1M tokens（與 4.6 相同） |
| Extended Thinking | 原生支援，Adaptive mode |
| Multimodal | Text + 更高解析度 Vision |
| SWE-bench Verified（Adaptive） | 87.6%（2026 年 5 月 13 日） |
| Released | 2026 年 4 月 16 日（API、Bedrock、Vertex、Microsoft Foundry 正式可用） |

**最適合：** 自主 coding agents（Claude Code 的核心）、多檔案重構、複雜推理。與 4.6 同價，對大多數工作負載來說是直接升級。
**注意事項：** 成本敏感的工作負載可使用 Sonnet 4.6；Opus 4.7 主要適合需要頂級 coding／agentic 品質的任務。

<a id="claude-mythos-preview-anthropic---restricted-access"></a>
### Claude Mythos Preview（Anthropic）- 限制存取

| 屬性 | 數值 |
|-----------|-------|
| Status | 未發布 - 僅限 Project Glasswing 合作夥伴（約 11 個組織：AWS、Apple、Cisco、Google、Microsoft、NVIDIA、Palo Alto 等） |
| Reason for restriction | 具雙重用途的資安能力 |
| SWE-bench Verified | 93.9%（2026 年 5 月 13 日 - 目前 SOTA） |
| Released | 2026 年 4 月 7 日（限制合作夥伴預覽） |

**最適合：** 目前不適合正式環境。之所以在此追蹤，是因為它創下 SWE-bench Verified 公開 SOTA，也反映了前沿內部能力所在位置。

<a id="claude-opus-46-anthropic"></a>
### Claude Opus 4.6（Anthropic）

| 屬性 | 數值 |
|-----------|-------|
| Context Window | 1M tokens |
| Max Output | 128K tokens |
| Input Cost | $5.00 / 1M tokens |
| Output Cost | $25.00 / 1M tokens |
| Extended Thinking | 原生 adaptive thinking（可設定 `budget_tokens`） |
| Multimodal | Text + Vision |
| Highlights | Anthropic 最強模型；coding 與推理表現卓越 |
| Released | 2026 年 2 月 |

**最適合：** 最複雜的推理、自主軟體工程、agentic 工作流。
**注意事項：** 屬於高價位；若任務不需要極致能力，可改用 Sonnet 4.6。

<a id="claude-sonnet-46-anthropic"></a>
### Claude Sonnet 4.6（Anthropic）

| 屬性 | 數值 |
|-----------|-------|
| Context Window | 1M tokens |
| Input Cost | $3.00 / 1M tokens |
| Output Cost | $15.00 / 1M tokens |
| Extended Thinking | 支援 |
| Multimodal | Text + Vision |
| Highlights | 可處理過去需要 Opus 等級的任務；成本／品質平衡最佳 |
| Released | 2026 年 2 月 |

**最適合：** 正式環境 coding agents（Claude Code 的核心）、大規模複雜推理。
**注意事項：** 現在以更低成本涵蓋大多數 Opus 級任務，是多數工作負載的強力預設選項。

<a id="gpt-54-openai"></a>
### GPT-5.4（OpenAI）

| 屬性 | 數值 |
|-----------|-------|
| Context Window | 272K tokens（standard）；另有 extended 版本 |
| Input Cost | $2.50 / 1M tokens |
| Output Cost | $15.00 / 1M tokens |
| Multimodal | Text、Vision、原生 computer use |
| Highlights | 內建 computer-use 能力；相較 GPT-5.2 事實錯誤少 33%；結合 coding 與 agentic 優勢 |
| Released | 2026 年 3 月 |

**最適合：** 具備 computer use 的 agentic 工作流、coding、專業工作任務。
**注意事項：** 272K+ tokens 的長上下文定價會加倍。

<a id="gpt-54-mini-openai"></a>
### GPT-5.4-mini（OpenAI）

| 屬性 | 數值 |
|-----------|-------|
| Context Window | 272K tokens |
| Input Cost | $0.75 / 1M tokens |
| Output Cost | $4.50 / 1M tokens |
| Highlights | GPT-5 等級高流量工作負載的最佳成本／效能選擇 |
| Released | 2026 年 3 月 |

**最適合：** 高流量 API 呼叫、成本最佳化推理、正式環境 chatbot。

<a id="gpt-54-pro-openai"></a>
### GPT-5.4 Pro（OpenAI）

| 屬性 | 數值 |
|-----------|-------|
| Context Window | 272K tokens |
| Input Cost | $30.00 / 1M tokens |
| Output Cost | $180.00 / 1M tokens |
| Highlights | 最強推理能力；為最困難任務提供的高階方案 |
| Released | 2026 年 3 月 |

**最適合：** 競賽級數學、複雜多步推理。
**注意事項：** 非常昂貴；大量工作負載應改用標準 GPT-5.4 或 mini。

<a id="gpt-55-openai---may-2026-new"></a>
### GPT-5.5（OpenAI）- 2026 年 5 月新版本

| 屬性 | 數值 |
|-----------|-------|
| Context Window | 1M tokens |
| Input Cost | $5.00 / 1M tokens |
| Output Cost | $30.00 / 1M tokens |
| Multimodal | Text、Image、Audio、Video |
| ARC-AGI-2 | 85.0%（2026 年 5 月 13 日 - 榜首） |
| Released | 2026 年 4 月 23 日 |

**最適合：** 最高品質的多模態工作負載；目前 ARC-AGI-2 榜首。定位為「面向真實工作的全新智慧等級」，在頂級推理＋多模態方面取代 GPT-5.4。
**注意事項：** 輸入成本約為 GPT-5.4 的 2 倍（$2.50 → $5.00），輸出也約 2 倍（$15 → $30）。若是對話型工作負載且此價格不合理，可改用 GPT-5.5 Instant。

<a id="gpt-55-instant-openai---may-2026-new"></a>
### GPT-5.5 Instant（OpenAI）- 2026 年 5 月新版本

| 屬性 | 數值 |
|-----------|-------|
| Status | 自 2026 年 5 月 5 日起成為 ChatGPT 與 API `chat-latest` 的預設 |
| Hallucination Reduction | 在高風險提示（醫療／法律／金融）上，比 GPT-5.3 Instant 少 52.5% 幻覺 |
| AIME 2025 | 81.2%（高於 GPT-5.3 Instant 的 65.4%） |
| Response Length | 比前代少約 30% 字數／行數 |
| Released | 2026 年 5 月 5 日 |

**最適合：** 等同 ChatGPT 預設的工作負載、即時對話，以及重視降低幻覺的高風險領域。
**注意事項：** 已取代 GPT-5.3 Instant 成為預設聊天模型。GPT-5.2-chat-latest 與 GPT-5.3-chat-latest 已於 2026 年 5 月 8 日棄用。

<a id="gpt-realtime-2-translate-whisper-openai---may-2026-new"></a>
### GPT-Realtime-2、Translate、Whisper（OpenAI）- 2026 年 5 月新版本

| 屬性 | 數值 |
|-----------|-------|
| Capability | 具備 GPT-5 級推理能力的即時語音 |
| Translate Coverage | 70+ 種輸入 → 13 種輸出語言 |
| Pricing | 每 1M audio tokens 為 $32 / $64（輸入／輸出） |
| Released | 2026 年 5 月 7 日 |

**最適合：** 即時語音 agents、多語翻譯、以語音為主的產品。Realtime API Beta 已於 2026 年 5 月 12 日移除，Realtime-2 是受支援的路徑。

<a id="gemini-31-pro-google"></a>
### Gemini 3.1 Pro（Google）

| 屬性 | 數值 |
|-----------|-------|
| Context Window | 1M tokens |
| Input Cost | $2.00 / 1M tokens（standard）；$4.00（200K+） |
| Output Cost | $12.00 / 1M tokens（standard）；$18.00（200K+） |
| Multimodal | 原生：Text、Vision、Audio、Video |
| Highlights | Google 最先進推理；具備強大的 agentic 與 coding 能力 |
| Released | 2026 年 2 月 |

**最適合：** 複雜推理、多模態分析、長上下文工作負載。
**注意事項：** 取代 Gemini 3 Pro Preview。Gemini 2.5 Pro／Flash 於 2026 年 6 月棄用。

<a id="gemini-31-flash-google"></a>
### Gemini 3.1 Flash（Google）

| 屬性 | 數值 |
|-----------|-------|
| Context Window | 1M tokens |
| Input Cost | $0.10 / 1M tokens |
| Output Cost | $3.00 / 1M tokens |
| Multimodal | 原生：Text、Vision、Audio、Video |
| Highlights | Google 最快模型；高流量場景下價格／效能最佳 |
| Released | 2026 年 3 月 |

**最適合：** 即時多模態應用、高流量管線、長上下文 RAG。

<a id="gemini-32-flash-google---may-2026-new"></a>
### Gemini 3.2 Flash（Google）- 2026 年 5 月新版本

| 屬性 | 數值 |
|-----------|-------|
| Status | 2026 年 5 月 5 日於 iOS Gemini app 與 Google AI Studio 低調推出（尚無正式公告） |
| Released | 2026 年 5 月 5 日 |

**最適合：** 很可能是 3.1 Flash 的後繼版本，用於高流量工作負載。請視為 preview；定價與完整能力仍待正式發布。

<a id="gemini-deep-research--deep-research-max-google---may-2026-new"></a>
### Gemini Deep Research / Deep Research Max（Google）- 2026 年 5 月新版本

| 屬性 | 數值 |
|-----------|-------|
| Built on | Gemini 3.1 Pro |
| Capabilities | 支援 MCP；原生圖表／資訊圖生成；延長 test-time compute；非同步背景工作流 |
| Released | 2026 年 4 月 21 日 |

**最適合：** 研究 agents、文件綜整、長時間執行的非同步工作流。其 MCP 支援讓它成為 Google 第一個具備一級工具整合能力的 research-agent 產品。

<a id="gemini-robotics-er-16-google-deepmind---may-2026-new"></a>
### Gemini Robotics-ER 1.6（Google DeepMind）- 2026 年 5 月新版本

| 屬性 | 數值 |
|-----------|-------|
| Domain | 實體機器人、具身推理 |
| New capability | 讀取儀表與視窗液位計 |
| Deployment | Boston Dynamics Spot |
| Released | 2026 年 4 月 14 日 |

**最適合：** 需要 vision-language grounding 來執行實體動作的機器人應用。可透過 Gemini API 與 AI Studio 使用。

<a id="grok-4-xai"></a>
### Grok 4（xAI）

| 屬性 | 數值 |
|-----------|-------|
| Context Window | 256K tokens |
| Input Cost | $3.00 / 1M tokens |
| Output Cost | $15.00 / 1M tokens |
| Highlights | 原生 tool use 與即時搜尋；推理能力具競爭力 |
| Released | 2025 年 7 月（Grok 4.20 beta：2026 年 2 月） |

**最適合：** 即時網路研究、重推理任務、X／web 即時整合。
**注意事項：** Grok 4.1 Fast 的價格為 $0.20／$0.50，適合高流量。

<a id="model-comparison-frontier-tier-may-2026"></a>
### 模型比較：前沿層級（2026 年 5 月）

| 模型 | 推理 | Coding | Context | Agentic | 成本 |
|-------|-----------|--------|---------|---------|------|
| Claude Mythos Preview（限制版） | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | n/a |
| Claude Opus 4.7 | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | $$$$ |
| GPT-5.5 | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | $$$$ |
| Claude Opus 4.6 | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | $$$$ |
| GPT-5.4 | ★★★★★ | ★★★★★ | ★★★★ | ★★★★★ | $$$ |
| Claude Sonnet 4.6 | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | $$$ |
| Gemini 3.1 Pro | ★★★★★ | ★★★★ | ★★★★★ | ★★★★ | $$ |
| Grok 4 | ★★★★ | ★★★★ | ★★★★ | ★★★★ | $$$ |
| GPT-5.4-mini | ★★★★ | ★★★★ | ★★★★ | ★★★ | $ |
| Gemini 3.1 Flash | ★★★ | ★★★ | ★★★★★ | ★★★ | $ |
| GPT-5.5 Instant | ★★★★ | ★★★★ | ★★★★ | ★★★★ | $$ |

<a id="production-heritage--maturity"></a>
### 正式環境血統與成熟度

雖然前沿模型在 benchmark 上領先，但許多企業系統仍依賴 **battle-tested** 模型：

| 模型家族 | 進入正式環境時間 | 成熟度說明 |
|--------------|------------------|---------------|
| **GPT-4o** | 2024 年 5 月 | 生態最成熟；延遲波動最低；rate limit 最高。 |
| **Claude 3.5 Sonnet / 3.7 Sonnet** | 2024 年 6 月 | tool-use 穩定性與結構化輸出的黃金標準。 |
| **Gemini 2.5 Pro** | 2025 年 3 月 | 已在大規模環境驗證；長上下文穩定。將於 2026 年 6 月由 3.x 取代。 |
| **o1 / o3** | 2024 年 9 月 | 推理模型失敗模式已被充分理解；o3 已取代 o1。 |

**為什麼要留在「較舊」的前沿模型？**
1. **一致性**：新模型在「發布窗口期」常有延遲尖峰與行為漂移。
2. **成本效率**：新版本發布後，上一代通常會便宜 50-80%。
3. **Guardrail 調校**：安全與審核層通常更成熟。

---

<a id="open-source-models"></a>
## 開源模型

<a id="llama-4-family-meta----new-april-2026"></a>
### Llama 4 家族（Meta）-- 2026 年 4 月新版本

| 模型 | 參數 | Context | 架構 | 備註 |
|-------|------------|---------|--------------|-------|
| Llama 4 Scout | 17B active / 16 experts (MoE) | 10M | Sparse MoE | 業界領先的 10M context；可放進單張 H100；勝過 Gemma 3、Gemini 2.0 Flash-Lite |
| Llama 4 Maverick | 17B active / 128 experts (MoE) | 1M | Sparse MoE | 勝過 GPT-4o 與 Gemini 2.0 Flash；以一半 active params 提供與 DeepSeek V3 相近的能力 |
| Llama 4 Behemoth | ~288B active（估計） | - | Dense MoE | 仍在訓練中；在 STEM benchmark 上優於 GPT-4.5、Gemini 2.0 Pro |

**優勢：**
- 第一代採用 Mixture-of-Experts 架構的 Llama
- 從底層即原生支援多模態（text、image、video input）
- 在 Hugging Face 開放權重；可透過 Meta AI 於 WhatsApp、Messenger、Instagram 使用
- Scout 的 10M token context window 在開放模型中領先業界

<a id="llama-3x-family-meta----previous-generation"></a>
### Llama 3.x 家族（Meta）-- 前一代

| 模型 | 參數 | Context | License | 備註 |
|-------|------------|---------|---------|-------|
| Llama 3.3 70B | 70B | 128K | Llama 3.3 | 仍被廣泛部署；強大的通用模型 |
| Llama 3.1 405B | 405B | 128K | Llama 3.1 | Meta 最大的 dense 模型；正被 Llama 4 取代 |

**注意：** Llama 3.x 仍廣泛用於正式環境，但由於 MoE，Llama 4 Scout／Maverick 以較低 active parameter 數即可提供更佳表現。

<a id="deepseek-family"></a>
### DeepSeek 家族

| 模型 | 參數 | Context | 狀態 | 備註 |
|-------|------------|---------|--------|-------|
| **DeepSeek V4 Pro** | 1.6T total / 49B active (MoE) | 1M | GA | 於 2026 年 4 月 24 日亮相。在 1M tokens 時僅需 V3.2 約 27% 算力／10% 記憶體。SWE-bench Verified 80.6%。NIST CAISI 評估（2026 年 5 月）顯示其約落後美國前沿 8 個月（Elo ~800）。在 Hugging Face 開放權重。**API：每 1M input/output 為 $0.435 / $0.87（75% 折扣於 2026 年 5 月 22 日改為永久，自 6 月 1 日生效）。** Cache-hit input 為 $0.003625/M。 |
| **DeepSeek V4 Flash** | 284B total / 13B active (MoE) | 1M | GA | 較小 active 版本，適合高吞吐量。**API：每 1M 為 $0.14 / $0.28（cache-hit 為 $0.0028/M）。** 截至 2026 年 5 月，是最便宜的 frontier-class 1M-context API。 |
| DeepSeek-V3.2 | 671B (MoE) | 128K | Frontier | 通用模型；98% cache-hit 折扣（基準價每 1M 為 $0.28/$0.42）。新系統大多已改用 V4 Flash。 |
| DeepSeek-V3 | 671B (MoE, 37B active) | 128K | Frontier | 以極低訓練成本達到 GPT-4o 等級；開放權重。 |
| DeepSeek-R1 | 671B (MoE) | 128K | Reasoning | 在數學／程式上可匹敵 o1；第一個開源推理模型。 |
| DeepSeek-R1-Distill | 7B–70B | - | Reasoning | 蒸餾到較小模型；高成本效益推理。 |

**2026 年 5 月關鍵背景：** DeepSeek V4 Pro（4 月 24 日發布，5 月 22 日將 75% 促銷折扣改為永久）以極低成本縮小了與美國前沿模型的差距。每 1M 為 $0.435 / $0.87 的 V4 Pro，大約比 Claude Opus 4.7（$5 / $25）便宜 10 倍，也比 GPT-5.5（$5 / $30）便宜 5-10 倍。V4 Flash 則進一步把價格壓到每 1M $0.14 / $0.28，且同樣提供 1M context window。兩者的 98% cache-hit 折扣，讓 V4 成為高流量 RAG 與分類工作負載中、對 cache 友善 prompt 的主導選擇。根據報導，由於 Huawei Ascend 訓練挑戰，DeepSeek R2（R1 的推理後繼）仍持續延後。

<a id="moonshot-kimi-family---may-2026-new"></a>
### Moonshot Kimi 家族 - 2026 年 5 月新版本

| 模型 | 參數 | Context | 備註 |
|-------|------------|---------|-------|
| **Kimi K2.6** | 1T total / 32B active (MoE) | - | 2026 年 4 月 20 日發布。Modified MIT license。原生 video input；Agent Swarm 可擴展到 300 個 sub-agents 與 4,000 個協同步驟。在 SWE-Bench Pro 與 GPT-5.5 同分（58.6%）；SWE-bench Verified 約 80.2%。 |
| Kimi K2-Thinking-0905 | - | - | 第一個在 AIME 2025 達到 100% 的模型（reasoning 版本）。 |

**最適合：** 長期跨度 agent 工作負載、影片理解，以及作為 closed frontier 之外的 open-weight agent stack 替代方案。

<a id="alibaba-qwen-3x-family---may-2026-new"></a>
### Alibaba Qwen 3.x 家族 - 2026 年 5 月新版本

| 模型 | 參數 | License | 備註 |
|-------|------------|---------|-------|
| **Qwen 3.6 Max-Preview** | ~1T MoE | Commercial preview | 約於 2026 年 4 月 20–27 日發布。262K context。依 Alibaba 說法，在六個 coding benchmark 奪冠。 |
| **Qwen 3.6-Plus** | - | - | 2026 年 4 月 2 日發布。強化 coding。 |
| **Qwen 3.6-35B-A3B** | 35B / 3B active MoE | Apache 2.0 | 2026 年 4 月 16 日發布。實用型 open-weight 主力模型。 |
| Qwen2.5-Coder-32B | 32B | Apache 2.0 | 前一代開源 coding 榜首。 |
| Qwen2.5-72B | 72B | Apache 2.0 | 前一代多語模型榜首。 |
| Qwen2.5-7B | 7B | Apache 2.0 | 高效率自託管選項。 |

<a id="mistral-family"></a>
### Mistral 家族

| 模型 | 參數 | Context | 備註 |
|-------|------------|---------|-------|
| **Mistral Medium 3.5** | 128B dense | 256K | 2026 年 5 月新版本。2026 年 4 月 29 日發布。把 Magistral（reasoning）+ Pixtral（vision）+ Devstral 2（coding）整合為單一模型。SWE-Bench Verified 77.6%。輸入 token 為 $1.50/M。 |
| **Voxtral TTS** | 4B open-weights | streaming | 2026 年 5 月新版本（3 月 23 日發布，CC BY-NC 4.0）。70ms 延遲、9 種語言、3 秒語音複製。 |
| Mistral Large 3 | 675B (MoE, 41B active) | 256K | Sparse MoE；與最佳 open-weight 模型同級；LMArena 中 OSS 非推理模型第 2 名。 |
| Mistral Small 4 | - | 256K | 混合 instruct／reasoning／coding；2026 年 3 月發布。 |
| Mistral 3 (14B/8B/3B) | 3B–14B | - | 統一家族：多語、多模態、Apache 2.0。 |
| Mixtral 8x22B | 141B (MoE) | - | 前一代；吞吐量場景仍可使用。 |

<a id="google-gemma-family---may-2026-new"></a>
### Google Gemma 家族 - 2026 年 5 月新版本

| 模型 | 參數 | Context | License | 備註 |
|-------|------------|---------|---------|-------|
| **Gemma 4 (31B dense)** | 31B | 256K | Apache 2.0 | 2026 年 4 月 2 日發布。140+ 種語言；原生 vision/audio；function calling。 |
| **Gemma 4 (26B-A4B MoE)** | 26B / 4B active | 256K | Apache 2.0 | Sparse MoE 版本。 |
| **Gemma 4 E4B** | 8B | 256K | Apache 2.0 | 適合 edge。 |
| **Gemma 4 E2B** | 5.1B / 2.3B active | 256K | Apache 2.0 | 最小版本；適合 mobile／embedded。 |

<a id="meta-muse-spark-closed-weights---may-2026-strategic-shift"></a>
### Meta Muse Spark（Closed Weights）- 2026 年 5 月策略轉向

| 屬性 | 數值 |
|-----------|-------|
| License | **Closed weights** - Meta Superintelligence Labs 的第一個專有模型 |
| Capabilities | 具備 Instant／Thinking／Contemplating 模式的多模態推理 |
| Released | 2026 年 4 月 8 日 |

**策略意義：** 這是 Meta 自最初 Llama 時代以來第一個非開放模型。這代表要做到 frontier 級品質，可能需要 closed-development 的反饋循環。Llama 4 Behemoth 的發布也同時因能力疑慮延後至 2026 年秋季之後。現在 open-vs-closed 的平衡已形成雙層：前沿 closed 領先 6–12 個月；open weights 再透過 distillation、RL 與生態系迭代追上。

---

<a id="specialized-models"></a>
## 專門模型

<a id="coding-mastery-april-2026"></a>
### Coding 能力（2026 年 4 月）

| 模型 | 專長 | 勝出原因 |
|-------|----------------|-------------|
| **Claude Sonnet 4.6 / Opus 4.6** | Software Engineering | Claude Code 的核心；SWE-bench 分數頂尖；1M context |
| **GPT-5.4** | Agentic coding | 原生 computer-use；強大的 full-stack coding |
| **Llama 4 Maverick** | Open-source coding | 在 coding benchmark 上勝過 GPT-4o；開放權重 |
| **Qwen2.5-Coder-32B** | 自託管 coding | 自託管 IDE 的最佳價格／效能比 |
| **DeepSeek-R1-Distill-70B** | 開源 reasoning+code | 70B 級別中最好的開源 coding 推理模型 |

<a id="reasoning--math"></a>
### 推理與數學

| 模型 | 方法 | 最適合 |
|-------|----------|----------|
| **GPT-5.4 Pro** | 最大算力推理 | 競賽數學、最困難的多步問題 |
| **Claude Opus 4.6 (thinking)** | Adaptive thinking | 軟體規劃、複雜邏輯、agentic 推理 |
| **DeepSeek-R1** | 以 RL 驅動的 thinking | 開源邏輯推論、競賽數學 |
| **Grok 4 (DeepSearch)** | 以 web 為依據的推理 | 需要即時資訊的研究任務 |

<a id="long-context-1m"></a>
### 長上下文（1M+）

| 模型 | 視窗 | Recall 表現 |
|-------|--------|-------------------|
| **Llama 4 Scout** | 10M | 開放權重中業界領先的 context window |
| **Gemini 3.1 Pro / Flash** | 1M | 在 1M context 下品質最佳，且已在大規模環境驗證 |
| **Claude Opus 4.6 / Sonnet 4.6** | 1M | 標準定價即可用完整 1M；recall 穩定 |
| **Llama 4 Maverick** | 1M | 具備 MoE 效率的 open-weight 1M context |

---

<a id="embedding-models"></a>
## Embedding 模型

<a id="api-embedding-models-april-2026"></a>
### API Embedding 模型（2026 年 4 月）

| 模型 | 維度 | Max Tokens | MTEB Score | Cost/1M |
|-------|------------|------------|------------|---------|
| OpenAI text-embedding-3-large | 3072 | 8191 | 64.6 | $0.13 |
| OpenAI text-embedding-3-small | 1536 | 8191 | 62.3 | $0.02 |
| Voyage-3 | 1024 | 32000 | 67.8 | $0.06 |
| Cohere embed-v3 | 1024 | 512 | 66.4 | $0.10 |
| Google text-embedding-004 | 768 | 2048 | 66.1 | $0.025 |

<a id="open-source-embedding-models"></a>
### 開源 Embedding 模型

| 模型 | 維度 | Max Tokens | MTEB | 備註 |
|-------|------------|------------|------|-------|
| BGE-large-en-v1.5 | 1024 | 512 | 63.9 | Instruction-tuned |
| E5-mistral-7b-instruct | 4096 | 32768 | 66.6 | 對 instructions 表現強 |
| Nomic-embed-text-v1.5 | 768 | 8192 | 62.3 | 長上下文、開放 |
| GTE-Qwen2-7B | 3584 | 32K | 72.1 | 最先進的開源 embedding |

<a id="embedding-selection-guide"></a>
### Embedding 選型指南

| 需求 | 建議 | 原因 |
|-------------|-------------|-----|
| 最佳品質 | Voyage-3 或 text-embedding-3-large | 最高 MTEB |
| 成本效率 | text-embedding-3-small | $0.02/1M |
| 自託管 | GTE-Qwen2-7B | 開源 MTEB 最佳 |
| 長文件 | Nomic 或 Voyage-3 | 8K+ context |
| 多語 | Cohere embed-v3 | 專為多語打造 |

---

<a id="model-selection-framework"></a>
## 模型選型框架

<a id="decision-tree"></a>
### 決策樹

```
What is your primary constraint?

├── Cost → Use smaller model, consider open source
│   ├── Very cost sensitive → GPT-5.4-mini, Claude Haiku 4.5, Gemini 3.1 Flash
│   └── Moderate budget → Claude Sonnet 4.6, GPT-5.4
│
├── Quality + Reasoning → Use frontier models
│   ├── Highest reasoning → GPT-5.4 Pro, Claude Opus 4.6
│   └── Coding + reasoning → Claude Sonnet 4.6 (Extended Thinking)
│
├── Latency → Use fast models
│   ├── <100ms response → Gemini 3.1 Flash, GPT-5.4-mini
│   └── <500ms response → Claude Haiku 4.5, Grok 4.1 Fast
│
├── Self-hosting → Use open models
│   ├── Maximum capability → Llama 4 Maverick, DeepSeek-V3
│   ├── Good balance → Llama 4 Scout, Llama 3.3 70B, Qwen2.5-72B
│   └── Edge/mobile → Mistral 3 3B, Phi-4
│
└── Privacy → Self-host or use on-prem
    └── Choose open models with appropriate license
```

<a id="semantic-routing"></a>
### 語意路由

在 2025-26 年，靜態決策樹正被 **Semantic Routers** 取代：
- **運作方式**：使用小型且快速的 embedding model 將查詢向量化。若符合「已知簡單」群集 → 使用便宜模型（例如 Gemini 3.1 Flash）。若命中「agentic／logic」群集 → 使用 Claude Opus 4.6 或 GPT-5.4。
- **效益**：不必寫死規則，也能自動完成成本最佳化。
- **實作方式**：可使用 `semantic-router`（Python）或自訂 Weaviate／Pinecone 分類器等工具。

---

<a id="sovereign-ai-and-data-residency"></a>
## 主權 AI 與資料駐留

**2026 年的監管現實：**
企業必須遵循 GDPR（EU）、DPDPA（India）、Saudi Arabia PDPL，以及產業別規範。「Sovereign AI」如今已成為一個產品類別。

| 解決方案 | 供應商 | 使用情境 |
|----------|----------|----------|
| **Azure Government/Sovereign** | Microsoft | 在 40+ 個地區提供專屬基礎設施；核准用於 US Gov／EU NIS2 |
| **AWS Sovereign Cloud** | Amazon | 實體隔離的 VPC；符合 GDPR 的 EU 區域 |
| **Google Distributed Cloud** | Google | 可 air-gap 的 on-prem Gemini 部署 |
| **Private Llama 4 / 3.3** | Meta（self-host） | 最大化資料主權；開放權重（Llama 4 MoE 或 3.3 dense） |
| **DeepSeek（self-host）** | DeepSeek（open） | 開放權重；資料不會離開你的基礎設施 |
| **Mistral Large 3（self-host）** | Mistral（Apache 2.0） | 675B MoE；開放權重；多語能力強 |

**取捨：** 主權雲相較標準全球區域通常有 **20-30% 溢價**，但對金融與政府領域屬於必要選項。

<a id="cost-comparison-at-scale-april-2026"></a>
### 大規模成本比較（2026 年 4 月）

假設每天 1M 次請求，每次 1K input + 500 output tokens：

| 模型 | 每日輸入成本 | 每日輸出成本 | 每月總成本 |
|-------|----------------|-----------------|-------------|
| Claude Sonnet 4.6 | $3,000 | $7,500 | $315,000 |
| GPT-5.4 | $2,500 | $7,500 | $300,000 |
| Gemini 3.1 Pro | $2,000 | $6,000 | $240,000 |
| GPT-5.4-mini | $750 | $2,250 | $90,000 |
| Gemini 3.1 Flash | $100 | $1,500 | $48,000 |
| Self-hosted Llama 4 Scout* | - | - | ~$15,000 |
| Self-hosted Llama 3.3 70B* | - | - | ~$50,000 |

*Self-hosted Llama 4 Scout 可放進單張 H100；Llama 3.3 70B 假設使用 4x H100 GPU

---

<a id="capability-comparison"></a>
## 能力比較

<a id="benchmark-performance-april-2026"></a>
### Benchmark 表現（2026 年 4 月）

| 模型 | MMLU | HumanEval | SWE-bench Verified | 備註 |
|-------|------|-----------|--------------------|-------|
| **Claude Opus 4.6** | - | - | - | 在推理與 coding 皆屬頂級；具體分數請查看最新資料 |
| **GPT-5.4** | - | - | - | 相較 GPT-5.2，事實錯誤少 33%；coding + agentic 能力強 |
| **Claude Sonnet 4.6** | - | - | - | 在許多任務上逼近 Opus 等級 |
| **Gemini 3.1 Pro** | - | - | - | Google 最先進推理模型 |
| **Grok 4** | - | - | - | 推理能力具競爭力；具備即時 web 整合 |
| **Llama 4 Maverick** | - | - | - | 在公布的 benchmark 上勝過 GPT-4o、Gemini 2.0 Flash |
| **DeepSeek-R1** | 90.8 | 92.6 | 49.2% | 第一個開源推理模型；數學／程式很強 |

*來源：各模型技術報告，以及 LMSYS Chatbot Arena / LMArena，2026 年 4 月。最新模型（Opus 4.6、GPT-5.4、Gemini 3.1）的 benchmark 分數變化很快，請務必再次確認最新排行榜。*

<a id="task-specific-recommendations-april-2026"></a>
### 任務類型建議（2026 年 4 月）

| 任務 | 建議模型 | 原因 |
|------|--------------------|-----|
| **Autonomous Coding Agent** | Claude Sonnet 4.6 / Opus 4.6 | Claude Code 的核心；1M context；tool reliability 頂尖 |
| **Complex Reasoning** | GPT-5.4 Pro, Claude Opus 4.6 (thinking), DeepSeek-R1 | 最大推理能力 |
| **Agentic Computer Use** | GPT-5.4 | 第一個具原生 computer-use 能力的通用模型 |
| **High-Volume API** | Gemini 3.1 Flash, GPT-5.4-mini | 同級中每 token 成本最低 |
| **Long Context RAG** | Gemini 3.1 Pro/Flash (1M), Claude Sonnet 4.6 (1M) | 已驗證的長距 recall |
| **Ultra-Long Context** | Llama 4 Scout (10M) | 業界領先的 10M context；開放權重 |
| **Multimodal Real-time** | Gemini 3.1 Flash | 原生即時 audio/video/text |
| **Private Production** | Llama 4 Maverick, Llama 3.3 70B, Qwen2.5-72B | 高能力且可本地控制 |
| **Open-source Coding** | Llama 4 Maverick, Qwen2.5-Coder-32B | 開放權重，coding benchmark 表現強 |
| **Creative/Chat** | GPT-5.4 | 對話品質與 instruction following 很強 |

---

<a id="interview-questions"></a>
## 面試題

<a id="q-how-would-you-select-a-model-for-a-production-rag-system"></a>
### Q：你會如何為正式環境的 RAG 系統選擇模型？

**強答範例：**
我會從以下面向評估：

**1. 品質需求：**
- 以真實領域中的代表性查詢做測試
- 衡量答案正確性、幻覺率、引用準確度

**2. 成本分析：**
```
Monthly cost = requests/day × 30 × avg_tokens × rate
```
一定要對前 2-3 個候選模型都算一遍。

**3. 延遲需求：**
- 如果需要 <200ms TTFT：Gemini 3.1 Flash、Claude Haiku 4.5、GPT-5.4-mini
- 如果品質最重要：可接受 Claude Opus 4.6 或 GPT-5.4 的 2-3 秒延遲

**4. 營運需求：**
- Self-hosting：Llama 4 Scout/Maverick、DeepSeek-V3
- 合規／資料駐留：Azure Sovereign 或 self-hosted

**5. 實際選型：**
- 先用 Claude Sonnet 4.6 或 GPT-5.4 做原型
- 對 80% 查詢以 Gemini 3.1 Flash 做 A/B test（節省成本）
- 透過 semantic routing，將困難查詢送往 frontier 模型

<a id="q-explain-the-tradeoffs-between-proprietary-and-open-source-models"></a>
### Q：請說明專有模型與開源模型之間的取捨。

**強答範例：**
| 因素 | 專有模型（OpenAI、Anthropic） | 開源模型（Llama、DeepSeek） |
|--------|--------------------------------|-----------------------------|
| 品質 | 通常略高 | 正快速追上 |
| 成本 | 按 token 計價 | 算力 + 營運成本 |
| 控制權 | 有限 | 完整 |
| 隱私 | 資料送到供應商 | 保留在本地 |
| 更新 | 自動 | 手動 |
| 客製化 | 有限 fine-tuning | 可完整 fine-tuning |
| Ops 負擔 | 幾乎沒有 | 顯著 |

**關鍵洞察（2026）**：DeepSeek-V3/R1，加上如今的 Llama 4，已改變這場討論——開放模型在許多 benchmark 上已能追平甚至超過 GPT-4o。隨著 Llama 4 Maverick 以一半 active parameters 達到接近 DeepSeek V3 的推理能力，差距比以往任何時候都更小。

<a id="q-what-is-the-difference-between-gpt-54-pro-and-claude-opus-46s-extended-thinking"></a>
### Q：GPT-5.4 Pro 與 Claude Opus 4.6 的 Extended Thinking 有何不同？

**強答範例：**
兩者都使用內部 chain-of-thought，但機制不同：

- **GPT-5.4 Pro**：OpenAI 的最高算力推理等級（每 1M tokens 為 $30/$180）。會配置大量算力進行推理。內部思考不會暴露。是 o3 系列的後繼。
- **Claude Opus 4.6 Adaptive Thinking**：會在獨立的 `<thinking>` 區塊回傳思考 tokens。可設定 `budget_tokens`。你可以檢視推理鏈以利除錯。提供完整 1M context 與 128K max output。

**正式環境選擇：** 若重視除錯與建立信任，Claude 的可見 thinking 更透明。若要在數學／競賽類任務追求最高原始推理能力，GPT-5.4 Pro 領先。若要兼顧成本效益的推理，Claude Sonnet 4.6 或 GPT-5.4-mini 都是不錯的選擇。

---

<a id="references"></a>
## 參考資料

- Anthropic: https://platform.claude.com/docs/en/about-claude/models/overview
- OpenAI Platform: https://developers.openai.com/api/docs/models
- Google AI: https://ai.google.dev/gemini-api/docs/models
- Meta Llama: https://www.llama.com/
- DeepSeek: https://api-docs.deepseek.com/
- xAI Grok: https://docs.x.ai/developers/models
- Mistral AI: https://docs.mistral.ai/models/
- LMArena Leaderboard: https://lmarena.ai/
- Hugging Face Open LLM Leaderboard: https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard

---

*下一篇：[Capability Assessment](02-capability-assessment.md)*

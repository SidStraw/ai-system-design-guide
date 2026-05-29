<a id="langsmith-observability"></a>
# LangSmith 可觀測性

在 2023 年，LLM observability 幾乎等同於「記錄字串」。現在，它已變成 **完整軌跡除錯** 與 **自動化評估 pipeline**。LangSmith 是 LangChain 原生的選項，而在擁擠的「LLMOps」層中，其他選手還包括 Langfuse（於 2026 年 1 月被 ClickHouse 收購）、LangWatch、Braintrust 與 Arize Phoenix。

<a id="table-of-contents"></a>
## 目錄

- [可觀測性金字塔](#pyramid)
- [Tracing 與軌跡](#tracing)
- [LLM 的單元測試（Datasets）](#datasets)
- [自動化評估器（LLM-as-Judge）](#evaluators)
- [管理部署：A/B Testing](#ab-testing)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="the-observability-pyramid"></a>
## 可觀測性金字塔

1. **頂層（價值）**：使用者任務是否真的被完成？（Success Rate）
2. **中層（流程）**：哪個 agent node 是瓶頸？（每個 node 的 Latency/Cost）
3. **底層（原始資料）**：精確的 prompt/completion 配對是什麼？（Traces）

---

<a id="tracing-and-trajectories"></a>
## Tracing 與軌跡

LangSmith 會自動擷取 **LangGraph** 或 **Chain** 中的每個節點。
- **Metadata Tagging**：為每條 trace 加上 `user_id`、`model_tier` 與 `is_canary` 標記。
- **Debugger**：你可以在 LangSmith UI 中「回放」一條 trace，修改 prompt 並觀察回應如何變化。這不需要重新執行整個應用程式。

---

<a id="unit-testing-for-llms-datasets"></a>
## LLM 的單元測試（Datasets）

打造 LLM 應用卻沒有 **Dataset**，就像是「憑感覺開發」。
- **Gold Standard Datasets**：由 `(Input, Expected_Output)` 配對組成的集合。
- **標準工作流**：每當使用者提供負面回饋時，該互動都會自動被送進「Correction Dataset」，供後續測試使用。

---

<a id="automated-evaluators"></a>
## 自動化評估器

你不可能每天早上手動檢查 1,000 筆 log。
- **LLM-as-Judge**：使用更強的模型（Claude Opus 4.7、GPT-5.5 reasoning、DeepSeek-R2），從 **Tone**、**Accuracy** 與 **Safe Action execution** 等面向替 production model 打分。
- **Custom Evaluators**：用 Python functions 檢查 regex patterns、JSON schema 有效性，或 Toxicity scores。

---

<a id="ab-testing"></a>
## A/B Testing

LangSmith 支援 **實驗分支**。
- 讓 2% 的流量跑在新的「System Prompt」版本上。
- 即時比較 **Success Rate** 與 **Token Cost**。
- 若失敗率超過門檻，則自動回滾。

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-why-is-trace-attribution-critical-for-staff-level-engineers"></a>
### 問：為什麼「Trace Attribution」對 Staff-level engineers 如此重要？

**理想回答：**
在複雜的多代理系統中，最終輸出可能很糟，但真正的錯誤其實發生在 10 個步驟前的某個「Researcher」節點。若沒有 **Trace Attribution**，你只能靠猜測決定要修哪個 prompt。Attribution 讓我看見整條 **推理脈絡**。我可以發現是「Researcher」沒有找到正確 URL，才導致「Summarizer」產生 hallucination。這樣就能做 **精準優化**，而不是廣泛地做「Prompt Engineering」。

<a id="q-how-do-you-justify-the-cost-of-an-observability-platform-like-langsmith"></a>
### 問：你會如何合理化 LangSmith 這類 observability platform 的成本？

**理想回答：**
這類成本會被 **開發者生產力** 與 **Token Efficiency** 抵銷。工程師花上一整天「猜」模型為何失敗，其成本往往遠高於每月訂閱費。此外，透過 LangSmith 找出那些會「Meander」的 agents（也就是走太多步的代理），我可以優化 graphs，將平均步數從 8 降到 5，直接帶來 **30-40% 的 LLM API 帳單下降**。

---

<a id="references"></a>
## 參考資料
- LangChain Team. "LangSmith: The Unified Evaluation Platform" (2025)
- Microsoft. "Tracing and Debugging Multi-Agent Systems" (2025)
- Weights & Biases. "Integrating LLOps into the CI/CD Pipeline" (2024/2025)

---

*下一步：[LlamaIndex and Data-Centric AI](04-llamaindex.md)*

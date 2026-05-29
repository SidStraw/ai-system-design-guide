<a id="cost-optimization-playbook"></a>
# 成本最佳化手冊

AI 成本已不再是「黑魔法」。它們可以被量測、預測，也高度可最佳化。隨著過去一年 API 定價下降 30-60%，成本槓桿如今主要落在 *routing* 與 *caching*，而不只是選比較便宜的供應商。本章涵蓋如何在不犧牲品質的前提下，把推論成本降低 10 倍。

<a id="table-of-contents"></a>
## 目錄

- [AI 的單位經濟學](#unit-economics)
- [模型分流（效率分層）](#model-cascading)
- [小型語言模型（SLMs）](#slms)
- [Spot Instance 策略](#spot-instances)
- [「Token Tax」最佳化](#token-tax)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-unit-economics-of-ai"></a>
## AI 的單位經濟學

我們以 **Tokens per Dollar ($)** 衡量成功。

| 組成 | 成本驅動因素 | 最佳化方式 |
|-----------|-------------|--------------|
| **Compute** | GPU 時間（$/hr） | 更佳利用率（Batching）。 |
| **VRAM** | KV Cache 大小 | GQA、Quantization。 |
| **Network** | Payload 大小 | 壓縮、本地 serving。 |
| **API** | 每 token 定價 | Caching、模型選擇。 |

---

<a id="model-cascading-efficiency-tiers"></a>
## 模型分流（效率分層）

最有效的節流策略，就是使用**能完成任務的最便宜模型**。

**分流模式：**
1. **Classifier**：用極小模型（0.5B）判斷查詢複雜度（$0.00）。
2. **Tier 1 (SLM)**：90% 的查詢（問候、簡易 Q&A）送往 8B 模型（$）。
3. **Tier 2 (Frontier)**：9% 的查詢（複雜推理）送往 405B / Claude Sonnet 4.6 / GPT-5.5 / Gemini 3.1 Pro 等級模型（$$$）。
4. **Tier 3 (Reasoning)**：1% 的查詢（專家級）送往 Claude Opus 4.7 或具 extended thinking 的 GPT-5.5 等 thinking models（$$$$$）。

**整體結果**：相較於全部流量都送到 Tier 2，成本可下降 80%。

---

<a id="small-language-models-slms-for-production"></a>
## 正式環境中的小型語言模型（SLMs）

3B-8B 模型（Llama 4 8B、Gemini 3.1 Flash、Claude Haiku 4.5）現在在多數 benchmark 上已能追平，甚至超越 2023 年的原版 GPT-4。
- **Use Case**：實體抽取、情緒分析、簡單 RAG。
- **Cost**：比前沿模型便宜 100 倍。
- **Latency**：回應時間 < 100ms。

<a id="the-deepseek-v4-floor"></a>
### DeepSeek V4 的成本地板
DeepSeek V4 Flash（2026 年 4 月 24 日發布）把便宜的 frontier-class inference 地板重新拉到 **每 1M tokens $0.14 / $0.28**，並提供 1M context window，且 cache-hit input 價格為 $0.0028/M。DeepSeek V4 Pro 在 2026 年 5 月 22 日將 75% 折扣永久化後，成本大約只有 Claude Opus 4.7 的 1/10（每 1M 為 $0.435 / $0.87 vs $5 / $25）。對於快取密集、高流量、且前綴重用頻繁的工作負載（共用知識庫的 RAG、批次分類、codebase agents），在開始做模型分流前，V4 Flash 或 V4 Pro 就已經是最強成本最佳化槓桿。正式採用前，請先到 [DeepSeek pricing page](https://api-docs.deepseek.com/quick_start/pricing) 驗證。

---

<a id="spot-instance-strategies"></a>
## Spot Instance 策略

對非即時工作負載（批次處理、資料抽取）而言，可使用 **GPU Spot Instances**（AWS Spot、Azure Spot、Lambda Labs）。

- **Risk**：GPU 可能在 30 秒通知後被回收。
- **Mitigation**：**Live KV-Cache Migration**。Serving frameworks 可在收到「Reclamation Signal」時，立即把進行中的請求 KV cache 串流到其他節點，確保工作不遺失。

---

<a id="the-token-tax-optimization"></a>
## 「Token Tax」最佳化

- **System Prompt Caching**：把常用前綴硬編碼，以取得 90% 折扣。
- **Output Truncation**：嚴格限制 `max_tokens`。
- **Negative Prompting**：「Don't be wordy」可節省約 15% 輸出 tokens（也就是成本）。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-how-do-you-justify-the-cost-of-an-ai-system-to-a-cfo"></a>
### Q: 你會如何向 CFO 證明 AI 系統成本合理？

**強答：**
我會聚焦在**效率的 ROI**。首先，我會實作「Model Cascading」，確保 90% 的流量都由每百萬 token 成本不到幾分錢的模型處理。其次，我會導入「Semantic Caching」，避免為相同答案重複付費。第三，我會建立「Inference Quotas」與「Chargeback Models」，讓每個業務單位都對自己的用量負責。把 AI 視為具有分級定價的「Commodity Resource」，我們就能從「無上限實驗」轉向「可預期的 OpEx」模式。

<a id="q-when-is-a-self-hosted-individual-gpu-cluster-cheaper-than-an-api"></a>
### Q: 什麼情況下，自行託管的單一 GPU 叢集會比 API 更便宜？

**強答：**
「交叉點」通常出現在**持續高吞吐量**時。如果你的應用程式每天 24 小時、每秒固定有 5-10 個請求，H100 預留實例的固定成本就會低於 API 的變動 token 成本。但如果流量是「尖峰型」或高度集中在上班時間，API 通常更便宜，因為你可以在離峰時段「為沉默付費」。對大多數企業來說，70B 等級模型的損益平衡點大約落在每月 5 億 tokens。

---

<a id="references"></a>
## 參考資料
- Google Cloud. "Cost Optimization for Generative AI" (2024)
- Anyscale. "LLM Inference: API vs. Self-Hosted Costs" (2024)

---

*下一篇：[Prompt Engineering Fundamentals](../05-prompting-and-context/01-prompt-engineering-fundamentals.md)*

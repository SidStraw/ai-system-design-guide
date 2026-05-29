<a id="pretraining-basics"></a>
# 預訓練基礎

預訓練是建構 LLM 時運算成本最高的階段，模型會在海量資料集上學習通用知識與語言模式。

<a id="table-of-contents"></a>
## 目錄

- [預訓練目標](#the-pretraining-objective)
- [資料課程與品質](#data-curriculum-and-quality)
- [縮放法則（推論最適）](#scaling-laws)
- [運算需求](#computational-requirements)
- [訓練穩定性](#training-stability)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-pretraining-objective"></a>
## 預訓練目標

大多數現代 LLM 都是 **Decoder-only**，並使用 **Causal Language Modeling (CLM)**：

```python
# Objective: Minimize Cross-Entropy Loss
Loss = -sum(log P(token_i | token_1, ..., token_{i-1}))
```

模型會根據上下文預測下一個 token。這個簡單的目標在大規模訓練下，會帶來湧現式推理能力。

---

<a id="data-curriculum-and-quality"></a>
## 資料課程與品質

焦點已從「更多資料」轉向「更好的課程設計」。

<a id="the-100t-token-horizon"></a>
### 100T Token 視野
前沿模型（Llama 4、GPT-5.5、Claude Opus 4.7、Gemini 3.1 Pro）的訓練資料量介於 15T 到 100T tokens。到了這個規模，**Deduplication** 與 **Quality Filtering** 成為主要差異化因素。

<a id="data-mixture-standard"></a>
### 資料混合標準
| 組成 | 比例 | 目的 |
|-----------|------------|---------|
| Web (CommonCrawl) | 50-60% | 通用知識、多樣文風 |
| Code (Github, StackOverflow)| 15-20% | **對 Logic 與 Reasoning 至關重要** |
| Books (Project Gutenberg) | 10% | 敘事一致性、長上下文 |
| Academic (ArXiv, PubMed) | 10% | 專業技術知識 |
| Synthetic (Model-generated) | 5-10% | 數學、邏輯，以及特定指令路徑 |

**細節：The "Code Effect"：**
研究顯示，在預訓練混合資料中增加程式碼比例，會提升模型在**非程式撰寫**推理任務（例如數學、邏輯謎題）上的表現，因為它學會了結構化思考。

---

<a id="scaling-laws-training-vs-inference-optimal"></a>
## 縮放法則：訓練最適 vs. 推論最適

<a id="the-chinchilla-paradigm-2022-2024"></a>
### Chinchilla 範式（2022-2024）
`Data Tokens (D) ≈ 20 * Parameters (N)`
對 70B 模型來說，這代表大約需要 1.4T tokens。

<a id="the-inference-optimal-paradigm"></a>
### 推論最適範式
現代模型（Llama 3、Llama 4）相較於 Chinchilla 都屬於**大幅過度訓練**。
- **Why?**：訓練成本只支付一次；推論成本卻要支付數十億次。
- **Result**：小模型（8B）現在會用 15T+ tokens 訓練，使它們能擁有接近舊世代 70B 模型的能力，但部署成本低得多。

| 策略 | Token/Param 比例 | 最適用場景 |
|----------|-------------------|----------|
| Chinchilla | 20:1 | 研究 / Proof of Concept |
| **Inference-Optimal** | **200:1 to 500:1**| 正式上線部署 |

---

<a id="training-stability"></a>
## 訓練穩定性

在「Ultra」級規模（100k+ GPUs）下訓練會面臨龐大的穩定性問題。

<a id="1-loss-spikes"></a>
### 1. Loss Spikes
Loss 突然飆升可能直接毀掉一次訓練執行。
- **標準修法**：**Periodic Checkpointing** 與 **Automatic Rollbacks**。
- **架構修法**：**Residual Scaling**（初始化權重時，讓 residual branch 從接近零開始）。

<a id="2-precision-fp8-vs-bf16"></a>
### 2. 精度：FP8 vs BF16
- **BF16**：2023-2024 年的穩定性標準。
- **FP8**：目前的正式環境標準。H100/B200 原生支援，可將記憶體使用量減半、吞吐量加倍，並透過 **Stochastic Rounding** 維持訓練穩定性。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-train-an-8b-model-on-15t-tokens-if-chinchilla-says-160b-tokens-is-optimal"></a>
### Q: 如果 Chinchilla 說 160B tokens 最理想，為什麼還要用 15T tokens 訓練 8B 模型？

**強答：**
Chinchilla 最適性關注的是在固定**訓練**算力預算下的最佳利用方式。但在正式環境中，我們更在意由推論主導的 **Total Cost of Ownership (TCO)**。透過過度訓練小模型，我們能把更多智慧「烘焙」進較少的參數裡。結果就是：模型在維持前沿品質的同時，部署效率顯著更高（更高 TPS、更低 VRAM）。

<a id="q-what-is-the-curriculum-in-llm-pretraining"></a>
### Q: LLM 預訓練中的「課程」是什麼？

**強答：**
課程指的是資料的排序與混合方式。常見的現代模式是：
1. **通用知識階段：**80% 的 tokens（Web、Books）。
2. **推理聚焦階段：**15% 的 tokens（Code、Math、Logic）。
3. **高品質「降溫」階段：**最後 1-5% 的 tokens 來自極高品質、人工策展或教科書級資料。這個「降溫」階段能讓模型在開始任何 fine-tuning 前更穩定、更能遵循指令。

---

<a id="references"></a>
## 參考資料
- Kaplan et al. "Scaling Laws for Neural Language Models" (2020)
- Hoffmann et al. "Training Compute-Optimal Large Language Models" (Chinchilla, 2022)
- Meta AI. "The Llama 3/4 Herd of Models" (2024/2025)

---

*下一篇：[Fine-Tuning Strategies](02-fine-tuning-strategies.md)*

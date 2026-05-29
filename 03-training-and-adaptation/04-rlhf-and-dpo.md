<a id="rlhf-and-dpo-alignment"></a>
# RLHF 與 DPO（Alignment）

Alignment 是確保 LLM 行為符合人類價值與指令的過程。這個領域已從傳統 RLHF，轉向更有效率且可擴展的方法，如 DPO 與 Online RL。

<a id="table-of-contents"></a>
## 目錄

- [Alignment 問題](#the-alignment-problem)
- [RLHF：基礎方法](#rlhf-foundation)
- [DPO：Direct Preference Optimization](#dpo)
- [線上對齊](#online-alignment)
- [推理模型的對齊](#alignment-for-reasoning)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-alignment-problem"></a>
## Alignment 問題

預訓練模型是「有知識但不受控的」。它們可能會：
1. 生成有害內容（Safety）。
2. 無法遵循指令（Instruction Following）。
3. 嚴重幻覺（Factuality）。

Alignment 透過建立「Reward Models」與「Policy Updates」來引導模型。

---

<a id="rlhf-the-foundation"></a>
## RLHF：基礎方法

Reinforcement Learning from Human Feedback (RLHF) 包含三個步驟：
1. **SFT**：Supervised Fine-Tuning。
2. **Reward Model (RM)**：在 `(Prompt, Winning_Response, Losing_Response)` 上訓練模型，以預測人類評分。
3. **PPO (Proximal Policy Optimization)**：透過 Reinforcement Learning 使用 RM 為 LLM 提供「reward signal」。

**細節**：由於訓練獨立 Reward Model 的額外成本，以及 PPO 的不穩定性，傳統 RLHF 現在被多數團隊認為過於複雜／不穩。

---

<a id="dpo-direct-preference-optimization"></a>
## DPO：Direct Preference Optimization

DPO 是業界標準。它移除了 Reward Model。

<a id="how-it-works"></a>
### 運作方式
DPO 透過數學推導，直接從偏好資料中求出最優 policy，等於讓 LLM 自己扮演 Reward Model。
- **Goal**：相對於固定的「reference model」，提高「winning」回應的機率，降低「losing」回應的機率。

<a id="the-multi-stage-alignment-pattern"></a>
### 多階段對齊模式
1. **Base SFT**：5k-10k 高品質樣本。
2. **DPO Step 1**：對齊 instruction following。
3. **DPO Step 2**：對齊 safety 與特定語氣。

---

<a id="online-alignment"></a>
## 線上對齊

**Offline DPO 的問題**：它只能從靜態資料學習。如果模型超越了那份資料，就會碰到天花板。

**解法：Online DPO（或 RLOO）**：
1. 模型針對一個 prompt 產生 4-8 個回應。
2. **Judge Model**（例如 GPT-5.5、Claude Opus 4.7）或**規則式 Reward**（例如程式執行結果）會即時替它們排序。
3. 模型根據這種「線上」回饋立刻更新 policy。

---

<a id="alignment-for-reasoning-models-o1deepseek-r1-style"></a>
## 推理模型的對齊（o1/DeepSeek-R1 風格）

對齊「Thinking」模型時，重點會從 **Response Preference** 轉向 **Process Preference**。

| 特性 | 標準對齊 | 推理對齊 |
|---------|-------------------|---------------------|
| Reward 目標 | 最終答案 | **Chain of Thought (CoT)** |
| Reward 訊號 | Helpful/Safe | **Correctness + Conciseness** |
| 方法 | 人類排序 | 規則式（例如「程式有跑過嗎？」） |

**Principal-level 細節**：「Verification-based RL」是當今前沿模型的關鍵。與其讓人類說哪個更好，我們改用可硬驗證的結果（數學答案、程式測試案例）作為 reward signal。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-is-dpo-often-preferred-over-rlhfppo"></a>
### Q: 為什麼 DPO 常比 RLHF/PPO 更受偏好？

**強答：**
DPO 之所以受偏好，主要是因為它更簡單也更穩定。PPO 需要在記憶體中同時維持四個模型（Policy、Reference、Value、Reward），非常耗 VRAM。此外，PPO 對超參數極度敏感，也常出現「reward hacking」或突然崩潰。DPO 則把 alignment 視為偏好配對上的分類問題，因此更穩健、更容易調整，也便宜得多。

<a id="q-what-is-the-risk-of-alignment-tax"></a>
### Q: 「Alignment Tax」有什麼風險？

**強答：**
「Alignment Tax」指的是模型在針對安全性或特定 persona 對齊後，其原始能力（例如寫程式、創意寫作、邏輯推理）出現下滑。因為模型被迫優先考慮安全或特定風格，所以可能變得「過於保守」，或失去預訓練時學到的細膩度。現代技術如 **Steerable Alignment** 與 **DPO-with-KL-penalty** 的目標，就是透過限制 policy 不要偏離原始預訓練分布太遠，來降低這種損失。

---

<a id="references"></a>
## 參考資料
- Rafailov et al. "Direct Preference Optimization: Your Language Model is Secretly a Reward Model" (2023)
- Schulman et al. "Proximal Policy Optimization Algorithms" (2017)
- OpenAI. "Learning to Reason with LLMs" (2024)

---

*下一篇：[Knowledge Distillation](05-knowledge-distillation.md)*

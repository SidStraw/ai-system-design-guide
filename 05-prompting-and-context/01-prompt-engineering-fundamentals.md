<a id="prompt-engineering-fundamentals"></a>
# 提示工程基礎

提示工程是設計輸入以引導 LLM 行為的方法。它已從「反覆試誤」演變為一種有紀律的架構實務；像 DSPy 這類框架更將它視為編譯問題，而不是單純的寫作練習。

<a id="table-of-contents"></a>
## 目錄

- [核心哲學：意圖 + 約束](#the-core-philosophy-intent--constraint)
- [指令階層](#the-instruction-hierarchy)
- [角色提示](#role-prompting)
- [指令清晰度與分隔符](#instruction-clarity-and-delimiters)
- [Zero-Shot 與 Few-Shot 的效率](#zero-shot-vs-few-shot-efficiency)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-core-philosophy-intent--constraint"></a>
## 核心哲學：意圖 + 約束

有效的提示，關鍵在於最大化 **Intent Disclosure**，同時最小化 **Output Variance**。

1. **意圖（Intent）**：精準說明模型應該做什麼。
2. **約束（Constraint）**：明確指出模型應該*避免*什麼（安全、語氣、格式）。

**原則**：「Prompting is Programming in Natural Language.」請把提示詞當成程式碼對待（版本控制、單元測試）。

---

<a id="the-instruction-hierarchy"></a>
## 指令階層

正式環境系統會使用分層的訊息結構：

| 角色 | 職責 | 細節 |
|------|------|------|
| **System** | 高階規則、人格設定、安全性。 | 對 frontier models 來說黏著度最高（H-rank）。 |
| **Developer** | 技術性覆寫（例如格式）。 | 是較新的角色，常見於「un-opinionated」模型。 |
| **User** | 具體且動態的查詢。 | 容易受到 injection 影響；必須隔離。 |
| **Assistant** | 先前回合的歷史。 | 是「recency bias」的來源。 |

---

<a id="role-prompting"></a>
## 角色提示

指定 persona 已不再只是「你是一位老師」。它更像是 **Capabilities Anchor**。

- **弱**：「You are a coder.」
- **強**：「You are a Staff Software Engineer at a Tier-1 tech company specializing in high-concurrency Rust systems. You prioritize memory safety and zero-cost abstractions.」

**為什麼有效**：它會把模型注意力聚焦到其訓練資料中，與該高階專業最相關的子集合，從而減少不相關的幻覺內容。

---

<a id="instruction-clarity-and-delimiters"></a>
## 指令清晰度與分隔符

目前的 frontier models 能處理極大量的上下文。分隔符能幫助模型區分「指令」與「資料」。

```markdown
# Instructions
Analyze the following text for PII.

# Data to Analyze
--- START OF USER DATA ---
$USER_INPUT_HERE
--- END OF USER DATA ---

# Output Schema
{ "pii_found": boolean, "types": [] }
```

**可使用的分隔符**：XML tags（`<context>`、`</context>`）、Markdown 標題（`#`）或 triple quotes（`"""`）。

---

<a id="zero-shot-vs-few-shot-efficiency"></a>
## Zero-Shot 與 Few-Shot 的效率

| 面向 | Zero-Shot | Few-Shot |
|------|-----------|----------|
| **延遲** | 最低（短提示） | 較高（範例 token） |
| **準確率** | 波動較大 | 高（格式穩定） |
| **適用情境** | 簡單對話、摘要 | 特定格式、細微邏輯 |

**策略**：如果模型屬於「Frontier Reasoning」模型（Claude Opus 4.7、具 extended thinking 的 GPT-5.5、DeepSeek-R2），請使用 **Zero-Shot + 清楚的 Chain-of-Thought**。如果是小模型（8B），請用 **Few-Shot** 來提供錨定。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-do-system-prompts-carry-more-weight-than-user-prompts-in-modern-llms"></a>
### 問：為什麼在現代 LLM 中，system prompt 通常比 user prompt 更有權重？

**強答案：**
System prompt 通常在模型的架構訓練（RLHF）中被賦予更高優先級，而且在某些架構裡，還可能被注入到特殊的「instruction-only」embedding space。從設計角度來看，system prompt 定義了整段互動的「Constitution」。如果 user prompt 與 system prompt 發生衝突（例如要求炸彈配方），一個對齊良好的模型會被訓練成優先遵守 system 的「Safety Constraint」，而不是 user 的「Task Intent」。

<a id="q-what-is-the-step-by-step-prompt-optimization"></a>
### 問：什麼是「Step-by-Step」提示最佳化？

**強答案：**
在 2022 年，「Think step by step」曾是觸發 Chain-of-Thought（CoT）的魔法短語。現代作法則是 **Programmatic CoT**。我們不再使用模糊的片語，而是提供明確的推理里程碑：「1. Identify the core problem. 2. List the constraints. 3. Propose 3 solutions. 4. Select the best one and justify.」這為模型的內部注意力提供了一條更具「決定性」的路徑，因此在正式環境 agent 中能得到更可靠的輸出。

---

<a id="references"></a>
## 參考資料
- OpenAI. "Prompt Engineering Guide" (2024-2025)
- Anthropic. "Claude Prompt Engineering Documentation" (2024)
- Google DeepMind. "The Power of Prompting" (2023)

---

*下一篇：[Few-Shot 與 In-Context Learning](02-few-shot-and-icl.md)*

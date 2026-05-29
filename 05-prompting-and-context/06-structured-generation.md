<a id="structured-generation"></a>
# 結構化生成

結構化生成是強制 LLM 以機器可讀格式（JSON、YAML、CSV）輸出，並達成 100% 可靠性的過程。這門技術已從「基於提示的請求」演進為「引擎層級的約束」。

<a id="table-of-contents"></a>
## 目錄

- [JSON Mode 革命](#the-json-mode-revolution)
- [Function Calling 與 Tool Use](#function-calling--tool-use)
- [受限解碼（CFG 與 Regex）](#constrained-decoding-cfg--regex)
- [多階段擷取模式](#multi-stage-extraction-pattern)
- [驗證與格式錯誤](#validation--formatting-errors)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-json-mode-revolution"></a>
## JSON Mode 革命

在過去，取得 JSON 往往是一場「只回傳 JSON，不要有其他文字」的搏鬥。
**標準作法**：使用原生的 `response_format: { type: "json_schema" }`（OpenAI / Gemini）或 tool-output schemas（Anthropic）。

- **好處**：100% 語法有效。模型從字面上就**無法**輸出不是合法 JSON 的字串。
- **幕後機制**：serving engine 會在每一步遮罩詞彙表，確保下一個只能挑選合法 JSON 字元（例如 `{`、`"`、`:`、`[`）。

---

<a id="function-calling--tool-use"></a>
## Function Calling 與 Tool Use

Function calling 是一種結構化生成：LLM 會「挑選」函式，並填入其參數。

```json
// Example Tool Call
{
  "name": "get_stock_price",
  "arguments": { "symbol": "AAPL", "interval": "1d" }
}
```

**細節**：**Parallel Function Calling** 現在已是標準能力。模型可以決定同時呼叫 5 個不同工具（例如查帳戶餘額、查信用分數、查貸款利率），再彙整結果。

---

<a id="constrained-decoding-cfg--regex"></a>
## 受限解碼（CFG 與 Regex）

對於自架模型（Llama-cpp、透過 Outlines 使用的 vLLM），我們會使用 **Context-Free Grammars（CFG）** 或 **Regex**。

```python
# Outlines Pattern
model = outlines.models.transformers("meta-llama/Llama-4-8B")
generator = outlines.generate.regex(model, r"(\d{3})-\d{3}-\d{4}")
# Result: The model can ONLY output telephone numbers.
```

---

<a id="multi-stage-extraction-pattern"></a>
## 多階段擷取模式

對於複雜資料擷取（例如從病歷中抽取 50 個欄位），不要一次做完。
- **Stage 1（Text-to-Text）**：先以自然語言抽取一組「雜亂但完整」的事實。
- **Stage 2（Text-to-JSON）**：再用更小、更便宜的模型，把這些自然語言事實轉成嚴格的 JSON schema。
- **好處**：能降低「hallucination under pressure」——大型模型在同時被迫推理、又要遵守嚴格語法時，通常會表現較差。

---

<a id="validation--formatting-errors"></a>
## 驗證與格式錯誤

即使使用了「JSON mode」，JSON 內部的 **Logic** 仍可能是錯的（例如欄位缺漏，或日期格式錯誤）。

**修復模式：**
1. 用 **Pydantic / Zod** 驗證輸出。
2. 若驗證失敗，把 **Traceback** 回傳給模型：
   「Error: Field 'age' must be an integer, got 'twenty'. Fix and re-generate.」
3. 大多數模型會在第一次重試時修正錯誤。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-is-json-mode-more-reliable-than-prompt-based-json-requests"></a>
### 問：為什麼「JSON Mode」比基於提示的 JSON 請求更可靠？

**強答案：**
基於提示的請求依賴模型是否**願意**遵守指令；「JSON Mode」（或 Constrained Decoding）則依賴 serving engine 在能力上**無法做別的事**。透過在推論層套用 **Logit Bias** 或 **Grammar Mask**，引擎會把下一個 token 的候選限制為僅有那些符合 schema 的選項。這消除了前言文字（例如「Sure, here is your JSON...」），也確保你不會因為高溫度或隨機性而拿到 malformed string。

<a id="q-what-is-the-risk-of-asking-an-llm-for-too-many-structured-fields-at-once"></a>
### 問：一次要求 LLM 產出太多結構化欄位，風險是什麼？

**強答案：**
這是在 **Schema Complexity** 與 **Information Integrity** 之間的取捨。隨著 schema 變得更大（例如 20+ 個階層式欄位），模型的注意力會被維持 JSON 結構（括號、鍵名、引號）所消耗，而不是拿來驗證資料正確性。結果往往會出現「Omission Hallucinations」：模型略過某些欄位，或用佔位資料填入。緩解方式是使用「Chain-of-Density」擷取，或把擷取拆成多個平行子任務。

---

<a id="references"></a>
## 參考資料
- OpenAI. "Structured Outputs Documentation" (August 2024 update)
- Outlines Project. "Context-Free Grammar Guided Generation" (2024)
- Willard et al. "Efficient Guided Generation for LLMs" (2023)

---

*下一篇：[提示最佳化（DSPy）](07-prompt-optimization-dspy.md)*

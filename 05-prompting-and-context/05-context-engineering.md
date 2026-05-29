<a id="context-engineering"></a>
# 上下文工程

上下文工程是一門科學：在 LLM 有限的「工作記憶」中，放入最有價值的 token。隨著 context window 已擴展到 1M+ tokens（Claude Sonnet 4.6、Gemini 3.1 Pro、GPT-5.5），且模型獲得了 Extended Thinking，焦點已從「把資料塞進去」轉向「排序相關性」與「管理計算預算」。

<a id="table-of-contents"></a>
## 目錄

- [長上下文典範（1M+ Tokens）](#the-long-context-paradigm-1m-tokens)
- [Extended Thinking 與 Budget Tokens](#extended-thinking--budget-tokens)
- [Lost-in-the-Middle](#lost-in-the-middle)
- [Context Budgeting 與 Token Awareness](#context-budgeting--token-awareness)
- [Prompt Caching 的經濟性](#prompt-caching-economics)
- [情境式壓縮（RAD-L）](#contextual-compression-rad-l)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="the-long-context-paradigm-1m-tokens"></a>
## 長上下文典範（1M+ Tokens）

像 Gemini 3.1 Pro（1M）、Claude Sonnet 4.6（1M）、Claude Opus 4.7（1M）與 GPT-5.5（1M）這類模型，都擁有巨大的 context window。

**洞見**：「Context is the new RAG.」
對於少於 100,000 份文件的資料集，把整個資料集直接放進 context window，通常比使用外部 vector database 更準、更快。這稱為 **「In-Context RAG」**。

---

<a id="extended-thinking--budget-tokens"></a>
## Extended Thinking 與 Budget Tokens

數個 frontier models 現在都提供在生成回應前，**可控制的內部推理**：

<a id="claude-sonnet-46-opus-47-extended-thinking"></a>
### Claude（Sonnet 4.6、Opus 4.7）：Extended Thinking

```python
response = client.messages.create(
    model="claude-3-7-sonnet-20250219",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000  # max internal reasoning tokens
    },
    messages=[{"role": "user", "content": "Refactor this codebase to be async..."}]
)

# Response has two blocks:
# 1. thinking block (visible for debug, not shown to user)
# 2. text block (the actual answer)
for block in response.content:
    if block.type == "thinking":
        print("[THINKING]", block.thinking)
    elif block.type == "text":
        print("[ANSWER]", block.text)
```

**關鍵參數：**
- `budget_tokens`：1,024 → 100,000。越高 = 準確率越好、成本越高。
- thinking tokens 以標準費率計費。10K 的 thinking budget = 每次請求額外 +$0.15。
- 支援 streaming——thinking blocks 會先於 text 串流出來。

<a id="o3-openai--reasoning-effort"></a>
### o3（OpenAI）— Reasoning Effort

```python
response = client.chat.completions.create(
    model="o3",
    reasoning_effort="medium",  # "low" | "medium" | "high"
    messages=[{"role": "user", "content": "Prove P=NP or disprove it."}]
)
# Reasoning tokens are invisible — o3 never exposes its internal chain
```

**Effort levels 與成本（約略）：**
| Effort | 速度 | 成本倍率 | 最適合 |
|--------|------|----------|--------|
| low | 快 | 1x | 簡單邏輯、快速查詢 |
| medium | 中 | 3-5x | 程式開發、分析 |
| high | 慢 | 8-20x | 博士級問題、ARC-AGI |

<a id="when-to-enable-thinking--reasoning"></a>
### 何時啟用 Thinking / Reasoning

| 條件 | 建議 |
|------|------|
| 複雜的多步驟程式重構 | ✅ 啟用（budget: 8K-20K） |
| 簡單 Q&A / 擷取 | ❌ 關閉——只會增加成本與延遲 |
| STEM / 數學問題 | ✅ 啟用（o3-mini medium） |
| 高流量 chatbot | ❌ 關閉——使用標準模式 |
| 安全性關鍵決策 | ✅ 啟用——額外推理能抓出 edge cases |

**正式環境模式**：使用複雜度分類器來決定是否開啟 Extended Thinking。如果查詢複雜度分數 < 0.5，就完全跳過 thinking mode（對重推理工作負載可節省 60-80% 成本）。

```python
def smart_generate(query: str) -> str:
    complexity = classifier.predict(query)  # 0-1 score
    
    if complexity > 0.7:
        # Enable Extended Thinking for hard problems
        return claude_with_thinking(query, budget_tokens=8000)
    else:
        # Standard fast mode for simple tasks
        return claude_standard(query)
```

---

<a id="lost-in-the-middle"></a>
## Lost-in-the-Middle

在 2023 年，模型對提示中間位置資訊的準確率會下降。
**現況**：frontier models（Claude Sonnet 4.6、Claude Opus 4.7、Gemini 3.1 Pro、GPT-5.5）已經改善很多，但 **Attention Gradient** 依然存在。
- **最佳實務**：把關鍵指令與 gold-standard 範例放在提示的**最前面**與**最後面**。中間區域 = 原始資料 / 知識片段。
- **使用 chunk ordering**：重新排序檢索到的文件，讓最相關的排在最前與最後。

---

<a id="context-budgeting--token-awareness"></a>
## Context Budgeting 與 Token Awareness

每一個 token 都會花錢，也會增加 TTFT（Time to First Token）。

| 組件 | 預算（Tokens） | 原因 |
|------|----------------|------|
| **System Prompt** | 500 - 1,000 | 核心邏輯與 persona。 |
| **History** | 2,000 - 5,000 | 對話式「狀態」。 |
| **Data/Search** | 10k - 1M | 取決於任務深度。 |
| **Output Reserve** | 1,000 - 4,000 | 必須預留推理空間。 |

---

<a id="prompt-caching-economics"></a>
## Prompt Caching 的經濟性

幾乎所有主要供應商（OpenAI、DeepSeek、Anthropic、Google）都支援 **Prefix Caching**。

- **交會點**：如果你會重複使用 100k token 的上下文（例如整個程式庫）超過 2 次，cache 折扣實際上就會讓它比 RAG 更便宜。
- **Cache Hits**：$0.05 / 1M tokens。
- **Cache Misses**：$5.00 / 1M tokens。

**架構上的選擇**：把你的「System Prompt + Base Knowledge」設計成固定不變，才能維持 100% 的 cache hit rate。

---

<a id="contextual-compression-rad-l"></a>
## 情境式壓縮（RAD-L）

對於極長的上下文（10M+），我們會使用 **Reasoning-Aware Deletion（RAD-L）**。
- **做法**：先由一個很小的輔助模型（0.1B）掃描文字，在提示送進大型 frontier model 之前，移除「填充」詞、常見語言模式與不相關段落。
- **好處**：在準確率下降不到 1% 的情況下，把提示大小縮減 20-50%。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-when-would-you-choose-long-context-over-rag"></a>
### 問：什麼情況下你會選擇 Long Context，而不是 RAG？

**強答案：**
當高保真檢索與跨文件推理非常重要時，我會選擇 Long Context。RAG 會受到「Retrieval Gap」影響——如果 vector search 沒抓到相關片段，模型就永遠看不到它。Long Context（最高可到 2M tokens）則提供 100% 召回率。具體來說，我會把它用在程式庫分析、法律文件審閱，以及多檔案財務稽核。對於動態的 web-scale 資料，或超出任何 context window 的十億級文件資料集，我仍會使用 RAG。

<a id="q-how-do-you-handle-the-high-ttft-associated-with-million-token-prompts"></a>
### 問：你如何處理百萬 token 提示所帶來的高 TTFT？

**強答案：**
首要解法是 **Context Caching**。透過把大型文件快取在 GPU cluster 上，模型就不必在每次回合都重新「讀取」（prefill）整整 1M tokens。對快取提示來說，TTFT 幾乎可以接近 1k token 提示。此外，對於未快取請求，我也會使用 **Streaming Prefill**，讓模型在仍處理龐大上下文後半段時，就先生成初步摘要或「Thought」。

---

<a id="references"></a>
## 參考資料
- Liu et al. "Lost in the Middle" (2023/2024 update)
- Anthropic. "Extended Thinking: Technical Guide" (2025) — https://docs.anthropic.com/
- OpenAI. "o3 and o3-mini System Card" (2025)
- Google. "Gemini 2.0 Flash: Technical Report" (2024)

---

*下一篇：[結構化生成](06-structured-generation.md)*

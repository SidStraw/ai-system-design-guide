<a id="tokenization-deep-dive"></a>
# Tokenization 深入解析

Tokenization 是把文字轉成模型可處理的離散單位（tokens）的過程。它會直接影響模型能力、成本與效能。

<a id="table-of-contents"></a>
## 目錄

- [為什麼 Tokenization 很重要](#why-tokenization-matters)
- [Tokenization 演算法](#tokenization-algorithms)
- [Vocabulary 設計取捨](#vocabulary-design-tradeoffs)
- [特殊 Tokens](#special-tokens)
- [多語言 Tokenization](#multilingual-tokenization)
- [成本估算的 Token 計數](#token-counting-for-cost-estimation)
- [常見的 Tokenization 問題](#common-tokenization-issues)
- [實務上的 Tokenization 模式](#practical-tokenization-patterns)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="why-tokenization-matters"></a>
## 為什麼 Tokenization 很重要

<a id="for-system-design"></a>
### 對系統設計而言

1. **成本**：LLM API 依 token 計費。Tokenization 效率會直接影響成本。
2. **Context 限制**：決定能放進 context 的是 token 數，不是字數。
3. **能力**：某些任務（字元計數、anagrams）之所以困難，就是因為 tokenization。
4. **一致性**：同一段文字在不同模型上會被切成不同 tokens。

<a id="for-understanding-llm-behavior"></a>
### 對理解 LLM 行為而言

**經典面試題**：為什麼 GPT 很難數出 "strawberry" 裡有幾個字母？

因為 "strawberry" 會被切成多個 subwords。模型從來沒有直接看到單一字元；它看到的是 subword 單位。要數字母，就必須推理 token 內部的結構。

---

<a id="tokenization-algorithms"></a>
## Tokenization 演算法

<a id="byte-pair-encoding-bpe"></a>
### Byte Pair Encoding (BPE)

最常見的演算法。GPT 系列、Llama、Claude 都使用它。

**訓練演算法：**
1. 先從單一 bytes 的詞彙表開始（256 個 tokens）
2. 統計訓練語料中所有相鄰 token pairs
3. 將最常出現的 pair 合併成一個新 token
4. 重複直到達到目標詞彙表大小

**範例：**
```
Corpus: "low lower lowest"
Initial: ['l', 'o', 'w', ' ', 'l', 'o', 'w', 'e', 'r', ' ', 'l', 'o', 'w', 'e', 's', 't']

Step 1: Most frequent pair is ('l', 'o'). Merge to 'lo'.
['lo', 'w', ' ', 'lo', 'w', 'e', 'r', ' ', 'lo', 'w', 'e', 's', 't']

Step 2: Most frequent pair is ('lo', 'w'). Merge to 'low'.
['low', ' ', 'low', 'e', 'r', ' ', 'low', 'e', 's', 't']

Step 3: Most frequent pair is ('low', 'e'). Merge to 'lowe'.
['low', ' ', 'lowe', 'r', ' ', 'lowe', 's', 't']

Continue until vocabulary size target...
```

**特性：**
- 在詞彙表固定後，tokenization 是決定性的
- 常見字詞傾向成為單一 token
- 稀有字詞會被拆成 subwords

<a id="wordpiece"></a>
### WordPiece

BERT 系列模型使用它。

**與 BPE 的關鍵差異：**
- BPE：依頻率合併
- WordPiece：依 likelihood 改善幅度合併

```
Score = freq(AB) / (freq(A) * freq(B))
```

這讓它更偏好那些比隨機共現更有意義的合併。

**視覺標記：**WordPiece 會用 ## 前綴表示延續 token：
```
"embedding" becomes ["em", "##bed", "##ding"]
```

<a id="unigram-sentencepiece"></a>
### Unigram (SentencePiece)

T5、ALBERT，以及部分多語言模型使用它。

**訓練演算法：**
1. 先建立很大的候選詞彙表
2. 計算移除每個 token 後造成的 loss
3. 移除那些讓 loss 增加最少的 tokens
4. 重複直到達到目標詞彙表大小

**關鍵差異：**它依賴機率，而不是頻率，因此能從早期不理想的合併中恢復。

<a id="comparison"></a>
### 比較

| 演算法 | 合併準則 | Tokenization | 使用者 |
|--------|----------|--------------|--------|
| BPE | 頻率 | 決定性 | GPT、Llama、Claude |
| WordPiece | Likelihood | 決定性 | BERT、DistilBERT |
| Unigram | 機率 | 機率式 | T5、mT5、XLNet |

---

<a id="vocabulary-design-tradeoffs"></a>
## Vocabulary 設計取捨

<a id="vocabulary-size"></a>
### Vocabulary 大小

| 大小 | 範例 | 優點 | 缺點 |
|------|------|------|------|
| 小（10K） | 某些早期模型 | 較小的 embeddings | token 序列很長 |
| 中（32K） | Llama 2 | 取得良好平衡 | 多語言效率不足 |
| 大（128K） | Llama 3/4、GPT-4o | **2025 年末的標準**。壓縮率高。 | embeddings table 更大 |
| 超大（200K+） | GPT-5.2 (o200k) | 原生多模態與多語言效率更好 | LM Head 的記憶體壓力更高 |

**2025 年的詞彙表擴張（深入說明）：**
- **Llama 3/4 (128k)**：從 32k 擴展到 128k 後，Meta 將英文壓縮率提升了約 15%，對 Hindi 等非英文語言則提升 3-4 倍。
- **GPT-4o/5.2 (o200k_base)**：Tiktoken 最新編碼在程式碼與多語言文字上有更好的壓縮率，透過同樣語意使用更少 tokens，間接降低 API 成本。

<a id="character-vs-subword-vs-word"></a>
### Character vs Subword vs Word

| 粒度 | 範例 | "running" 的 tokens | 取捨 |
|------|------|----------------------|------|
| Character | ByT5 | ['r','u','n','n','i','n','g'] | 能處理任何文字，但序列非常長 |
| Subword | GPT | ['running'] 或 ['run','ning'] | 平衡最佳 |
| Word | 早期 NLP | ['running'] | 序列短，但無法處理 OOV |

現代 LLM 幾乎一律使用 subword tokenization，在詞彙表大小與序列長度之間取得平衡。

<a id="byte-level-bpe"></a>
### Byte-Level BPE

GPT-2 引入了 byte-level BPE：
- 基礎詞彙表是 256 個 bytes，而不是字元
- 可以表示任何文字，不需要 UNK tokens
- Unicode 會自然地以 byte sequences 處理

```python
# Character-level: Needs explicit handling of characters
text = "cafe"  # Unknown character might become [UNK]

# Byte-level: Works with any text (no UNK needed)
text = "cafe"  # Becomes bytes, then BPE operates on bytes
```

---

<a id="special-tokens"></a>
## 特殊 Tokens

特殊 tokens 用來處理一般文字之外的結構資訊：

| Token | 用途 | 範例 |
|-------|------|------|
| BOS | 序列開頭 | 表示生成開始 |
| EOS | 序列結尾 | 表示完成 |
| PAD | Padding | 將 batch 補到相同長度 |
| UNK | 未知 token | OOV 的回退方案（byte BPE 中很少見） |
| SEP | 分隔符 | 用來切分片段（BERT 風格） |

<a id="chat-templates"></a>
### Chat Templates

現代 chat 模型會使用特殊 tokens 來表示對話結構：

**Llama 2 格式：**
```
[INST] <<SYS>>
You are a helpful assistant.
<</SYS>>

User message here [/INST] Assistant response here
```

**ChatML（OpenAI 風格）：**
```
<|im_start|>system
You are a helpful assistant.<|im_end|>
<|im_start|>user
Hello!<|im_end|>
<|im_start|>assistant
Hi there!<|im_end|>
```

**為什麼這很重要：**
- 格式錯誤會導致結果變差
- 特殊 tokens 不存在於預訓練資料中
- 像 transformers 這類函式庫會用 chat_template 自動格式化

---

<a id="multilingual-tokenization"></a>
## 多語言 Tokenization

<a id="the-challenge"></a>
### 挑戰

主要以英文訓練的 tokenizer，對其他語言的效率通常很差：

| 語言 | "Hello" 的 tokens | 對應問候語的 tokens |
|------|--------------------|----------------------|
| 英文 | 1（"Hello"） | - |
| 中文 | - | 2-3+ |
| 日文 | - | 3-5+ |
| 韓文 | - | 2-4+ |

**成本含意：**非英文使用者為了同樣的語意單位，往往要多付 2-3 倍成本。

<a id="solutions"></a>
### 解法

1. **多語言訓練語料**：以平衡的多語言資料訓練 tokenizer
2. **更大的詞彙表**：為非英文 tokens 騰出更多空間
3. **語言專用 tokenizer**：依語系分別建立 tokenizer

**多語言支援良好的模型：**
- mT5、XLM-R：在 100+ 種語言上訓練
- GPT-4、Claude 3.5：大詞彙表涵蓋多語言
- Gemini：從一開始就以多語言為設計目標

| 模型 | 中文 | 日文 | 韓文 | Hindi |
|------|------|------|------|-------|
| GPT-2 | 2.5x | 3.0x | 2.8x | 6.0x |
| GPT-4 (cl100k) | 1.4x | 1.6x | 1.5x | 3.2x |
| GPT-5.2 (o200k) | 1.1x | 1.2x | 1.1x | 1.4x |
| Llama 3/4 (128k)| 1.2x | 1.3x | 1.2x | 1.5x |

---

<a id="multimodal-tokenization-pixels-to-tokens"></a>
## 多模態 Tokenization（pixels-to-tokens）

現代原生多模態模型不只是「看見」圖片；它們會把圖片 token 化。

<a id="image-tokenization-vision-transformers"></a>
### 影像 Tokenization（Vision Transformers）
圖片會被切成 patches（例如 14x14 pixels）。每個 patch 都會送進 vision encoder（如 SigLIP），產生一個視覺 token。
- **固定 Token 成本**：多數模型會在特定解析度下，為每張圖片使用固定數量 tokens（例如每張圖 256 或 729 tokens）。
- **動態解析度**：某些模型（Gemini 3）會依據圖片的長寬比與細節程度，使用可變數量的 tokens。

<a id="audiovideo-tokenization"></a>
### 音訊／影片 Tokenization
- **音訊**：透過 EnCodec 之類的 codec 壓縮成離散單位，再表示成 audio tokens 序列。
- **影片**：視為影格序列（temporal tokenization）。1 秒影片若以 1FPS 取樣，成本可能與 1 張高解析度圖片相當。

---

<a id="token-counting-for-cost-estimation"></a>
## 成本估算的 Token 計數

<a id="quick-estimation-rules"></a>
### 快速估算法則

對英文文字而言：
- **字數轉 tokens**：每個單字約 1.3 tokens
- **字元轉 tokens**：每 4 個字元約 1 個 token
- **頁數轉 tokens**：每頁約 500-800 tokens

```python
def estimate_tokens(text: str) -> int:
    # Rough estimation for English
    word_count = len(text.split())
    return int(word_count * 1.3)
```

<a id="accurate-counting"></a>
### 精準計數

請使用模型專屬 tokenizer：

```python
import tiktoken

# For OpenAI models
encoding = tiktoken.encoding_for_model("gpt-4")
tokens = encoding.encode("Your text here")
token_count = len(tokens)

# For Llama/Anthropic, use transformers
from transformers import AutoTokenizer
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-2-7b")
tokens = tokenizer.encode("Your text here")
token_count = len(tokens)
```

<a id="cost-calculation"></a>
### 成本計算

```python
def calculate_cost(input_text: str, output_text: str, model: str) -> float:
    pricing = {
        "gpt-4o": {"input": 2.50, "output": 10.00},  # per 1M tokens
        "gpt-4o-mini": {"input": 0.15, "output": 0.60},
        "claude-3.5-sonnet": {"input": 3.00, "output": 15.00},
    }

    encoding = tiktoken.encoding_for_model(model)
    input_tokens = len(encoding.encode(input_text))
    output_tokens = len(encoding.encode(output_text))

    cost = (
        (input_tokens / 1_000_000) * pricing[model]["input"] +
        (output_tokens / 1_000_000) * pricing[model]["output"]
    )
    return cost
```

---

<a id="common-tokenization-issues"></a>
## 常見的 Tokenization 問題

<a id="issue-1-token-boundary-misalignment"></a>
### 問題 1：Token 邊界未對齊

**問題：**文字操作可能無法對齊 token 邊界。

```python
text = "Hello world"
# Tokens: ["Hello", " world"]  # Note: space is part of second token

# Truncating at character 6 ("Hello ") splits a token
```

**解法：**在管理 context 時，永遠要以 token 邊界截斷。

<a id="issue-2-inconsistent-tokenization"></a>
### 問題 2：Tokenization 不一致

**問題：**同一段文字會因上下文不同而被切成不同 tokens。

```python
# GPT tokenizer example
"New York"     # Might be ["New", " York"]
"NewYork"      # Might be ["New", "York"]
" New York"    # Might be [" New", " York"]
```

**含意：**token 數量會隨周邊文字改變。務必對完整 context 做 tokenization。

<a id="issue-3-code-and-structured-data"></a>
### 問題 3：程式碼與結構化資料

**問題：**程式碼與 JSON 常常會被低效率地 token 化。

```python
# Python code often tokenizes poorly
"def calculate_average(numbers):"
# Becomes many tokens: ["def", " calculate", "_", "average", "(", "numbers", "):", ...]

# JSON keys tokenize individually
'{"firstName": "John"}'
# Many tokens for structure
```

**緩解方式：**
- 有些模型使用為程式碼最佳化的 tokenizer
- 傳送前可考慮壓縮 JSON
- 若支援，使用 structured output modes

<a id="issue-4-whitespace-handling"></a>
### 問題 4：空白處理

**問題：**不同 tokenizer 對空白的處理方式不同。

```python
# Leading spaces often become separate tokens
" Hello"  # [" ", "Hello"] or [" Hello"]

# Multiple spaces may merge or stay separate
"Hello  world"  # Behavior varies by tokenizer
```

**最佳實務：**tokenization 前先正規化空白。

---

<a id="practical-tokenization-patterns"></a>
## 實務上的 Tokenization 模式

<a id="pattern-1-context-window-management"></a>
### 模式 1：Context Window 管理

```python
def fit_to_context(
    system_prompt: str,
    user_message: str,
    history: list[str],
    max_tokens: int = 8000,
    reserve_for_output: int = 2000
) -> str:
    encoding = tiktoken.encoding_for_model("gpt-4")

    available = max_tokens - reserve_for_output

    # System prompt always included
    tokens_used = len(encoding.encode(system_prompt))
    available -= tokens_used

    # User message always included
    tokens_used = len(encoding.encode(user_message))
    available -= tokens_used

    # Add history from most recent, drop oldest if needed
    included_history = []
    for msg in reversed(history):
        msg_tokens = len(encoding.encode(msg))
        if msg_tokens <= available:
            included_history.insert(0, msg)
            available -= msg_tokens
        else:
            break

    return format_prompt(system_prompt, included_history, user_message)
```

<a id="pattern-2-chunking-at-token-boundaries"></a>
### 模式 2：在 Token 邊界切塊

```python
def chunk_at_token_boundaries(
    text: str,
    chunk_size: int = 500,
    overlap: int = 50
) -> list[str]:
    encoding = tiktoken.encoding_for_model("gpt-4")
    tokens = encoding.encode(text)

    chunks = []
    start = 0
    while start < len(tokens):
        end = min(start + chunk_size, len(tokens))
        chunk_tokens = tokens[start:end]
        chunk_text = encoding.decode(chunk_tokens)
        chunks.append(chunk_text)
        start = end - overlap

    return chunks
```

<a id="pattern-3-token-budget-allocation"></a>
### 模式 3：Token 預算分配

```python
class TokenBudget:
    def __init__(self, total: int):
        self.total = total
        self.allocated = {}

    def allocate(self, component: str, tokens: int) -> bool:
        used = sum(self.allocated.values())
        if used + tokens > self.total:
            return False
        self.allocated[component] = tokens
        return True

    def remaining(self) -> int:
        return self.total - sum(self.allocated.values())

# Usage
budget = TokenBudget(total=8000)
budget.allocate("system_prompt", 500)
budget.allocate("retrieved_context", 2000)
budget.allocate("user_message", 200)
budget.allocate("output_reserve", 2000)
# Remaining: 3300 tokens for conversation history
```

---

<a id="interview-questions"></a>
## 面試題

<a id="q-why-does-gpt-4-struggle-with-simple-character-counting"></a>
### Q：為什麼 GPT-4 連簡單的字元計數都會出錯？

**強回答：**
Tokenization 會把文字轉成 subword 單位，而不是字元。當你問「strawberry 裡有幾個 r？」時，模型看到的可能是 ["str", "aw", "berry"]，而不是單獨字母。

模型必須推理它無法直接觀察到的 token 內部結構。這需要記住或計算 token 的字元組成，而這種能力是 emergent capability，不一定可靠。

解法是先要求模型把單字逐字拼出來，再開始計數。這會強制建立字元層級的 tokens。

<a id="q-how-would-you-estimate-token-count-for-cost-planning"></a>
### Q：你會如何估算 token 數量來做成本規劃？

**強回答：**
若只是粗估：英文文字可用字數乘上 1.3。

若要精準計數：使用模型專屬 tokenizer。
- OpenAI：tiktoken 函式庫
- 其他：transformers AutoTokenizer

重要考量：
- 非英文文字通常會多出 1.5-3 倍 tokens
- 程式碼與結構化資料的 tokenization 效率較差
- 要預留額外 output tokens 預算（通常單價更高）
- 別忘了 system prompts 與 formatting tokens

在正式環境做成本估算時，我會抽樣真實 requests、量測實際 token 使用量，再加上 safety margins。

<a id="q-what-happens-when-switching-tokenizers-between-models"></a>
### Q：在不同模型之間切換 tokenizer 會發生什麼事？

**強回答：**
每個模型家族都有自己的 tokenizer。你不能跨模型重用 tokens，原因如下：

1. **詞彙表不同：**相同 token IDs 代表不同字串
2. **合併規則不同：**同一段文字的切分方式不同
3. **特殊 tokens 不同：**chat formatting 也不同

實務含意：
- 做 token 計數時一定要用正確 tokenizer
- 快取的 embeddings 是模型專屬的
- Prompt templates 需要依模型調整
- Fine-tuned models 會繼承其 base tokenizer

<a id="q-how-do-you-handle-tokenization-for-rag-chunking"></a>
### Q：你會如何處理 RAG chunking 的 tokenization？

**強回答：**
關鍵考量：

1. **在 token 邊界切塊：**若切在 token 中間，decode 後的文字會損壞
2. **把 template tokens 算進去：**system prompt、formatting 都會吃掉 tokens
3. **保留餘裕：**取回的 chunks 加上問題必須能放進 context

實作方式：
```python
# Determine available tokens for chunks
available = max_context - system_prompt_tokens - question_tokens - output_reserve

# Chunk with overlap at token boundaries
chunks = chunk_at_token_boundaries(document, chunk_size=500, overlap=50)

# Select chunks until budget exhausted
selected = []
tokens_used = 0
for chunk in ranked_chunks:
    chunk_tokens = count_tokens(chunk)
    if tokens_used + chunk_tokens <= available:
        selected.append(chunk)
        tokens_used += chunk_tokens
```

---

<a id="references"></a>
## 參考資料

- Sennrich et al. "Neural Machine Translation of Rare Words with Subword Units" (BPE, 2016)
- Wu et al. "Google's Neural Machine Translation System" (WordPiece, 2016)
- Kudo and Richardson "SentencePiece: A simple and language independent subword tokenizer" (2018)
- OpenAI tiktoken library: https://github.com/openai/tiktoken
- HuggingFace tokenizers: https://github.com/huggingface/tokenizers

---

*上一篇：[LLM Internals](01-llm-internals.md) | 下一篇：[Attention Mechanisms](03-attention-mechanisms.md)*

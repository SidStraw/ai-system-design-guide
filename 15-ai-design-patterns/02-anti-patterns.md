<a name="ai-anti-patterns"></a>
# AI 反模式

認識「不該做什麼」與了解最佳實踐同樣重要。本章整理了 AI 系統設計中的常見錯誤。

<a name="table-of-contents"></a>
## 目錄

- [架構反模式](#architecture-anti-patterns)
- [RAG 反模式](#rag-anti-patterns)
- [代理人反模式](#agent-anti-patterns)
- [提示工程反模式](#prompting-anti-patterns)
- [評估反模式](#evaluation-anti-patterns)
- [生產環境反模式](#production-anti-patterns)
- [面試題目](#interview-questions)

---

<a name="architecture-anti-patterns"></a>
## 架構反模式

<a name="the-god-prompt"></a>
### 神級提示（God Prompt）

**問題：** 單一巨大提示試圖處理所有事情。

```python
# ANTI-PATTERN: God Prompt
SYSTEM_PROMPT = """
You are a helpful assistant. You can:
1. Answer questions about our products
2. Help with technical support
3. Process refunds
4. Schedule appointments
5. Translate languages
6. Write code
7. Analyze data
8. Generate reports
... [continues for 5000 tokens]
"""
```

**失敗原因：**
- 指令佔用上下文，擠壓使用者內容的空間
- 模型難以處理相互衝突的指令
- 無法針對所有情境進行最佳化
- 更新會影響所有功能

**解決方案：**
```python
# PATTERN: Specialized components
class QueryRouter:
    async def route(self, query: str) -> str:
        intent = await self.classify_intent(query)
        handler = self.handlers[intent]
        return await handler.process(query)
```

---

<a name="single-provider-dependency"></a>
### 單一供應商依賴

**問題：** 整個系統依賴單一 LLM 供應商。

```python
# ANTI-PATTERN: Single provider
async def generate(prompt: str) -> str:
    return await openai.chat.completions.create(...)
```

**失敗原因：**
- 供應商中斷 = 系統完全失效
- 速率限制影響所有流量
- 缺乏議價籌碼
- 被鎖定在單一模型家族

**解決方案：**
```python
# PATTERN: Multi-provider with failover
class LLMClient:
    def __init__(self):
        self.providers = [OpenAI(), Anthropic(), Google()]
    
    async def generate(self, prompt: str) -> str:
        for provider in self.providers:
            try:
                return await provider.generate(prompt)
            except ProviderError:
                continue
        raise AllProvidersFailedError()
```

---

<a name="premature-fine-tuning"></a>
### 過早微調

**問題：** 在窮盡更簡單的方法之前就進行微調。

**失敗原因：**
- 成本高昂且耗時
- 需要高品質訓練資料（通常不易取得）
- 難以更新與維護
- 通常並非必要

**決策流程：**
```
嘗試提示工程
    ↓ （效果不佳）
嘗試少樣本範例
    ↓ （效果不佳）
嘗試 RAG 補充知識
    ↓ （效果不佳）
考慮微調（需要 500+ 筆範例）
```

---

<a name="rag-anti-patterns"></a>
## RAG 反模式

<a name="retrieve-everything"></a>
### 檢索所有內容

**問題：** 不論相關性，一律檢索大量文件。

```python
# ANTI-PATTERN: Retrieve everything
results = vector_db.search(query, top_k=50)
context = "\n".join([r.text for r in results])
```

**失敗原因：**
- 雜訊淹沒有效信號
- 超出上下文限制
- 在不相關內容上浪費 token
- 「迷失在中間」效應

**解決方案：**
```python
# PATTERN: Quality over quantity
results = vector_db.search(query, top_k=20)
reranked = await reranker.rerank(query, results)
context = "\n".join([r.text for r in reranked[:5] if r.score > 0.7])
```

---

<a name="no-chunking-strategy"></a>
### 缺乏分塊策略

**問題：** 對文件進行任意或無策略的分塊。

```python
# ANTI-PATTERN: Fixed-size blind chunking
chunks = [text[i:i+1000] for i in range(0, len(text), 1000)]
```

**失敗原因：**
- 在句子中間、段落中間截斷
- 失去語意連貫性
- 分離相關資訊
- 檢索品質差

**解決方案：**
```python
# PATTERN: Semantic-aware chunking
chunks = semantic_chunker.chunk(
    text,
    chunk_size=500,
    overlap=100,
    respect_boundaries=["paragraph", "section"]
)
```

---

<a name="ignoring-metadata"></a>
### 忽略元資料

**問題：** 將所有文件視為純文字一視同仁。

```python
# ANTI-PATTERN: Ignore metadata
embedding = embed(document.text)
vector_db.insert(embedding, {"text": document.text})
```

**失敗原因：**
- 無法依日期、來源、類型進行過濾
- 無法對文件進行存取控制
- 無法對新舊文件加權
- 遺失有價值的上下文

**解決方案：**
```python
# PATTERN: Rich metadata
vector_db.insert(embedding, {
    "text": document.text,
    "source": document.source,
    "date": document.date,
    "access_level": document.access_level,
    "document_type": document.type,
    "section": document.section
})

# Filter query
results = vector_db.search(
    query,
    filter={"date": {"$gte": "2024-01-01"}, "access_level": user.level}
)
```

---

<a name="agent-anti-patterns"></a>
## 代理人反模式

<a name="infinite-loop-risk"></a>
### 無限迴圈風險

**問題：** 代理人沒有終止條件。

```python
# ANTI-PATTERN: No limits
while not done:
    action = await agent.decide_action()
    result = await execute(action)
    done = agent.check_done(result)
```

**失敗原因：**
- 代理人可能永遠迴圈
- 成本螺旋式上升
- 永遠不回應使用者
- 資源耗盡

**解決方案：**
```python
# PATTERN: Multiple termination conditions
MAX_STEPS = 20
MAX_COST = 10.0
MAX_TIME = 300  # seconds

for step in range(MAX_STEPS):
    if cost_tracker.total > MAX_COST:
        return "Cost limit reached"
    if time.time() - start > MAX_TIME:
        return "Time limit reached"
    
    action = await agent.decide_action()
    result = await execute(action)
    
    if agent.check_done(result):
        return result
    
return "Step limit reached"
```

---

<a name="unsafe-tool-access"></a>
### 不安全的工具存取

**問題：** 賦予代理人不受限制的工具存取權。

```python
# ANTI-PATTERN: Full access
tools = [
    delete_file,
    execute_shell_command,
    send_email,
    database_query  # unrestricted!
]
```

**失敗原因：**
- 代理人可能刪除關鍵檔案
- 可能洩漏資料
- 可能執行惡意指令
- 無稽核記錄

**解決方案：**
```python
# PATTERN: Scoped, validated tools
tools = [
    ScopedFileTool(allowed_dirs=["/tmp/agent"]),
    RestrictedShellTool(allowed_commands=["ls", "cat"]),
    EmailTool(requires_confirmation=True),
    ReadOnlyDatabaseTool(allowed_tables=["products"])
]
```

---

<a name="agent-without-memory"></a>
### 沒有記憶的代理人

**問題：** 代理人每輪對話都從零開始。

```python
# ANTI-PATTERN: Stateless agent
async def handle_message(message: str) -> str:
    return await agent.run(message)  # No context
```

**失敗原因：**
- 無法執行多輪任務
- 重複相同的錯誤
- 無法從經驗中學習
- 使用者體驗差

**解決方案：**
```python
# PATTERN: Persistent memory
async def handle_message(session_id: str, message: str) -> str:
    memory = await memory_store.get(session_id)
    response = await agent.run(message, memory=memory)
    await memory_store.update(session_id, memory)
    return response
```

---

<a name="prompting-anti-patterns"></a>
## 提示工程反模式

<a name="vague-instructions"></a>
### 模糊指令

**問題：** 模糊的提示卻期待特定行為。

```python
# ANTI-PATTERN: Vague
prompt = "Help the user with their request."
```

**失敗原因：**
- 「幫助」定義不明
- 未指定格式
- 沒有邊界
- 行為不一致

**解決方案：**
```python
# PATTERN: Specific and structured
prompt = """
You are a customer support agent for TechCorp.

Your role:
- Answer questions about our products
- Help troubleshoot issues
- Escalate to human when unsure

Response format:
1. Acknowledge the issue
2. Provide a solution or ask clarifying questions
3. Offer next steps

Do NOT:
- Make promises about refunds (escalate instead)
- Provide legal or medical advice
- Share internal company information
"""
```

---

<a name="no-output-format"></a>
### 沒有輸出格式

**問題：** 期待結構化輸出卻未指定格式。

```python
# ANTI-PATTERN: Hope for structure
prompt = "Extract the person's name, date, and location from this text."
response = await llm.generate(prompt)
# Response: "The person is John, he was there on March 5th in NYC"
# Now try to parse that...
```

**解決方案：**
```python
# PATTERN: Explicit format
prompt = """
Extract information and return as JSON:
{
    "name": "string",
    "date": "YYYY-MM-DD",
    "location": "string"
}

Text: ...
"""
# Or use structured output APIs
response = await llm.generate(prompt, response_format={"type": "json_object"})
```

---

<a name="evaluation-anti-patterns"></a>
## 評估反模式

<a name="vibes-based-evaluation"></a>
### 感覺式評估

**問題：** 以「看起來不錯」作為評估方法。

```python
# ANTI-PATTERN: Manual spot-checking
for i in range(5):
    response = await generate(test_prompts[i])
    print(response)  # Developer looks at it
# "Looks good, ship it!"
```

**失敗原因：**
- 不可重現
- 人為挑選範例
- 缺乏基準比較
- 遺漏邊緣案例

**解決方案：**
```python
# PATTERN: Systematic evaluation
eval_dataset = load_eval_set()  # 100+ examples
results = []

for example in eval_dataset:
    response = await generate(example["input"])
    score = await evaluate(response, example["expected"])
    results.append(score)

metrics = {
    "accuracy": sum(results) / len(results),
    "failures": [e for e, r in zip(eval_dataset, results) if r < 0.5]
}
```

---

<a name="training-on-test-set"></a>
### 在測試集上訓練

**問題：** 將評估資料用於開發決策。

```python
# ANTI-PATTERN: Overfitting to eval
for iteration in range(100):
    accuracy = evaluate_on_test_set()  # Same set every time
    tweak_prompt_based_on_failures(test_set)  # Optimizing for test set
```

**失敗原因：**
- 過擬合特定範例
- 真實世界表現與評估結果不符
- 無法衡量真正的泛化能力

**解決方案：**
```python
# PATTERN: Proper data splits
dev_set = load_dev_set()      # For iteration
test_set = load_test_set()    # Final evaluation only

# Iterate on dev set
for iteration in range(100):
    accuracy = evaluate(dev_set)
    improve_based_on(dev_set)

# Final evaluation on untouched test set
final_accuracy = evaluate(test_set)
```

---

<a name="production-anti-patterns"></a>
## 生產環境反模式

<a name="no-rate-limiting"></a>
### 沒有速率限制

**問題：** 不限制每位使用者的 LLM 呼叫次數。

```python
# ANTI-PATTERN: Open access
@app.route("/generate")
async def generate():
    return await llm.generate(request.prompt)  # No limits!
```

**失敗原因：**
- 單一使用者可耗盡預算
- 拒絕服務風險
- 成本驚喜
- 缺乏公平使用機制

**解決方案：**
```python
# PATTERN: Rate limiting
@app.route("/generate")
@rate_limit(requests_per_minute=10, requests_per_day=100)
@cost_limit(max_cost_per_day=1.0)
async def generate():
    return await llm.generate(request.prompt)
```

---

<a name="no-caching"></a>
### 沒有快取

**問題：** 每個相同的請求都打到 LLM。

```python
# ANTI-PATTERN: No cache
async def answer_faq(question: str) -> str:
    return await llm.generate(question)  # Same FAQ, same cost every time
```

**失敗原因：**
- 相同查詢浪費金錢
- 不必要的延遲
- 相同問題得到不一致的答案

**解決方案：**
```python
# PATTERN: Semantic caching
async def answer_faq(question: str) -> str:
    cached = await cache.get_similar(question, threshold=0.95)
    if cached:
        return cached.response
    
    response = await llm.generate(question)
    await cache.set(question, response)
    return response
```

---

<a name="interview-questions"></a>
## 面試題目

<a name="q-what-is-the-biggest-anti-pattern-you-see-in-llm-applications"></a>
### 問：您認為 LLM 應用中最大的反模式是什麼？

**優質回答範例：**

「危害最大的是「神級提示」反模式：用單一巨大提示試圖處理所有情境。

**為何常見：** 從一個提示開始，隨著需求增加不斷添加指令，看起來更簡單。

**失敗原因：**
- 指令佔用上下文，擠壓使用者內容的空間
- 相互衝突的指令讓模型混淆
- 無法針對不同使用情境最佳化
- 修改會產生不可預期的副作用

**解決方法：** 路由至專門的處理器。每個處理器都有針對單一任務最佳化的聚焦提示。路由器本身可以很簡單（基於關鍵字）或更智慧（對複雜情況使用 LLM）。

這個原則不只適用於提示。通用原則是：將複雜性分解為專門的元件，而非將一切塞入單一巨型系統。」

<a name="q-how-do-you-avoid-agent-runaway-costs"></a>
### 問：如何避免代理人失控的成本？

**優質回答範例：**

「在不同層級設置多重限制：

**每次請求的限制：**
- 最大步驟數（例如 20 步）
- 最大 token 數（例如 50K）
- 最大時間（例如 5 分鐘）

**每個工作階段的限制：**
- 每日 token 預算
- 每日費用上限

**每位使用者的限制：**
- 速率限制（每分鐘／小時／天的請求數）
- 成本歸因與上限

**監控：**
- 即時成本追蹤
- 異常警報（單次請求超過 $1）
- 成本飆升時的斷路器

**架構：**
- 從便宜到昂貴的模型進行層疊
- 快取常見操作
- 批次處理相似請求

關鍵在於假設代理人會試圖永遠運行。在每個層級都設置硬性停止條件。我曾見過代理人在沒有適當限制的情況下，在幾分鐘內累積高達 $1,000 的帳單。」

---

*上一章：[設計模式](01-design-patterns.md)*

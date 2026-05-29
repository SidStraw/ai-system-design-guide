<a name="chapter-5"></a>
## 第五章：基於程式碼的評估器

### 什麼是基於程式碼的評估？

基於程式碼的評估是指你**以程式碼撰寫的檢查**（例如 Python），用來驗證 AI 輸出的特定、客觀屬性。

### 何時使用基於程式碼的評估

**在不需要呼叫 LLM 就能測試的情況下，使用程式碼：**

1. **格式驗證** - Markdown 是否出現在文字訊息中？
2. **必要欄位檢查** - AI 是否包含所有必要資訊？
3. **工具呼叫驗證** - AI 是否呼叫了正確的工具？
4. **回應長度限制** - 回應是否在 500 個字元以內？
5. **禁止內容模式** - 是否有 PII（電子郵件、電話號碼）？

### 範例一：檢查文字訊息中的 Markdown

```python
import re

def eval_no_markdown_in_sms(trace) -> dict:
    response = trace['assistant_message']
    channel = trace['metadata']['channel']

    if channel != 'sms':
        return {'passed': True, 'reason': 'Not SMS'}

    markdown_patterns = [
        r'\*\*.*?\*\*',  # Bold
        r'\_\_.*?\_\_',   # Bold alt
        r'\#\#\s',        # Headers
        r'```',           # Code blocks
        r'\[.*?\]\(.*?\)'  # Links
    ]

    for pattern in markdown_patterns:
        if re.search(pattern, response):
            return {
                'passed': False,
                'reason': f'Found markdown pattern: {pattern}'
            }

    return {'passed': True, 'reason': 'No markdown found'}
```

**平台整合：**

**使用 LangWatch：**

```python
import langwatch

# Register as custom evaluator
@langwatch.evaluator(name="no_markdown_sms")
def eval_no_markdown_in_sms(trace):
    # ... implementation above ...
    return {'passed': result['passed'], 'score': 1.0 if result['passed'] else 0.0}

# Run on dataset
results = langwatch.evaluate.batch(
    dataset=traces_df,
    evaluators=["no_markdown_sms"]
)
```

**使用 Langfuse：**

```python
from langfuse import get_client

langfuse = get_client()

# Run on each trace and log scores
for trace in traces:
    result = eval_no_markdown_in_sms(trace)
    
    langfuse.create_score(
        trace_id=trace.id,
        name="no_markdown_sms",
        value=1 if result['passed'] else 0,
        data_type="BOOLEAN",
        comment=result['reason']
    )
```

### 範例二：驗證工具呼叫

```python
def eval_correct_tool_called(trace) -> dict:
    user_message = trace['user_message'].lower()
    tool_calls = trace['tool_calls']

    rules = {
        'availability': ['available', 'vacant', 'open units'],
        'schedule_tour': ['tour', 'visit', 'see'],
        'get_price': ['price', 'rent', 'cost', 'how much']
    }

    expected_tool = None
    for tool, keywords in rules.items():
        if any(keyword in user_message for keyword in keywords):
            expected_tool = tool
            break

    if not expected_tool:
        return {'passed': True, 'reason': 'No specific tool expected'}

    called_tools = [call['function'] for call in tool_calls]

    if expected_tool in called_tools:
        return {'passed': True, 'reason': f'Correctly called {expected_tool}'}
    else:
        return {
            'passed': False,
            'reason': f'Expected {expected_tool}, called {called_tools}'
        }
```

### 範例三：驗證看房確認中的必要資訊

```python
import re

def eval_tour_confirmation_complete(trace) -> dict:
    response = trace['assistant_message'].lower()

    if 'tour' not in response and 'visit' not in response:
        return {'passed': True, 'reason': 'Not a tour confirmation'}

    required_elements = {'date': False, 'time': False, 'address': False}

    date_patterns = [
        r'\d{1,2}/\d{1,2}/\d{4}',
        r'\d{1,2}-\d{1,2}-\d{4}',
        r'(mon|tues|wednes|thurs|fri|satur|sun)day'
    ]
    if any(re.search(p, response) for p in date_patterns):
        required_elements['date'] = True

    time_patterns = [r'\d{1,2}:\d{2}\s?(am|pm)', r'\d{1,2}\s?(am|pm)']
    if any(re.search(p, response) for p in time_patterns):
        required_elements['time'] = True

    if 'street' in response or 'ave' in response or 'unit' in response:
        required_elements['address'] = True

    missing = [k for k, v in required_elements.items() if not v]

    if not missing:
        return {'passed': True, 'reason': 'All required elements present'}
    else:
        return {'passed': False, 'reason': f'Missing: {", ".join(missing)}'}
```

### 基於程式碼評估的優點

1. **快速** - 無需 API 呼叫，即時得到結果
2. **低成本** - 不消耗 token
3. **確定性** - 相同輸入永遠產生相同輸出
4. **易於除錯** - 堆疊追蹤、中斷點可正常運作
5. **無幻覺** - 程式碼完全按照你的指示執行

### 結合基於程式碼與基於 LLM 的評估

一套完整的評估套件通常包含：
- **2–3 個基於程式碼的評估**，用於客觀檢查
- **1–2 個基於 LLM 的評估**，用於主觀判斷

```python
# Code-based evals (fast, cheap, deterministic)
1. check_no_markdown_in_sms()
2. validate_tool_calls()
3. check_response_length()

# LLM-based evals (slower, but handles nuance)
4. evaluate_dietary_adherence()
5. evaluate_response_helpfulness()
```

**混合評估套件的平台比較：**

**LangWatch 方式（統一）：**
```python
import langwatch

# All evaluators registered in one place
langwatch.evaluate.batch(
    dataset=traces_df,
    evaluators=[
        "no_markdown_sms",  # Code-based (custom)
        "tool_validation",   # Code-based (custom)
        "dietary_compliance", # LLM-based (built-in)
        "helpfulness"        # LLM-based (built-in)
    ]
)
```

**Langfuse 方式（靈活但需手動）：**
```python
# Run code-based evals
for trace in traces:
    markdown_result = eval_no_markdown_in_sms(trace)
    tool_result = eval_correct_tool_called(trace)
    
    # Log code-based scores
    langfuse.create_score(trace_id=trace.id, name="markdown", ...)
    langfuse.create_score(trace_id=trace.id, name="tools", ...)

# Run LLM-based evals separately
llm_results = run_llm_judges(traces)
for result in llm_results:
    langfuse.create_score(trace_id=result.trace_id, ...)
```

### 測試你的基於程式碼評估

**務必以已知的正確和錯誤案例測試你的評估：**

```python
def test_no_markdown_evaluator():
    eval = NoMarkdownEvaluator()

    # Test case 1: Clean SMS
    clean_trace = {
        'assistant_message': 'Your tour is scheduled for 2PM',
        'metadata': {'channel': 'sms'}
    }
    result = eval.evaluate(clean_trace)
    assert result.passed == True

    # Test case 2: SMS with markdown
    markdown_trace = {
        'assistant_message': 'Your tour is **confirmed** for 2PM',
        'metadata': {'channel': 'sms'}
    }
    result = eval.evaluate(markdown_trace)
    assert result.passed == False

    # Test case 3: Email (should pass, we don't check email)
    email_trace = {
        'assistant_message': 'Your tour is **confirmed**',
        'metadata': {'channel': 'email'}
    }
    result = eval.evaluate(email_trace)
    assert result.passed == True

    print("All tests passed!")
```

---

<a name="chapter-6"></a>
## 第六章：RAG 系統評估

### 什麼是 RAG？

**RAG（Retrieval Augmented Generation，檢索增強生成）**意味著你的 AI：
1. 從資料庫中**檢索**相關資訊
2. **利用該資訊**生成回應

### 為何 RAG 需要特殊評估

RAG 有**兩種失敗模式：**

1. **檢索失敗** - 未能找到正確資訊
2. **生成失敗** - 使用資訊的方式有誤

你需要**分別**評估兩者，才能知道問題出在哪裡。

### 建立 BM25 檢索引擎

在為特定領域（例如食譜）建立基於關鍵字的檢索時，有一個關鍵洞察：**你的分詞器至關重要**。

#### 針對特定領域內容的自訂分詞器

```python
import re

# Preserves numbers, temperatures, measurements
_TOKEN_RE = re.compile(
    r"\d+\s*[x x]\s*\d+"      # Dimensions like 9x13
    r"|(?:\d+/?\d+)"           # Fractions like 1/2
    r"|(?:\d+(?:\.\d+)?)"      # Numbers like 375
    r"|(?:degrees[fc])"           # Temperature units
    r"|[a-z]+"                  # Regular words
)

def tokenize(text: str) -> list[str]:
    s = (text or "").lower()
    # Normalize temperature references
    s = s.replace("degrees f", "degreesf").replace("degree f", "degreesf")
    s = s.replace("mins", "min").replace("minutes", "min")
    return _TOKEN_RE.findall(s)
```

**為何這很重要：** 標準分詞器會去除數字。但在食譜中，「375」（溫度）、「9x13」（烤盤尺寸）和「1/2」（用量）都是關鍵搜尋詞。

### 為 RAG 測試生成合成查詢

與其手動撰寫測試查詢，不如使用 LLM 生成依賴文件中特定事實的查詢：

```python
SYSTEM_PROMPT = """You are an advanced user of a recipe search engine.
Given a recipe, write ONE realistic cooking question that depends on
a precise, technical detail contained in THIS recipe. Focus on:
1) Specific methods (e.g., marinate 4 hours, bake at 375 degrees F)
2) Appliance settings (e.g., air fryer 400 degrees F for 12 minutes)
3) Ingredient prep details (e.g., slice onions paper-thin)
4) Timing specifics (e.g., rest dough 30 minutes)
5) Temperature precision (e.g., internal 165 degrees F)

Return EXACTLY a single JSON object:
{"query": "...?", "salient_fact": "<exact quote or paraphrase>"}"""
```

這會生成如下查詢：
- 「薑餅城堡餅乾應該在幾度烘烤？」（顯著事實：「350 degrees F for 8-10 minutes」）
- 「麵包麵團需要發酵多久？」（顯著事實：「rise for 1 hour until doubled」）

`salient_fact` 是你的基準真相——你知道哪道食譜有答案。

### 評估檢索品質

#### Recall@K

「正確食譜是否出現在前 K 個結果中？」

```python
def recall_at_k(k, output, metadata, **kwargs):
    """Check if ground-truth recipe is in top-k results"""
    ground_truth_id = metadata.get("source_recipe_id")
    if not ground_truth_id:
        return 0.0

    top_ids = output.get("top_ids", [])
    for rank, doc_id in enumerate(top_ids, 1):
        if str(doc_id) == str(ground_truth_id):
            return 1.0 if rank <= k else 0.0
    return 0.0

# Create specific evaluators
def RecallAt1(**kwargs): return recall_at_k(1, **kwargs)
def RecallAt3(**kwargs): return recall_at_k(3, **kwargs)
def RecallAt5(**kwargs): return recall_at_k(5, **kwargs)
```

#### 平均倒數排名（MRR）

「如果找到了，它排在第幾位？」

```python
def MRR(output, metadata, **kwargs):
    ground_truth_id = metadata.get("source_recipe_id")
    if not ground_truth_id:
        return 0.0

    top_ids = output.get("top_ids", [])
    for rank, doc_id in enumerate(top_ids, 1):
        if str(doc_id) == str(ground_truth_id):
            return 1.0 / rank
    return 0.0
```

### 執行 RAG 實驗

#### 使用 LangWatch

```python
import langwatch

def bm25_task(example):
    query = example["input"]["input"]
    hits = retrieve_bm25(query, corpus, bm25, tokenized_corpus, top_n=5)
    return {"top_ids": [h["id"] for h in hits], "top_titles": [h["title"] for h in hits]}

# Register custom metrics
@langwatch.metric(name="recall_at_1")
def recall_at_1_metric(output, expected):
    return recall_at_k(1, output, expected)

# Run experiment
results = langwatch.evaluate.batch(
    dataset=synthetic_queries_dataset,
    task=bm25_task,
    metrics=["recall_at_1", "recall_at_3", "recall_at_5", "mrr"]
)
```

**LangWatch 優勢：** 內建 RAG 指標，自動視覺化檢索效能。

#### 使用 Langfuse

```python
from langfuse import Evaluation

def bm25_task(*, item, **kwargs):
    query = item["input"]["query"]
    hits = retrieve_bm25(query, corpus, bm25, tokenized_corpus, top_n=5)
    return {"top_ids": [h["id"] for h in hits], "top_titles": [h["title"] for h in hits]}

def recall_at_1_eval(*, output, expected_output, **kwargs):
    ground_truth_id = expected_output.get("source_recipe_id")
    found = str(ground_truth_id) in [str(x) for x in output.get("top_ids", [])[:1]]
    return Evaluation(name="recall@1", value=1.0 if found else 0.0)

result = langfuse.run_experiment(
    name="bm25-retrieval",
    data=synthetic_queries_data,
    task=bm25_task,
    evaluators=[recall_at_1_eval],
)
```

### 診斷 RAG 失敗

當 RAG 測試失敗時，診斷**失敗所在：**

```python
def diagnose_rag_failure(query, target_recipe_id, retriever, pipeline):
    # Step 1: Check retrieval
    retrieved = retriever.search(query, k=5)
    retrieved_ids = [d.id for d in retrieved]

    if target_recipe_id not in retrieved_ids:
        return {'failure_point': 'RETRIEVAL',
                'issue': f'Recipe not in top 5'}

    # Step 2: Check document quality
    correct_doc = [d for d in retrieved if d.id == target_recipe_id][0]
    # Does the doc actually contain the answer?

    # Step 3: Check generation
    answer = pipeline(query, retrieved)
    is_correct = eval_factual_correctness(query, retrieved, answer)

    if not is_correct:
        return {'failure_point': 'GENERATION',
                'issue': 'Answer incorrect despite good retrieval'}

    return {'failure_point': None, 'status': 'PASS'}
```

### 改善 RAG 效能

**當檢索失敗時：**
1. 嘗試不同的分塊策略
2. 新增元資料過濾器
3. 使用混合搜尋（關鍵字 + 語意）
4. 實作查詢擴展
5. 嘗試重新排序模型
6. 使用特定領域的分詞器（例如上述保留數字的分詞器）

**當生成失敗時：**
1. 改善系統提示詞
2. 新增少量範例（few-shot examples）
3. 使用思維鏈（chain-of-thought）提示
4. 新增明確的依據指示
5. 實作引用要求

---


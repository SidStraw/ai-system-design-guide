<a name="chapter-5"></a>
## Chapter 5: Code-Based Evaluators

### What Are Code-Based Evals?

Code-based evals are **checks you write in programming code** (like Python) to verify specific, objective properties of your AI's outputs.

### When to Use Code-Based Evals

**Use code when you can test something without calling an LLM:**

1. **Format validation** - Is markdown appearing in text messages?
2. **Required field checks** - Did the AI include all required information?
3. **Tool call validation** - Did the AI call the right tool?
4. **Response length constraints** - Is the response under 500 characters?
5. **Prohibited content patterns** - Are there PII (emails, phone numbers)?

### Example 1: Check for Markdown in Text Messages

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

**Platform Integration:**

**With LangWatch:**

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

**With Langfuse:**

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

### Example 2: Validate Tool Calls

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

### Example 3: Validate Required Information in Tour Confirmations

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

### Benefits of Code-Based Evals

1. **Fast** - No API calls, instant results
2. **Cheap** - No tokens used
3. **Deterministic** - Same input always gives same output
4. **Easy to debug** - Stack traces, breakpoints work normally
5. **No hallucination** - Code does exactly what you tell it

### Combining Code-Based and LLM-Based Evals

A complete eval suite typically has:
- **2-3 code-based evals** for objective checks
- **1-2 LLM-based evals** for subjective judgments

```python
# Code-based evals (fast, cheap, deterministic)
1. check_no_markdown_in_sms()
2. validate_tool_calls()
3. check_response_length()

# LLM-based evals (slower, but handles nuance)
4. evaluate_dietary_adherence()
5. evaluate_response_helpfulness()
```

**Platform Comparison for Mixed Eval Suites:**

**LangWatch approach (unified):**
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

**Langfuse approach (flexible but manual):**
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

### Testing Your Code-Based Evals

**Always test your evals with known good and bad cases:**

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
## Chapter 6: RAG System Evaluation

### What is RAG?

**RAG (Retrieval Augmented Generation)** means your AI:
1. **Retrieves** relevant information from a database
2. **Uses that information** to generate a response

### Why RAG Needs Special Evaluation

RAG has **two failure modes:**

1. **Retrieval fails** - Doesn't find the right information
2. **Generation fails** - Uses the information wrong

You need to evaluate **both** separately to know where problems occur.

### Building a BM25 Retrieval Engine

When building keyword-based retrieval for a domain like recipes, here's the key insight: **your tokenizer matters**.

#### Custom Tokenizer for Domain-Specific Content

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

**Why this matters:** Standard tokenizers strip numbers. But in recipes, "375" (temperature), "9x13" (pan size), and "1/2" (measurement) are critical search terms.

### Generating Synthetic Queries for RAG Testing

Instead of manually writing test queries, use an LLM to generate queries that depend on specific facts in your documents:

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

This generates queries like:
- "What temperature should I bake the gingerbread castle cookies at?" (salient fact: "350 degrees F for 8-10 minutes")
- "How long should I let the bread dough rise?" (salient fact: "rise for 1 hour until doubled")

The `salient_fact` is your ground truth - you know which recipe has the answer.

### Evaluating Retrieval Quality

#### Recall@K

"Did the correct recipe appear in the top K results?"

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

#### Mean Reciprocal Rank (MRR)

"If we found it, how high did it rank?"

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

### Running RAG Experiments

#### With LangWatch

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

**LangWatch advantage:** Built-in RAG metrics, automatic visualization of retrieval performance.

#### With Langfuse

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

### Diagnosing RAG Failures

When a RAG test fails, diagnose WHERE:

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

### Improving RAG Performance

**When retrieval fails:**
1. Try different chunking strategies
2. Add metadata filters
3. Use hybrid search (keyword + semantic)
4. Implement query expansion
5. Try reranking models
6. Use domain-specific tokenizers (like the number-preserving one above)

**When generation fails:**
1. Improve system prompt
2. Add few-shot examples
3. Use chain-of-thought prompting
4. Add explicit grounding instructions
5. Implement citation requirements

---


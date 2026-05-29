<a name="chapter-4"></a>
## Chapter 4: Building LLM-as-a-Judge Evaluators

### What is LLM-as-a-Judge?

An **LLM judge** is an AI that evaluates other AI outputs. It reads traces and scores them.

**Why use it?**
- Automates evaluation at scale
- Provides consistent judgment
- Much faster than manual review

**The challenge:**
Most people build judges wrong. Their judges hallucinate, miss problems, or create false confidence.

### When to Use LLM-as-a-Judge

**Use LLM judges for:**
- Subjective quality assessments
- Policy compliance checking
- Context understanding
- Dietary adherence
- Tone appropriateness
- Multi-step reasoning checks

**Don't use LLM judges for:**
- Format validation (use code)
- Required field checks (use code)
- Simple pattern matching (use code)
- Exact string matching (use code)

**Rule of thumb:** If you can express it as an if/else statement, use code. If you need judgment, use LLM.

### The Complete LLM Judge Workflow

Building reliable LLM judges requires a rigorous 7-step workflow:

#### Overview: The Pipeline

```
1. Generate traces (run your AI on test queries)
2. Label a subset manually (or with a powerful LLM)
3. Split into Train / Dev / Test sets
4. Develop your judge prompt using Train examples
5. Validate on Dev set (iterate until good)
6. Final evaluation on Test set (unbiased metrics)
7. Run on all traces + correct with judgy
```

### Step 1: Generate Traces

Run your AI system on diverse test queries to create traces. Use your platform's auto-instrumentation (see Chapter 2) to capture everything automatically.

### Step 2: Label Ground Truth Data

Label 150-200 traces as PASS or FAIL. You can do this manually (most accurate) or use a powerful LLM:

```
You are an expert nutritionist evaluating dietary adherence.

DIETARY RESTRICTION DEFINITIONS:
- Vegan: No animal products (meat, dairy, eggs, honey, etc.)
- Vegetarian: No meat or fish, but dairy and eggs are allowed
- Gluten-free: No wheat, barley, rye, or other gluten-containing grains
- Keto: Very low carb (<20g net carbs), high fat, moderate protein
[... full definitions — see Appendix C for the full list ...]

EVALUATION CRITERIA:
- PASS: Recipe clearly adheres to the dietary preferences
- FAIL: Recipe contains ingredients that violate dietary preferences

Query: {query}
Dietary Restriction: {dietary_restriction}
Response: {response}

Return JSON: {"label": "PASS" or "FAIL", "explanation": "..."}
```

**Platform-Specific Labeling:**

**With LangWatch (built-in evaluators):**

```python
import langwatch

# LangWatch has 40+ built-in evaluators including dietary compliance
results = langwatch.evaluate.batch(
    dataset=traces_df,
    evaluators=["dietary_compliance"],  # Built-in evaluator
    model="gpt-4o"
)

# Or create custom evaluator
custom_evaluator = langwatch.evaluators.create(
    name="dietary_adherence",
    prompt=LABELING_PROMPT,
    model="gpt-4o"
)

results = langwatch.evaluate.batch(
    dataset=traces_df,
    evaluators=[custom_evaluator]
)
```

**With Langfuse (custom implementation):**

```python
from langfuse.openai import OpenAI

client = OpenAI()

labels = []
for trace in traces:
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": LABELING_PROMPT.format(**trace)}],
        temperature=0
    )
    labels.append(parse_json(response.choices[0].message.content))
```

**LangWatch advantage:** 40+ built-in evaluators save time for common use cases.
**Langfuse advantage:** Complete control over custom evaluation logic.

### Step 3: Split Data (Train / Dev / Test)

This is critical and often skipped! You need three separate sets:

- **Train (~15%):** Used to select few-shot examples for your judge prompt
- **Dev (~40%):** Used to iterate and improve your judge prompt
- **Test (~45%):** Used ONCE for final, unbiased evaluation

```python
from sklearn.model_selection import train_test_split

# First split: separate test set
train_dev, test = train_test_split(
    labeled_data, test_size=0.45,
    stratify=labeled_data['label'],  # Maintain PASS/FAIL ratio
    random_state=42
)

# Second split: separate train from dev
train, dev = train_test_split(
    train_dev, test_size=0.73,  # 40% of original
    stratify=train_dev['label'],
    random_state=42
)
```

**Why stratified splitting?** You need both PASS and FAIL examples in every split. Without stratification, you might get a dev set with all PASS examples, making it useless for testing failure detection.

### Step 4: Build Your Judge Prompt

Your judge prompt needs **four key parts:**

#### Part 1: Role and Domain Definitions

```
You are an expert nutritionist and dietary specialist evaluating
whether recipe responses properly adhere to specified dietary
restrictions.

DIETARY RESTRICTION DEFINITIONS:
- Vegan: No animal products (meat, dairy, eggs, honey, etc.)
- Vegetarian: No meat or fish, but dairy and eggs are allowed
- Gluten-free: No wheat, barley, rye, or other gluten-containing grains
- Dairy-free: No milk, cheese, butter, yogurt, or other dairy products
- Keto: Very low carb (typically <20g net carbs), high fat
- Paleo: No grains, legumes, dairy, refined sugar, or processed foods
[... all 16 definitions — see Appendix C for the full list ...]
```

#### Part 2: Clear Evaluation Criteria

```
EVALUATION CRITERIA:
- PASS: The recipe clearly adheres to the dietary preferences
  with appropriate ingredients and preparation methods
- FAIL: The recipe contains ingredients or methods that violate
  the dietary preferences
- Consider both explicit ingredients AND cooking methods
```

#### Part 3: Few-Shot Examples (From Your Train Set!)

This is where the train set pays off. Select 1-3 examples of correct judgments:

```
Example 1 (PASS):
Query: "Gluten-free pizza dough that actually tastes good"
Response: [Recipe using gluten-free all-purpose flour blend,
  baking powder, olive oil, honey, apple cider vinegar...]
Explanation: The recipe uses gluten-free flour blend. All other
  ingredients (baking powder, salt, olive oil, honey) do not
  contain gluten. The preparation method does not introduce any
  gluten-containing elements.
Label: PASS

Example 2 (FAIL):
Query: "Raw vegan Mediterranean quinoa salad"
Response: [Recipe with cooked quinoa, fresh vegetables,
  olive oil, lemon juice...]
Explanation: The recipe FAILS because it includes cooked quinoa.
  Raw vegan diets do not allow foods heated above 118 degrees F (48 degrees C),
  and cooking quinoa involves boiling, which exceeds this limit.
Label: FAIL
```

#### Part 4: Output Format

```
Now evaluate the following:
Query: {query}
Dietary Restriction: {dietary_restriction}
Recipe Response: {response}

RETURN YOUR EVALUATION IN JSON FORMAT:
"label": "PASS" or "FAIL",
"explanation": "Detailed explanation citing specific ingredients or methods"
```

### Why Binary Scores Work Best

**Some people want 1-5 scales or percentages. Don't do this.**

**With binary (PASS/FAIL):**
- Only need to verify two things
- Clear decision boundary
- Easier to debug
- Simpler to explain to stakeholders

**With 1-5 scale:**
- Need to verify every score aligns
- What's the difference between 2 and 3?
- 5x more work to validate
- Business decisions are binary anyway

**Remember:** Either you fix something or you don't. Either it's broken or it's not.

### Step 5: Validate on Dev Set

Run your judge on the Dev set and compare to ground truth. Here's how with each platform:

#### Evaluator Functions (Platform-Agnostic)

```python
def eval_tp(*, output, expected, **kwargs):
    """True Positive: Judge says PASS, ground truth is PASS"""
    judge = output.get("label", "").upper()
    truth = expected.get("label", "").upper()
    return 1.0 if judge == "PASS" and truth == "PASS" else 0.0

def eval_tn(*, output, expected, **kwargs):
    """True Negative: Judge says FAIL, ground truth is FAIL"""
    judge = output.get("label", "").upper()
    truth = expected.get("label", "").upper()
    return 1.0 if judge == "FAIL" and truth == "FAIL" else 0.0

def eval_fp(*, output, expected, **kwargs):
    """False Positive: Judge says PASS, ground truth is FAIL"""
    judge = output.get("label", "").upper()
    truth = expected.get("label", "").upper()
    return 1.0 if judge == "PASS" and truth == "FAIL" else 0.0

def eval_fn(*, output, expected, **kwargs):
    """False Negative: Judge says FAIL, ground truth is PASS"""
    judge = output.get("label", "").upper()
    truth = expected.get("label", "").upper()
    return 1.0 if judge == "FAIL" and truth == "PASS" else 0.0
```

#### Running the Experiment

**With LangWatch:**

```python
import langwatch

# Create custom evaluator with your judge prompt
judge_evaluator = langwatch.evaluators.create(
    name="dietary-judge-v1",
    prompt=judge_prompt_template,
    model="gpt-4o",
    temperature=0
)

# Run on dev set
results = langwatch.evaluate.batch(
    dataset=dev_dataset,
    evaluators=[judge_evaluator],
    metrics=["tp", "tn", "fp", "fn", "tpr", "tnr"]
)

# LangWatch automatically calculates TPR and TNR
print(f"TPR: {results.metrics['tpr']:.1%}")
print(f"TNR: {results.metrics['tnr']:.1%}")
```

**LangWatch advantage:** Built-in metric calculation, no need to manually compute confusion matrix.

**With Langfuse:**

```python
from langfuse import Evaluation

def accuracy_evaluator(*, input, output, expected_output, **kwargs):
    judge = output.get("label", "").upper()
    truth = expected_output.get("label", "").upper()
    correct = judge == truth
    return Evaluation(name="accuracy", value=1.0 if correct else 0.0)

result = langfuse.run_experiment(
    name="judge-dev-validation",
    data=dev_data,  # list of {"input": ..., "expected_output": ...}
    task=judge_task,
    evaluators=[accuracy_evaluator],
)

print(result.format())

# Calculate TPR/TNR manually from results
tp = sum(1 for r in results if r["judge"] == "PASS" and r["truth"] == "PASS")
tn = sum(1 for r in results if r["judge"] == "FAIL" and r["truth"] == "FAIL")
fp = sum(1 for r in results if r["judge"] == "PASS" and r["truth"] == "FAIL")
fn = sum(1 for r in results if r["judge"] == "FAIL" and r["truth"] == "PASS")

tpr = tp / (tp + fn) if (tp + fn) > 0 else 0
tnr = tn / (tn + fp) if (tn + fp) > 0 else 0

print(f"TPR: {tpr:.1%}")
print(f"TNR: {tnr:.1%}")
```

**Langfuse advantage:** More control over evaluation logic, better for complex custom metrics.

### The Metrics That Actually Matter

**Most people only look at "agreement":**

```
Agreement = (Judge agrees with me) / (Total traces)
Example: 90% agreement
```

**Why this is misleading:**

If failures only happen 10% of the time, a judge that always says "pass" gets 90% accuracy by being completely useless!

**The two metrics you actually need:**

#### 1. TPR (True Positive Rate) - Recall

**"When there's actually a PASS, how often does the judge correctly say PASS?"**

```
TPR = True Positives / (True Positives + False Negatives)
```

#### 2. TNR (True Negative Rate) - Specificity

**"When there's actually a FAIL, how often does the judge correctly say FAIL?"**

```
TNR = True Negatives / (True Negatives + False Positives)
```

### Real Results: Why Iteration Matters

**After careful prompt iteration (production-quality judge):**

```
Test Set Performance:
  True Positive Rate (TPR): 95.7%
  True Negative Rate (TNR): 100.0%
  Balanced Accuracy: 97.8%
  Total predictions: 33
  Correct predictions: 32
  Overall Accuracy: 97.0%
```

**First attempt (before iteration):**

```
Test Set Performance:
  True Positive Rate (TPR): 90.1%
  True Negative Rate (TNR): 22.2%  <-- TOO LOW!
  Accuracy: 84.0%
```

Notice the first attempt had a TNR of only 22.2% — meaning when a recipe actually violated dietary restrictions, the judge only caught it 22% of the time! This is dangerous (imagine telling a diabetic a recipe is safe when it isn't). After careful prompt iteration, the judge achieved 100% TNR.

### Target Metrics

**Good judge:**
- TPR > 80%
- TNR > 80%

**Great judge:**
- TPR > 90%
- TNR > 90%

**Both must be high!** A judge with TPR=95% but TNR=40% is useless because you'll miss most real failures.

### Iterating on Your Judge Prompt

**Your first prompt won't be perfect. That's expected.**

**Process:**

1. **Test your judge** on Dev set
2. **Calculate TPR and TNR**
3. **Look at errors:**
   - Where did it miss real failures? (False Negatives)
   - Where did it false alarm? (False Positives)
4. **Update the prompt:**
   - Add missed scenarios to criteria
   - Add false alarm scenarios to "NOT a failure" section
   - Add 1-2 more examples of correct judgments
5. **Test again on Dev set**
6. **Repeat until both metrics > 80%**
7. **THEN test once on Test set for final, unbiased metrics**

### Step 6: Final Evaluation on Test Set

Once your judge performs well on Dev, run it on the Test set ONCE:

```python
# Calculate final metrics from test set results
tp = sum(1 for r in results if r["judge"] == "PASS" and r["truth"] == "PASS")
tn = sum(1 for r in results if r["judge"] == "FAIL" and r["truth"] == "FAIL")
fp = sum(1 for r in results if r["judge"] == "PASS" and r["truth"] == "FAIL")
fn = sum(1 for r in results if r["judge"] == "FAIL" and r["truth"] == "PASS")

tpr = tp / (tp + fn)
tnr = tn / (tn + fp)

print(f"Final TPR: {tpr:.1%}")
print(f"Final TNR: {tnr:.1%}")
```

### Step 7: Run on All Traces at Scale

Once validated, run your judge on ALL production traces:

**With LangWatch (batch evaluation with built-in concurrency):**

```python
import langwatch

# Run judge on all production traces
results = langwatch.evaluate.batch(
    dataset=all_traces_df,
    evaluators=[judge_evaluator],
    concurrency=20,  # Parallel processing
    cache=True  # Cache results for duplicate traces
)

# Get summary statistics
pass_rate = results.metrics["pass_rate"]
print(f"Raw pass rate: {pass_rate:.1%}")
```

**LangWatch advantage:** Automatic concurrency management, built-in caching, progress tracking.

**With Langfuse (experiment on dataset):**

```python
result = langfuse.run_experiment(
    name="full-evaluation",
    data=all_traces_data,
    task=judge_task,
    evaluators=[accuracy_evaluator],
    max_concurrency=20,
)
```

**With plain OpenAI (platform-agnostic):**

```python
import openai
from concurrent.futures import ThreadPoolExecutor

client = openai.OpenAI()

def run_judge(trace):
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": judge_prompt.format(**trace)}],
        temperature=0,
    )
    return parse_json(response.choices[0].message.content)

with ThreadPoolExecutor(max_workers=20) as executor:
    results = list(executor.map(run_judge, all_traces))
```

**Example result:** Raw pass rate on 1000 traces = 84.4%

But this raw rate doesn't account for judge errors. Chapter 10 covers how to correct for this using the `judgy` library.

### LLM-as-Judge Across Different Domains

The recipe bot is one example. Here's how the same methodology applies to other domains:

**Customer Support Bot:**
```
Criterion: "Did the agent follow the refund policy correctly?"
PASS: Agent offered refund within 30-day window per policy
FAIL: Agent denied valid refund or offered refund outside policy
```

**Code Generation Assistant:**
```
Criterion: "Does the generated code actually solve the user's problem?"
PASS: Code compiles, handles edge cases, follows the user's constraints
FAIL: Code has syntax errors, misses requirements, or uses deprecated APIs
```

**Medical Information Bot:**
```
Criterion: "Does the response include appropriate disclaimers?"
PASS: Includes "consult your doctor" and avoids specific diagnoses
FAIL: Provides diagnosis-like statements without medical disclaimers
```

**E-commerce Search:**
```
Criterion: "Are the recommended products relevant to the query?"
PASS: Products match stated preferences (size, color, price range)
FAIL: Products violate stated filters or preferences
```

The structure is always the same: define the criterion, write PASS/FAIL definitions, add few-shot examples, validate with TPR/TNR.

---


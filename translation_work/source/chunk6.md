<a name="chapter-10"></a>
## Chapter 10: Statistical Correction with judgy

### The Problem: Your Judge Isn't Perfect

Even a good judge makes mistakes. If your judge has:
- TPR = 95.7% (misses 4.3% of real passes)
- TNR = 100% (never misses a real fail)

Then the raw pass rate from your judge is slightly biased.

### What is judgy?

[judgy](https://github.com/ai-evals-course/judgy) is a Python library that corrects for judge errors using statistical methods. It takes:

1. **Test labels** (ground truth from your labeled data)
2. **Test predictions** (what your judge said about the labeled data)
3. **Unlabeled predictions** (what your judge said about all production traces)

And returns a corrected success rate with confidence intervals.

### How to Use judgy

```python
import numpy as np
from judgy import estimate_success_rate

# From your test set evaluation (Step 6 from Chapter 4)
test_labels = np.array([0, 1, 1, 1, 1, 1, 1, 1, 0, 1, ...])  # Ground truth
test_preds = np.array([0, 1, 1, 1, 1, 1, 1, 1, 0, 1, ...])   # Judge predictions

# From running judge on all production traces (Step 7)
unlabeled_preds = np.array([1, 1, 0, 1, 1, 1, 0, 1, ...])  # Judge on all data

# Compute corrected rate
results = estimate_success_rate(
    test_labels=test_labels,
    test_preds=test_preds,
    unlabeled_preds=unlabeled_preds
)
```

### Real Results: Before and After Correction

```
Final Evaluation on 1000 traces:
  Raw observed success rate:  84.4%
  Corrected success rate:     88.2%  (+3.8 percentage points)
  95% Confidence Interval:    [84.4%, 98.5%]

Interpretation:
  The Recipe Bot adheres to dietary preferences approximately
  88.2% of the time. We are 95% confident the true rate is
  between 84.4% and 98.5%.
```

**Why the correction matters:** The raw rate (84.4%) underestimates the true performance because the judge has a slight false-negative tendency (TPR=95.7%, not 100%). The corrected rate (88.2%) accounts for this bias.

### Platform Integration

**Platform-agnostic:** `judgy` works with results from any platform. Export your test set results and production predictions, then run the correction:

```python
# With LangWatch results
test_results = langwatch.get_experiment_results(experiment_id="test-eval")
test_labels = test_results["ground_truth"]
test_preds = test_results["judge_predictions"]

production_results = langwatch.get_evaluation_results(eval_id="production-run")
unlabeled_preds = production_results["predictions"]

# Run judgy correction
corrected = estimate_success_rate(test_labels, test_preds, unlabeled_preds)
```

```python
# With Langfuse results (manual export)
test_labels = [score.value for score in test_scores if score.name == "ground_truth"]
test_preds = [score.value for score in test_scores if score.name == "judge"]
unlabeled_preds = [score.value for score in production_scores if score.name == "judge"]

# Run judgy correction
corrected = estimate_success_rate(test_labels, test_preds, unlabeled_preds)
```

### For PMs: How to Report These Results

When presenting to stakeholders:

```
"Our Recipe Bot correctly follows dietary restrictions 88% of the time,
with 95% confidence that the true rate is between 84% and 99%.

This means approximately 12% of recipes may contain ingredients that
violate the user's stated dietary preferences. For high-risk diets
(diabetic-friendly, nut-free), we recommend additional safeguards."
```

This is much more credible than "we tested it and it seems to work."

---

<a name="chapter-11"></a>
## Chapter 11: Closing the Loop — From Evals to Improvements

### The Most Common Failure: Measuring Without Acting

Many teams build great eval suites, then never systematically use the results to improve their system. Evals are only valuable if they drive action.

### The Improvement Cycle

```
1. Run evals → identify top failure mode
2. Root-cause the failure (is it prompt? retrieval? tool? data?)
3. Implement a fix (change prompt, add guardrail, fix tool)
4. Run evals again → confirm improvement, check for regressions
5. Repeat with the next failure mode
```

### Root-Causing Failures

When your eval identifies a failure, ask **where** in the pipeline it happened:

| Failure Location | Symptoms | Typical Fixes |
|---|---|---|
| **System prompt** | Wrong tone, missing capabilities, policy violations | Edit prompt, add examples, add constraints |
| **Retrieval** | Wrong documents, missing context | Better chunking, reranking, query expansion |
| **Tool calls** | Wrong tool selected, wrong parameters | Improve tool descriptions, add validation |
| **Generation** | Hallucination, wrong format, ignores context | Few-shot examples, structured output, temperature tuning |
| **Post-processing** | Truncation, encoding issues, format errors | Fix parsing code, add validation |

### Regression Testing

Every time you fix something, you risk breaking something else. Set up regression tests:

```python
class RegressionSuite:
    def __init__(self):
        self.known_cases = []  # Cases that previously failed and were fixed

    def add_regression_case(self, input, expected_output, failure_description):
        self.known_cases.append({
            "input": input,
            "expected": expected_output,
            "original_failure": failure_description,
        })

    def run(self, pipeline):
        regressions = []
        for case in self.known_cases:
            output = pipeline(case["input"])
            if not passes_eval(output, case["expected"]):
                regressions.append({
                    "input": case["input"],
                    "original_failure": case["original_failure"],
                    "current_output": output,
                })
        return regressions

# Usage: run before every prompt change or model switch
suite = RegressionSuite()
suite.add_regression_case(
    input="Give me a vegan recipe with honey",
    expected_output="Should explain honey isn't vegan and suggest alternatives",
    failure_description="Bot used to include honey in vegan recipes"
)
```

**Platform Support for Regression Testing:**

**LangWatch:** Built-in regression test suites, automatic comparison with baseline runs.

**Langfuse:** Manual tracking via datasets, requires custom logic for regression detection.

### Model Comparison with Evals

When evaluating whether to switch models (e.g., GPT-4o vs. Claude vs. Gemini):

```python
MODELS = ["gpt-4o", "claude-sonnet-4-5-20250929", "gemini-2.0-flash"]

for model in MODELS:
    results = run_eval_suite(model=model, test_set=test_data)
    print(f"{model}: TPR={results['tpr']:.1%}, TNR={results['tnr']:.1%}, "
          f"cost=${results['cost']:.2f}, latency={results['latency_p50']:.0f}ms")
```

### For PMs: The Improvement Playbook

After every eval cycle, create a simple report:

```
EVAL REPORT — Week of [date]

Top 3 failure modes this week:
1. [Failure mode] — [X]% of traces — [Root cause] — [Action item]
2. [Failure mode] — [X]% of traces — [Root cause] — [Action item]
3. [Failure mode] — [X]% of traces — [Root cause] — [Action item]

Improvements from last week:
- [Previous fix]: Failure rate went from X% to Y% ✅

Regressions detected: [None / List]
```

---

<a name="chapter-12"></a>
## Chapter 12: Human Annotation Best Practices

### When Manual Labels Beat LLM Labels

- **Ambiguous cases** where even experts disagree — you need to capture that disagreement
- **High-stakes domains** (medical, legal, financial) where errors have real consequences
- **New failure modes** that your LLM judge hasn't been trained to detect
- **Ground truth calibration** — even if you use LLM labeling at scale, validate a sample manually

### Inter-Annotator Agreement

If two humans disagree on a label, your eval criteria aren't clear enough.

**Process:**
1. Have 2-3 people independently label the same 50 traces
2. Calculate agreement rate (% they agree)
3. If agreement < 80%, your criteria need to be more specific
4. Discuss disagreements, update criteria, re-label

```python
def cohen_kappa(labels_a, labels_b):
    """Calculate inter-annotator agreement"""
    agree = sum(a == b for a, b in zip(labels_a, labels_b))
    p_observed = agree / len(labels_a)

    # Expected agreement by chance
    p_a_pos = sum(a == "PASS" for a in labels_a) / len(labels_a)
    p_b_pos = sum(b == "PASS" for b in labels_b) / len(labels_b)
    p_expected = p_a_pos * p_b_pos + (1 - p_a_pos) * (1 - p_b_pos)

    kappa = (p_observed - p_expected) / (1 - p_expected)
    return kappa

# Interpretation:
# kappa > 0.8: Excellent agreement (criteria are clear)
# kappa 0.6-0.8: Good agreement (minor clarifications needed)
# kappa < 0.6: Poor agreement (rewrite criteria)
```

### Label Quality > Label Quantity

**50 high-quality labels beat 500 noisy labels.** Invest time in:
1. Clear, written labeling guidelines with examples
2. Edge case documentation ("if you see X, label it as Y because...")
3. Regular calibration sessions where labelers discuss disagreements

### For PMs/QAs: You Are the Best Labelers

PMs and QAs often produce better labels than engineers because:
- You know what a good user experience looks like
- You understand the product's policies and constraints
- You think from the user's perspective, not the code's perspective

---

<a name="chapter-13"></a>
## Chapter 13: Cost, Latency & Scaling Evals

### The Cost Problem

Running GPT-4o as a judge on 10,000 traces is expensive. Here's how to manage costs:

### Strategy 1: Use Cheaper Models for Judges

Not every eval needs the best model:

| Judge Model | Cost (per 1K traces) | When to Use |
|---|---|---|
| GPT-4o / Claude Opus | ~$5-15 | Complex subjective judgments, safety-critical |
| GPT-4o-mini / Claude Haiku | ~$0.50-1.50 | Clear-cut criteria, well-defined rubrics |
| Code-based | $0 | Format checks, pattern matching, validation |

**Tip:** Start with a strong model, validate your judge prompt, then test if a cheaper model gives similar TPR/TNR. Often it does.

### Strategy 2: Sample Instead of Exhaustive

You don't need to eval every trace:

```python
import random

def sample_traces(traces, sample_rate=0.1, min_sample=100):
    """Sample a fraction of traces for evaluation"""
    sample_size = max(int(len(traces) * sample_rate), min_sample)
    return random.sample(traces, min(sample_size, len(traces)))

# 10% sample of 50,000 daily traces = 5,000 evals
# Statistical confidence is still high with proper sampling
```

### Strategy 3: Tiered Evaluation

Run cheap evals on everything, expensive evals on a sample:

```python
# Tier 1: Run on ALL traces (code-based, free)
tier1_results = [eval_format(t) for t in all_traces]

# Tier 2: Run on traces that passed Tier 1 (cheap LLM, ~$0.50/1K)
tier1_passed = [t for t, r in zip(all_traces, tier1_results) if r['passed']]
tier2_results = run_llm_eval(tier1_passed, model="gpt-4o-mini")

# Tier 3: Run on a sample (expensive LLM, ~$5/1K)
sample = random.sample(tier1_passed, 500)
tier3_results = run_llm_eval(sample, model="gpt-4o")
```

### Strategy 4: Cache Duplicate Evaluations

If the same input appears multiple times, cache the eval result:

```python
import hashlib

eval_cache = {}

def cached_eval(trace, eval_fn):
    key = hashlib.md5(str(trace['input'] + trace['output']).encode()).hexdigest()
    if key not in eval_cache:
        eval_cache[key] = eval_fn(trace)
    return eval_cache[key]
```

**Platform Support for Caching:**

**LangWatch:** Built-in caching for evaluations, automatically deduplicates identical traces.

**Langfuse:** Manual caching required, but supports custom cache keys via metadata.

### Latency Considerations for Real-Time Guardrails

| Check Type | Typical Latency | Suitable for Real-Time? |
|---|---|---|
| Regex/code checks | <1ms | Yes |
| Embedding similarity | 10-50ms | Yes |
| Small LLM (Haiku-class) | 200-500ms | Marginal (adds noticeable delay) |
| Large LLM (GPT-4o-class) | 1-3s | No (use offline only) |

---

<a name="chapter-14"></a>
## Chapter 14: Practical Implementation Guide

### Your First Two Weeks with Evals

### Week 1: Foundation

#### Day 1-2: Set Up Logging (4 hours)

**Goal:** Capture traces of every AI interaction.

Pick your platform and set it up:

**LangWatch:**
```bash
pip install langwatch
# Sign up at langwatch.ai or run self-hosted Docker
```

```python
import langwatch
langwatch.init()  # That's it! Auto-instrumentation enabled
```

**Langfuse:**
```bash
pip install langfuse openai
# Sign up at cloud.langfuse.com or self-host
```

```python
from langfuse.openai import OpenAI  # Drop-in replacement
client = OpenAI()  # Auto-traced
```

Then instrument your app (see Chapter 2 for full examples).

**Deliverable:** Every AI interaction is logged and visible in your observability UI.

#### Day 3: Manual Error Analysis (3 hours)

**Goal:** Review 100 traces and take notes.

1. Open your trace viewer (LangWatch or Langfuse UI)
2. Browse through traces
3. Note problems in a spreadsheet or CSV
4. Budget 30-60 seconds per trace

**Deliverable:** 40-50 error notes from 100 traces.

#### Day 4: Categorize Errors (2 hours)

**Goal:** Group your notes into 5-6 categories.

1. Export your notes
2. Use an LLM to suggest categories
3. Refine categories to be specific and actionable
4. Label each note with a category
5. Count occurrences

**Deliverable:** Prioritized list of what's broken, with frequency data.

#### Day 5-7: Build Your First Eval (6 hours)

**Goal:** Create one code-based eval and one LLM judge.

**Code-based eval (Day 5):** Pick your highest-frequency objective issue.

**LLM judge (Day 6-7):**
1. Write the judge prompt with criteria + examples
2. Label 50-100 traces as ground truth
3. Split into train/dev/test
4. Validate on dev set (iterate prompt until TPR/TNR > 80%)
5. Test on test set for final metrics

**Deliverable:** Two working evals you can run on new traces.

### Week 2: Automation and Monitoring

#### Day 8-9: Automate Eval Runs

**With LangWatch:**
```python
import langwatch

# All evaluators (code + LLM) in one place
results = langwatch.evaluate.batch(
    dataset=daily_traces,
    evaluators=[
        "no_markdown_sms",      # Code-based (custom)
        "dietary_compliance",   # LLM-based (built-in)
    ]
)

print(f"Pass rate: {results.metrics['pass_rate']:.1%}")
```

**With Langfuse:**
```python
# Run evaluators separately
for trace in daily_traces:
    # Code-based
    markdown_result = eval_no_markdown(trace)
    langfuse.create_score(trace_id=trace.id, name="markdown", ...)
    
    # LLM-based
    dietary_result = run_dietary_judge(trace)
    langfuse.create_score(trace_id=trace.id, name="dietary", ...)
```

#### Day 10-11: Set Up Alerts

```python
def check_for_degradation(current_rate, historical_avg, threshold=1.5):
    """Alert if failure rate spikes"""
    return current_rate > historical_avg * threshold

# Example alert
if check_for_degradation(today_failure_rate, avg_failure_rate):
    send_slack_alert("Eval failure rate spiked!")
```

**LangWatch:** Built-in alerting via email, Slack, or webhook when metrics cross thresholds.

**Langfuse:** Custom alerting requires integration with your monitoring system.

#### Day 12-14: Dashboard

**LangWatch:** Built-in analytics dashboard, no setup needed.

**Langfuse:** Create custom dashboard using their API:
```python
# Fetch recent scores
scores = langfuse.api.score.list(limit=1000, from_timestamp=last_week)

# Aggregate and visualize
failure_rates = aggregate_by_day(scores)
plot_dashboard(failure_rates)
```

### Ongoing: 30 Minutes Per Week

**Every Monday (15 minutes):**
1. Check your observability UI for anomalies
2. Review any alerts from past week
3. Note patterns

**Every Month (2 hours):**
1. Do error analysis on 50 new traces
2. Look for new failure modes
3. Add new evals if needed
4. Retire evals that never fire

**After Major Changes (1 hour):**
1. Run full eval suite
2. Compare to baseline
3. Investigate any regressions

---

<a name="chapter-15"></a>
## Chapter 15: Common Mistakes to Avoid

### Mistake #1: Skipping Error Analysis

**What people do:** Jump straight to building LLM judges or dashboards.
**Why it's wrong:** You don't know what to measure yet.
**Fix:** Always start with error analysis. Spend real time reviewing traces.

### Mistake #2: Using Only Agreement for Validation

**What people do:** "My judge has 90% agreement with humans, ship it!"
**Why it's wrong:** A judge that always says "pass" gets 90% agreement when failures are rare.
**Fix:** Always calculate TPR and TNR separately. Both must be high.

### Mistake #3: PM/QA Delegates Error Analysis

**What people do:** "This is technical, let engineering review the logs."
**Why it's wrong:** Engineers don't have product intuition or domain expertise.
**Fix:** PMs and QAs must do error analysis. This is core product/quality work.

### Mistake #4: Not Splitting Data (Train/Dev/Test)

**What people do:** Use all labeled data to build and test the judge.
**Why it's wrong:** You're overfitting to your test data. Your metrics are meaningless.
**Fix:** Use the 15%/40%/45% split. Never touch the test set until final evaluation.

### Mistake #5: Not Doing Evals Until After Launch

**What people do:** Build the product, ship it, then start thinking about evals.
**Fix:** Build evals while building the product, not after.

### Mistake #6: Building Too Many Evals

**What people do:** "Let's have an eval for everything!"
**Fix:** Start with 2-3 evals for your biggest problems. Add more only when needed.
**Rule:** If an eval hasn't fired in 3 months, remove it.

### Mistake #7: Low TNR (Ignoring False Positives)

**What people do:** "My eval catches all real problems (TPR=95%), good enough."
**Why it's wrong:** If it also false-alarms constantly (TNR=22% like a naive first attempt), you'll ignore it.
**Fix:** Both TPR AND TNR must be high. Low TNR means the eval is useless.

### Mistake #8: Not Testing the Evals Themselves

**What people do:** Write an eval, assume it works, run it on all data.
**Fix:** Test your evals with known good and bad cases before deploying.

### Mistake #9: Copy-Pasting Eval Prompts

**What people do:** "This LLM judge prompt worked for someone else, I'll use it."
**Fix:** Write evals specific to YOUR product, YOUR policies, YOUR users.

### Mistake #10: Not Versioning System Prompts

**What people do:** Edit system prompt directly in production.
**Fix:** Use your platform's prompt management (LangWatch, Langfuse, etc.) to version prompts. Log which version was used with each trace.

### Mistake #11: Not Correcting for Judge Bias

**What people do:** Report the raw pass rate from the judge as the true rate.
**Fix:** Use judgy to correct for judge errors and report confidence intervals.

### Mistake #12: Over-Engineering Early

**What people do:** Build a distributed eval platform before reviewing a single trace.
**Fix:** Start simple. CSV + Python script + any observability tool. Add complexity only when simple stops working.

---

<a name="chapter-16"></a>
## Chapter 16: Tools and Resources

### Observability Platforms

| Tool | Type | Best For | Cost |
|------|------|----------|------|
| **LangWatch** | Open source, cloud or self-hosted | Simple setup, built-in evaluators, great analytics | Free tier + paid |
| **Langfuse** | Open source, cloud or self-hosted | Custom pipelines, maximum flexibility, large community | Free tier + paid |
| **Braintrust** | Cloud | Excellent UI, team collaboration | Paid |
| **LangSmith** | Cloud | LangChain users | Paid |
| **Build Your Own** | Custom | Learning, custom needs | Free |

### Eval Frameworks

- **LangWatch Evaluators** - 40+ built-in evaluators covering safety, quality, RAG, and custom domains
- **Langfuse Evals** - Built-in LLM-as-Judge, custom evaluators via SDK
- **Simple Evals** (OpenAI) - Lightweight model-graded evals
- **Ragas** - Specialized for RAG evaluation
- **DeepEval** - Comprehensive eval framework

### Key Libraries

- **judgy** - Statistical bias correction for LLM judges: [github.com/ai-evals-course/judgy](https://github.com/ai-evals-course/judgy)
- **rank_bm25** - BM25 retrieval for RAG systems
- **litellm** - Unified LLM API interface

### Platform Comparison Matrix

| Feature | LangWatch | Langfuse | Notes |
|---------|-----------|----------|-------|
| **Setup Time** | 5 min (3 lines) | 15 min (more config) | LangWatch: langwatch.init() |
| **Built-in Evaluators** | 40+ | 0 (all custom) | LangWatch saves significant dev time |
| **Custom Evaluators** | Yes (decorator) | Yes (full SDK) | Both support custom logic |
| **Analytics Dashboard** | Built-in, automatic | Build your own | LangWatch: zero-config analytics |
| **Cost Tracking** | Automatic | Manual tagging | LangWatch tracks per-model costs |
| **Community Size** | Growing | Large, established | Langfuse has more integrations |
| **Self-Hosting** | Docker (simple) | Docker (more complex) | Both are fully open-source |
| **Prompt Management** | Yes | Yes (more mature) | Langfuse has richer versioning UI |
| **Caching** | Built-in | Manual | LangWatch auto-caches duplicate evals |
| **Batch Evaluation** | Native API | Via experiments | LangWatch: simpler for large batches |
| **Real-time Evals** | Supported | Via scores API | Both work, LangWatch is faster to set up |

**When to choose LangWatch:**
- You want to start fast (< 10 min setup)
- You need built-in evaluators for common use cases
- You want automatic analytics without configuration
- You prefer opinionated tooling that "just works"

**When to choose Langfuse:**
- You need maximum flexibility for custom workflows
- You have complex evaluation logic
- You want the largest community and integration ecosystem
- You prefer building your own dashboards and analytics

**Why not both?**
Many teams use both: LangWatch for quick evals and analytics, Langfuse for deep customization. They're complementary, not competitive.

### Key Principles (Revisited)

1. **Start simple** - Don't over-engineer
2. **Error analysis first** - Always
3. **PMs and QAs must be involved** - This is product/quality work
4. **Both TPR and TNR matter** - Not just agreement
5. **Code evals when possible** - LLM judges when needed
6. **Test your evals** - They can have bugs too
7. **Split your data** - Train/Dev/Test is non-negotiable
8. **Correct for bias** - Use judgy for honest metrics
9. **Version your prompts** - Track what changed when
10. **Iterate based on data** - Not hunches

---


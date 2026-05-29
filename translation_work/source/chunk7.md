<a name="appendix-a"></a>
## Appendix A: Glossary for PMs & QAs

A plain-language glossary of the technical terms used throughout this guide. Share this with non-technical stakeholders.

### Evaluation & Metrics Terms

| Term | Definition |
|------|-----------|
| **Eval (Evaluation)** | A systematic test that checks if an AI system is working correctly for a specific criterion |
| **LLM-as-a-Judge** | Using a language model to automatically evaluate the output of another AI system |
| **Ground Truth** | Human-verified labels that represent the "correct" answer; used to measure judge accuracy |
| **True Positive Rate (TPR)** | The percentage of actual positives (e.g., good responses) that the judge correctly identifies. Also called *recall* or *sensitivity*. Formula: TP / (TP + FN) |
| **True Negative Rate (TNR)** | The percentage of actual negatives (e.g., bad responses) that the judge correctly catches. Also called *specificity*. Formula: TN / (TN + FP) |
| **False Positive (FP)** | When the judge says "Pass" but the real answer is "Fail" — a missed defect |
| **False Negative (FN)** | When the judge says "Fail" but the real answer is "Pass" — a false alarm |
| **Precision** | Of all items the judge labeled positive, how many were actually positive. Formula: TP / (TP + FP) |
| **F1 Score** | The harmonic mean of precision and recall — a single number balancing both. Formula: 2 * (Precision * Recall) / (Precision + Recall) |
| **Confusion Matrix** | A 2x2 table showing TP, FP, FN, TN counts — the foundation of all classification metrics |
| **Confidence Interval (CI)** | A range of values (e.g., 72%–81%) within which the true metric likely falls, given sampling uncertainty |
| **Bias Correction** | Adjusting raw judge scores to account for systematic over- or under-counting of passes/fails |
| **Cohen's Kappa** | A statistic measuring agreement between two raters (or a rater and ground truth), adjusting for chance agreement. Values: <0.2 poor, 0.4–0.6 moderate, 0.6–0.8 substantial, >0.8 almost perfect |

### Data & Workflow Terms

| Term | Definition |
|------|-----------|
| **Train/Dev/Test Split** | Dividing labeled data into three sets: Train (for building the judge prompt), Dev (for iterating), Test (for final unbiased measurement) |
| **Stratified Split** | Splitting data so each subset has the same proportion of Pass/Fail labels as the original |
| **Few-Shot Examples** | Example input-output pairs included in a prompt to show the model what good evaluation looks like |
| **Open Coding** | Reading traces and writing freeform notes about what's going wrong — no categories yet |
| **Axial Coding** | Grouping your open-coded notes into categories (error types) and counting frequency |
| **Dimensional Sampling** | Systematically creating test inputs that cover all important dimensions (topics, edge cases, user types) |
| **Failure Mode** | A specific, named way the AI system can fail (e.g., "dietary violation," "hallucinated citation") |
| **Error Taxonomy** | The organized list of all failure modes for your application, ranked by frequency and severity |

### Observability & Platform Terms

| Term | Definition |
|------|-----------|
| **Trace** | A complete record of one AI interaction — from user input through all processing steps to final output |
| **Span** | A single unit of work within a trace (e.g., one LLM call, one database lookup, one tool invocation) |
| **Instrumentation** | Adding code to your application so that traces and spans are automatically captured |
| **Dataset** | A stored collection of examples (inputs + expected outputs) used for running experiments |
| **Experiment** | Running your AI system (or judge) against a dataset and recording all results |
| **Annotation** | A label or score attached to a trace or span — can be human-generated or from an automated eval |
| **Prompt Version** | A saved snapshot of a prompt template, allowing you to track changes and compare performance |

### RAG-Specific Terms

| Term | Definition |
|------|-----------|
| **RAG (Retrieval-Augmented Generation)** | An AI architecture that retrieves relevant documents before generating a response |
| **BM25** | A classic keyword-based search algorithm used as a baseline for retrieval quality |
| **Recall@K** | Of all relevant documents, what fraction appear in the top K retrieved results |
| **MRR (Mean Reciprocal Rank)** | Average of 1/rank for the first relevant document — higher means relevant docs appear sooner |
| **Chunking** | Splitting large documents into smaller pieces for retrieval |
| **Context Window** | The maximum amount of text an LLM can process in a single call |
| **Hallucination** | When an LLM generates information not supported by the retrieved context |

### Statistical Terms

| Term | Definition |
|------|-----------|
| **p_obs (Observed Rate)** | The raw pass rate from the judge, before any correction |
| **θ̂ (Theta-hat)** | The corrected true success rate after accounting for judge errors |
| **judgy** | A Python library that computes corrected success rates and confidence intervals given TPR and TNR |
| **Sampling** | Evaluating a random subset of traces instead of all traces — used to manage cost |
| **Statistical Significance** | Whether an observed difference is likely real or could be due to random chance |

---

<a name="appendix-b"></a>
## Appendix B: Quick Reference

### When to Use What Type of Eval

| Situation | Type | Example |
|-----------|------|---------|
| Format checking | Code-based | No markdown in SMS |
| Required fields | Code-based | Tour confirmation has date/time |
| Tool selection | Code-based | Called correct function |
| Subjective quality | LLM judge | Response is helpful |
| Policy compliance | LLM judge | Handoff requirements met |
| Dietary adherence | LLM judge | Recipe follows restrictions |
| Factual accuracy | LLM judge | Answer matches sources |
| Response length | Code-based | Under 500 characters |

### Metrics Cheat Sheet

```
Confusion Matrix:
                 Actual Positive  |  Actual Negative
                 -----------------|-----------------
Predicted Pos    |      TP        |       FP        |
Predicted Neg    |      FN        |       TN        |

TPR (Recall) = TP / (TP + FN)      "Catches real positives"
TNR (Specificity) = TN / (TN + FP) "Avoids false alarms"
Precision = TP / (TP + FP)
F1 Score = 2 * (Precision * Recall) / (Precision + Recall)

Target for evals:
- TPR > 80% (catches real issues)
- TNR > 80% (doesn't false alarm)
```

### Data Split Ratios

```
Train: ~15%  (few-shot examples for judge prompt)
Dev:   ~40%  (iterate and improve judge prompt)
Test:  ~45%  (final, unbiased evaluation - use ONCE)
```

### Time Estimates

| Activity | Time | Frequency |
|----------|------|-----------|
| Initial setup (LangWatch) | 30 min | Once |
| Initial setup (Langfuse) | 1 hour | Once |
| Error analysis (100 traces) | 1 hour | Monthly |
| Build code-based eval | 1 hour | As needed |
| Build LLM judge (full pipeline) | 4-6 hours | As needed |
| Validate eval on dev set | 1 hour | Per iteration |
| Weekly maintenance | 30 min | Weekly |

### Platform Quick Start

**LangWatch (fastest):**
```python
import langwatch
langwatch.init()
# Done! Auto-tracing enabled
```

**Langfuse (more config):**
```python
from langfuse.openai import OpenAI
client = OpenAI()
# Set environment variables first
```

---

<a name="appendix-c"></a>
## Appendix C: Complete Judge Prompts from Production

This is a production-quality judge prompt that achieved TPR=95.7% and TNR=100%:

```
You are an expert nutritionist and dietary specialist evaluating whether
recipe responses properly adhere to specified dietary restrictions.

DIETARY RESTRICTION DEFINITIONS:
- Vegan: No animal products (meat, dairy, eggs, honey, etc.)
- Vegetarian: No meat or fish, but dairy and eggs are allowed
- Gluten-free: No wheat, barley, rye, or other gluten-containing grains
- Dairy-free: No milk, cheese, butter, yogurt, or other dairy products
- Keto: Very low carb (typically <20g net carbs), high fat, moderate protein
- Paleo: No grains, legumes, dairy, refined sugar, or processed foods
- Pescatarian: No meat except fish and seafood
- Kosher: Follows Jewish dietary laws (no pork, shellfish, mixing meat/dairy)
- Halal: Follows Islamic dietary laws (no pork, alcohol, proper slaughter)
- Nut-free: No tree nuts or peanuts
- Low-carb: Significantly reduced carbohydrates (typically <50g per day)
- Sugar-free: No added sugars or high-sugar ingredients
- Raw vegan: Vegan foods not heated above 118 degrees F (48 degrees C)
- Whole30: No grains, dairy, legumes, sugar, alcohol, or processed foods
- Diabetic-friendly: Low glycemic index, controlled carbohydrates
- Low-sodium: Reduced sodium content for heart health

EVALUATION CRITERIA:
- PASS: The recipe clearly adheres to the dietary preferences with
  appropriate ingredients and preparation methods
- FAIL: The recipe contains ingredients or methods that violate the
  dietary preferences
- Consider both explicit ingredients and cooking methods

Example 1:
Query and Response: [Gluten-free pizza dough using gluten-free flour blend,
baking powder, olive oil, honey, apple cider vinegar...]
Explanation: The recipe uses gluten-free flour blend. All other ingredients
do not contain gluten. The preparation method does not introduce any
gluten-containing elements.
Label: PASS

Example 2:
Query and Response: [Raw vegan quinoa salad with cooked quinoa,
fresh vegetables, olive oil, lemon juice...]
Explanation: The recipe FAILS because it includes cooked quinoa.
Raw vegan diets do not allow foods heated above 118 degrees F (48 degrees C),
and cooking quinoa involves boiling, which exceeds this limit.
Label: FAIL

Now evaluate the following recipe response:

Query: {query}
Dietary Restriction: {dietary_restriction}
Recipe Response: {response}

RETURN YOUR EVALUATION IN JSON FORMAT:
"label": "PASS" or "FAIL",
"explanation": "Detailed explanation citing specific ingredients or methods"
```

---

<a name="appendix-d"></a>
## Appendix D: Pipeline State Evaluator Prompts

Complete evaluator prompts for each pipeline state. Each follows the same structure:

### Standard Evaluator Structure

```
1. Role definition ("You are an expert evaluator for the X state")
2. What the state should do (3-4 bullet points)
3. Evaluation criteria (3-4 numbered criteria)
4. What counts as a failure (4-5 specific failure types)
5. What does NOT count as a failure (2-3 acceptable variations)
6. Input/Output template variables
7. Output format (JSON with label and explanation)
```

### Available Evaluators

| State | Key Criteria | Common Failures |
|-------|-------------|----------------|
| ParseRequest | Accuracy, completeness, format | Misinterpretation, missing constraints |
| PlanToolCalls | Tool selection, ordering, rationale | Missing tools, incorrect tools |
| GenRecipeArgs | Query relevance, filter accuracy | Missing dietary filters, wrong servings |
| GetRecipes | Relevance, dietary compliance | Irrelevant recipes, dietary violations |
| GenWebArgs | Relevance, context alignment | Off-topic queries, too generic |
| GetWebInfo | Relevance, quality | Irrelevant results, off-topic content |
| ComposeResponse | Recipe accuracy, step clarity, constraint compliance | Contradictions, missing info, violations |

Full evaluator prompts for each state follow the structure above, tailored to the specific responsibilities and failure modes of each pipeline stage.

---

<a name="appendix-e"></a>
## Appendix E: Judge Prompt Engineering Tips

A collection of techniques that consistently improve LLM judge accuracy. Use this as a checklist when building or debugging a judge.

### 1. Explanation Before Verdict

Always ask the judge to explain its reasoning *before* giving the final label. This is the single most impactful technique.

```
❌ Bad:  "Label: PASS or FAIL. Explanation: ..."
✅ Good: "Explanation: [your reasoning]. Label: PASS or FAIL"
```

**Why it works:** When the model writes the label first, the explanation becomes a post-hoc rationalization. When reasoning comes first, the model actually deliberates, and the label follows logically.

### 2. Be Ruthlessly Specific About Criteria

Vague criteria lead to inconsistent judgments. Define exactly what counts as Pass and Fail.

```
❌ Vague:  "Does the response follow dietary restrictions?"
✅ Specific: "PASS: Every ingredient in the recipe is compatible with the stated
   dietary restriction. FAIL: At least one ingredient violates the restriction,
   OR the cooking method introduces a violation (e.g., frying in butter for
   dairy-free)."
```

### 3. Include "What Does NOT Count as a Failure"

Judges tend to be overly strict. Explicitly list acceptable variations to calibrate leniency.

```
What does NOT count as a failure:
- Suggesting optional toppings that can be omitted
- Using brand names instead of generic ingredient names
- Minor formatting issues in the recipe
- Providing substitution suggestions alongside the main recipe
```

### 4. Use Domain-Specific Few-Shot Examples

Generic examples are far less effective than examples from your actual data. Always pull few-shot examples from your Train set.

**Example selection strategy:**
- 1 clear Pass (easy case)
- 1 clear Fail (easy case)
- 1 borderline case (the kind the judge will struggle with most)

**Include the reasoning** in each example, not just the label. The judge learns the reasoning pattern, not just the answer.

### 5. Temperature Settings

| Use Case | Temperature | Rationale |
|----------|-------------|-----------|
| Binary classification (Pass/Fail) | 0.0 | Deterministic, reproducible |
| Likert scale scoring (1-5) | 0.0–0.3 | Low variance, consistent |
| Generating diverse critiques | 0.5–0.7 | Some creativity for different angles |
| Brainstorming failure modes | 0.7–1.0 | High creativity for exploration |

For judge evaluation, always use temperature 0.0. You want the same input to produce the same output every time.

### 6. Structured Output Formats

Tell the judge exactly how to format its response. JSON is preferred for parsing reliability.

```
Return your evaluation as JSON:
{
  "explanation": "Step-by-step reasoning about the response...",
  "label": "PASS or FAIL",
  "confidence": "HIGH, MEDIUM, or LOW",
  "flagged_items": ["list of specific problematic items, if any"]
}
```

**Tip:** The `confidence` field is useful for identifying borderline cases during error analysis but is not a reliable calibrated probability.

### 7. Guard Against Common Judge Biases

| Bias | Description | Mitigation |
|------|-------------|------------|
| **Leniency bias** | Judge defaults to "Pass" too often | Add explicit failure examples; emphasize "when in doubt, FAIL" |
| **Verbosity bias** | Judge favors longer, more detailed responses | Add examples where a short response passes and a long one fails |
| **Position bias** | Judge favors the first/last option in a list | Randomize order if comparing multiple outputs |
| **Sycophancy bias** | Judge agrees with confident-sounding text | Add examples where confident text is wrong |
| **Anchoring bias** | Judge is swayed by the first piece of evidence | Instruct judge to consider ALL evidence before concluding |

### 8. Iterative Refinement Workflow

```
1. Write initial prompt with 2-3 few-shot examples
2. Run on Dev set → calculate TPR and TNR
3. Find the worst errors (cases where judge was wrong)
4. For each error:
   a. Understand WHY the judge was wrong
   b. Add a clarification, edge case, or new example to the prompt
5. Re-run on Dev set → check if metrics improved
6. Repeat steps 3-5 until TPR > 80% and TNR > 80%
7. Run ONCE on Test set for final, unbiased metrics
```

**Common iteration patterns:**
- TPR too low → Judge is missing real failures. Add more Fail examples, make fail criteria more explicit.
- TNR too low → Judge has too many false alarms. Add "what does NOT count as a failure" section, add Pass examples for edge cases.
- Both low → Criteria are ambiguous. Rewrite from scratch with clearer definitions.

### 9. Model Selection for Judges

| Model Tier | When to Use | Typical Accuracy |
|------------|------------|-----------------|
| GPT-4o / Claude Sonnet 4.6 | High-stakes evals, complex reasoning | 85–95% |
| GPT-4o-mini / Claude Haiku | Cost-sensitive, high-volume evals | 75–90% |
| Open-source (Llama, Mistral) | Self-hosted, privacy-sensitive | 70–85% |

**Tip:** Start with the most capable model to establish a performance ceiling. Then test whether a cheaper model can match it for your specific use case. Often it can — especially with good few-shot examples.

### 10. Prompt Versioning

Always version your judge prompts. Track:
- The prompt text
- The few-shot examples used
- The model and temperature
- Dev set metrics (TPR, TNR) at that version
- Date and reason for the change

Both LangWatch and Langfuse have built-in prompt versioning. Use it.

**With LangWatch:**
```python
import langwatch

langwatch.prompts.create(
    name="dietary-judge-v3",
    description="Added edge cases for keto",
    template=judge_prompt_text,
    model="gpt-4o",
    temperature=0
)
```

**With Langfuse:**
```python
from langfuse import get_client

langfuse = get_client()

langfuse.create_prompt(
    name="dietary-judge",
    prompt=judge_prompt_text,
    labels=["staging"],  # promote to "production" after validation
)
```

---


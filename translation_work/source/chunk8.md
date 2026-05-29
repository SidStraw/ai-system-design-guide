<a name="appendix-f"></a>
## Appendix F: Platform Methods Reference (LangWatch & Langfuse)

### LangWatch

#### Tracing

```python
import langwatch

# Initialize (auto-instruments OpenAI, LangChain, LlamaIndex, etc.)
langwatch.init()

# Add custom spans
@langwatch.span(type="chain")
def my_pipeline(question):
    """Parent span for the whole pipeline"""
    sql = generate_sql(question)
    results = execute_query(sql)
    return synthesize_answer(question, results)

@langwatch.span(type="llm")
def generate_sql(question):
    """Tracked as an LLM generation"""
    return client.chat.completions.create(...)

@langwatch.span(type="tool")
def execute_query(sql):
    """Tracked as a tool call"""
    return db.execute(sql)
```

#### Querying Spans

```python
import langwatch

# Get all spans for a specific name
spans_df = langwatch.get_spans(
    filters={"name": "ParseRequest"}
)

# Get spans within a time range
spans_df = langwatch.get_spans(
    filters={
        "timestamp_gte": "2025-02-01",
        "timestamp_lte": "2025-02-09"
    }
)
```

#### Datasets

```python
import pandas as pd
import langwatch

df = pd.DataFrame({
    "query": ["Query 1", "Query 2"],
    "expected_answer": ["Answer 1", "Answer 2"]
})

dataset = langwatch.datasets.create(
    name="my-dataset",
    dataframe=df
)
```

#### Evaluators

```python
import langwatch

# Use built-in evaluators (40+ available)
results = langwatch.evaluate.batch(
    dataset=traces_df,
    evaluators=[
        "dietary_compliance",   # Built-in
        "toxicity",             # Built-in
        "prompt_injection",     # Built-in
    ]
)

# Create custom evaluator
@langwatch.evaluator(name="custom_check")
def my_evaluator(trace):
    # Your logic here
    return {"passed": True, "score": 1.0}

# Run custom evaluator
results = langwatch.evaluate.batch(
    dataset=traces_df,
    evaluators=["custom_check"]
)
```

#### Experiments

```python
import langwatch

def my_task(example):
    query = example["input"]["query"]
    return {"answer": my_pipeline(query)}

# Run experiment with automatic metrics
results = langwatch.evaluate.batch(
    dataset=dataset,
    task=my_task,
    evaluators=["accuracy", "latency", "cost"]
)

# View results
print(results.metrics)
```

#### Prompt Management

```python
import langwatch

# Create prompt
prompt = langwatch.prompts.create(
    name="recipe-assistant-v1",
    template=[
        {"role": "system", "content": "You are a recipe assistant..."},
        {"role": "user", "content": "{{question}}"}
    ],
    model="gpt-4o-mini",
    temperature=0.7
)

# Use at runtime
messages = prompt.render(question="How do I make pancakes?")
response = client.chat.completions.create(
    messages=messages,
    model=prompt.model,
    temperature=prompt.temperature
)
```

### Langfuse

#### Tracing

```python
from langfuse.openai import OpenAI  # Drop-in replacement
from langfuse import observe, get_client

client = OpenAI()  # Calls are automatically traced

@observe()
def my_pipeline(question):
    """Creates a parent trace"""
    return generate_answer(question)

@observe(as_type="generation")
def generate_answer(question):
    """Tracked as a generation"""
    return client.chat.completions.create(...)
```

#### Querying Traces

```python
from langfuse import get_client

langfuse = get_client()

traces = langfuse.api.trace.list(limit=100, tags=["production"])
trace = langfuse.api.trace.get("trace_id")
```

#### Datasets

```python
from langfuse import get_client

langfuse = get_client()

langfuse.create_dataset(name="my-dataset")

langfuse.create_dataset_item(
    dataset_name="my-dataset",
    input={"query": "What is AI?"},
    expected_output={"answer": "Artificial Intelligence"},
)
```

#### Experiments

```python
from langfuse import Evaluation

def my_task(*, item, **kwargs):
    query = item["input"]["query"]
    return my_pipeline(query)

def my_evaluator(*, output, expected_output, **kwargs):
    correct = output == expected_output.get("answer")
    return Evaluation(name="accuracy", value=1.0 if correct else 0.0)

result = langfuse.run_experiment(
    name="baseline",
    data=test_data,
    task=my_task,
    evaluators=[my_evaluator],
)
print(result.format())
```

#### Scores (Evaluation Results)

```python
from langfuse import get_client

langfuse = get_client()

# Score a trace
langfuse.create_score(
    trace_id="trace_id",
    name="dietary_adherence",
    value=1,  # 0 or 1
    data_type="BOOLEAN",
    comment="Recipe correctly follows vegan restrictions",
)

# Score within context
with langfuse.start_as_current_observation(as_type="span", name="eval") as span:
    span.score(name="accuracy", value=0.95, data_type="NUMERIC")
```

#### Prompt Management

```python
from langfuse import get_client

langfuse = get_client()

langfuse.create_prompt(
    name="my-prompt",
    type="chat",
    prompt=[
        {"role": "system", "content": "You are a {{role}}"},
        {"role": "user", "content": "{{question}}"},
    ],
    labels=["production"],
)

prompt = langfuse.get_prompt("my-prompt", type="chat")
compiled = prompt.compile(role="chef", question="Best pasta recipe?")
```

---

<a name="appendix-g"></a>
## Appendix G: 30-Day Learning Path

### Week 1: Foundations (Engineer, PM, or QA)

| Day | Activity | Time | Role Focus |
|-----|----------|------|------------|
| 1 | Pick your platform (LangWatch or Langfuse), install it | 1h | All |
| 2 | Instrument your app with auto-tracing | 2h | Engineer |
| 2 | Browse the trace viewer UI, understand traces visually | 1h | PM/QA |
| 3 | Create a test dataset with dimensional sampling | 2h | All |
| 4 | Upload dataset to your platform, run first experiment | 1h | All |
| 5 | Review 50 traces, take notes (open coding) | 1h | All |
| 6 | Categorize errors using LLM (axial coding) | 1h | All |
| 7 | Prioritize: frequency x severity matrix | 30m | All |

### Week 2: Code-Based Evals

| Day | Activity | Time | Role Focus |
|-----|----------|------|------------|
| 8 | Build 2 code-based evals for your top issues | 2h | Engineer |
| 8 | Define eval criteria in plain English | 1h | PM/QA |
| 9 | Test evals with known good/bad cases | 1h | All |
| 10 | Run evals on all traces, calculate failure rates | 1h | All |
| 11-14 | Iterate based on results | 2h | All |

### Week 3: LLM Judge

| Day | Activity | Time | Role Focus |
|-----|----------|------|------------|
| 15 | Label 100-150 traces as ground truth | 3h | All |
| 16 | Split into Train/Dev/Test | 30m | Engineer |
| 17 | Write first judge prompt with few-shot examples | 2h | All |
| 18 | Validate on Dev set, calculate TPR/TNR | 1h | All |
| 19 | Iterate prompt until metrics > 80% | 2h | All |
| 20 | Final test on Test set | 30m | All |
| 21 | Run judge on all traces + correct with judgy | 1h | All |

### Week 4: Advanced Topics & Production

| Day | Activity | Time | Role Focus |
|-----|----------|------|------------|
| 22 | RAG evaluation — retrieval metrics + answer quality (Ch. 6) | 2h | Engineer |
| 23 | Multi-step pipeline evaluation (Ch. 7) | 2h | Engineer |
| 24 | Multi-turn conversation evaluation (Ch. 8) | 2h | Engineer |
| 25 | Safety evals — prompt injection, PII leakage (Ch. 9) | 2h | All |
| 26 | Set up regression test suite (Ch. 11) | 2h | Engineer |
| 27 | Human annotation calibration — measure inter-annotator agreement (Ch. 12) | 1h | All |
| 28 | Optimize for cost — tiered evaluation, sampling strategy (Ch. 13) | 1h | All |
| 29 | Create monitoring dashboard + automated eval runs | 2h | Engineer |
| 30 | Document eval suite, present to stakeholders, plan maintenance | 2h | All |

---

## Lessons Learned

Real lessons from implementing complete eval pipelines in production:

**On Building Judges (Chapters 4, 10)**

1. **LLM-as-Judge is powerful but needs guardrails** - Without proper validation, a judge can confidently give wrong answers. Always validate against ground truth.

2. **You must test evaluators against ground truth** - A judge that seems reasonable but has TNR=22% is actively harmful — it misses most real failures.

3. **Train/Dev/Test splits enable confidence** - Without them, you're fooling yourself about your judge's quality. This is non-negotiable.

4. **Iterating on judge prompts is crucial** - The first prompt is never good enough. Plan for 3-5 iterations minimum. See Appendix E for techniques.

5. **Explanation-before-verdict is the #1 technique** - Asking the judge to reason before labeling improves accuracy more than any other single change.

**On Process & Methodology (Chapters 3, 11, 12)**

6. **Error analysis is the real work** - Fancy tools don't matter if you haven't sat down and looked at your failures. Open coding → axial coding → prioritization is the workflow that works.

7. **Human annotators disagree more than you think** - Measure inter-annotator agreement (Cohen's kappa) before trusting your ground truth. If humans can't agree, the judge won't either.

8. **Closing the loop is what separates good teams from great ones** - Running evals is only half the job. The other half is systematically turning failures into improvements and preventing regressions.

**On Production & Scale (Chapters 9, 13)**

9. **Safety evals are not optional** - Prompt injection, PII leakage, and jailbreak detection should be running before you worry about quality evals.

10. **Start expensive, then optimize** - Use GPT-4o/Claude Sonnet to establish your performance ceiling, then test whether a cheaper model can match it. Often it can.

11. **Sampling beats exhaustive evaluation** - Evaluating 10% of traces with statistical rigor gives you a better answer than evaluating 100% with a bad judge.

12. **Good observability tools make the workflow 10x faster** - Integrated tracing, evaluation, datasets, and experiments in one platform (LangWatch, Langfuse, etc.) saves enormous time vs. stitching together custom scripts.

**On Platform Choice**

13. **LangWatch for speed, Langfuse for depth** - LangWatch gets you results in hours with built-in evaluators. Langfuse gives maximum control for complex custom logic. Many teams use both.

14. **Built-in evaluators save weeks of dev time** - LangWatch's 40+ built-in evaluators cover most common use cases. If you're reinventing safety checks or RAG metrics, you're wasting time.

15. **Community matters for long-term success** - Langfuse's larger community means more integrations, more examples, more support. LangWatch's simpler API means faster onboarding.

---

## Conclusion

AI evals are not just "testing" — they're a product development methodology that touches engineering, product management, and quality assurance.

**Key takeaways:**

1. **Everyone needs evals** — Not just big companies. If your AI app touches users, you need systematic evaluation.
2. **Start with error analysis** — Sit down and look at your failures before building anything automated (Chapter 3).
3. **PMs and QAs must lead** — Error analysis and criteria definition are product/quality work, not just engineering tasks.
4. **Build incrementally** — Start with code-based evals, then add LLM judges, then add safety evals. Don't try to do everything at once.
5. **Measure what matters** — Application-specific criteria, not generic "helpfulness" scores.
6. **Both TPR and TNR** — A judge that catches failures but also false-alarms is harmful. Measure both.
7. **Split your data** — Train/Dev/Test is mandatory. Without it, you're overfitting your judge.
8. **Correct for bias** — Use statistical correction (Chapter 10) for honest metrics.
9. **Close the loop** — Evals that don't lead to improvements are wasted effort (Chapter 11).
10. **Plan for scale** — Start with the best model, then optimize for cost (Chapter 13).

**Your action plan (see Appendix G for details):**

1. Week 1: Set up observability (LangWatch or Langfuse), do error analysis
2. Week 2: Build 2-3 core code-based evals
3. Week 3: Build and validate an LLM judge with proper train/dev/test splits
4. Week 4: Advanced topics — RAG evals, multi-turn evals, safety evals, automation
5. Ongoing: 30 minutes per week maintenance + regression testing

**Platform decision:**
- Choose **LangWatch** if you want to start fast (<30 min setup) and use built-in evaluators
- Choose **Langfuse** if you need maximum flexibility and have complex custom workflows
- Use **both** if you want the best of both worlds (many teams do this)

**Remember:** The teams shipping the best AI products are the ones with the best evals. Not the fanciest models. Not the biggest teams. The ones who systematically measure and improve.

Start today. Your future self will thank you.

---

## Learning Resources

### Platform Documentation & Learning Hubs

- **LangWatch Docs**: [docs.langwatch.ai](https://docs.langwatch.ai)
- **LangWatch GitHub**: [github.com/langwatch/langwatch](https://github.com/langwatch/langwatch)
- **Langfuse Docs**: [langfuse.com/docs](https://langfuse.com/docs)
- **Langfuse GitHub**: [github.com/langfuse/langfuse](https://github.com/langfuse/langfuse)
- **Maven Course (AI Evals for Engineers & PMs)**: [maven.com/parlance-labs/evals](https://maven.com/parlance-labs/evals)
- **HuggingFace Evaluation Guidebook**: [github.com/huggingface/evaluation-guidebook](https://github.com/huggingface/evaluation-guidebook)

### Research & Thought Leadership

- **OpenAI Evals Platform**: [evals.openai.com](https://evals.openai.com/)
- **OpenAI Cookbook** (practical examples & guides): [cookbook.openai.com](https://cookbook.openai.com/)
- **OpenAI Research**: [openai.com/research](https://openai.com/research)
- **OpenAI Docs (Evals)**: [platform.openai.com/docs/guides/evals](https://platform.openai.com/docs/guides/evals)
- **Anthropic Research**: [anthropic.com/research](https://www.anthropic.com/research)
- **METR** (Model Evaluation & Threat Research): [metr.org](https://metr.org/)
- **Eugene Yan on eval process**: [eugeneyan.com/writing/eval-process](https://eugeneyan.com/writing/eval-process/)

### Blogs That Shaped This Guide

- **Hamel Husain's Blog**: [hamel.dev](https://hamel.dev/) — Applied AI engineering, LLM evals deep-dives
- **Shreya Shankar's Site**: [sh-reya.com](https://www.sh-reya.com/) — LLM data systems research, eval methodology
- **Maxim AI Articles**: [getmaxim.ai/articles](https://www.getmaxim.ai/articles) — Agentic evaluation patterns

### Open-Source Tools & Libraries

| Tool | Focus | License | Links |
|------|-------|---------|-------|
| **LangWatch** | Observability & built-in evals | Apache 2.0 | [GitHub](https://github.com/langwatch/langwatch) · [Docs](https://docs.langwatch.ai) |
| **Langfuse** | Custom pipelines & tracing | MIT | [GitHub](https://github.com/langfuse/langfuse) · [Docs](https://langfuse.com/docs) |
| **RAGAS** | RAG-specific evaluation | Apache 2.0 | [GitHub](https://github.com/explodinggradients/ragas) · [Docs](https://docs.ragas.io/) |
| **Comet Opik** | LLM tracing & evaluation | Apache 2.0 | [GitHub](https://github.com/comet-ml/opik) · [Site](https://www.comet.com/site/products/opik/) |
| **judgy** | Statistical bias correction | Open | [GitHub](https://github.com/ai-evals-course/judgy) |
| **Braintrust** | Experimentation & logging | Partial | [Docs](https://www.braintrust.dev/docs) |
| **Galileo** | Hallucination detection | Proprietary | [Site](https://www.galileo.ai/) |
| **Maxim** | Agentic system evaluation | Proprietary | [Site](https://www.getmaxim.ai/) |

### Strategy Comparison Matrix

| Company | Focus | Open Source | Best For | Unique Strength |
|---------|-------|-------------|----------|-----------------|
| **LangWatch** | Observability + Built-in Evals | Yes (Apache 2.0) | Fast setup, analytics | 40+ built-in evaluators, auto-analytics |
| **Langfuse** | Custom Pipelines | Yes (MIT) | Data sovereignty, flexibility | Self-hostable, full control over data |
| **Anthropic** | Safety / Red Teaming | Partial | Frontier risks | Constitutional classifiers, multi-attempt adversarial testing |
| **OpenAI** | Preparedness / Business | Evals toolkit | Enterprise context | SME probing, contextual evals |
| **RAGAS** | RAG-specific | Yes (Apache 2.0) | RAG pipelines | Reference-free metrics, synthetic test data generation |
| **Maxim** | Agentic Systems | No | Multi-agent apps | Simulation framework, no-code evaluation |
| **Braintrust** | Experimentation | Partial | Early-stage teams | Collaborative design, fast iteration |
| **Galileo** | Hallucinations | No | Quality assurance | ChainPoll, real-time monitoring |
| **Comet Opik** | LLM Tracing & Evals | Yes (Apache 2.0) | End-to-end observability | Framework integrations, online evaluation rules |
| **METR** | Catastrophic Risk | Research | Policy guidance | Autonomous capability assessment |

### Contact Me
- Om Bharatiya: [@ombharatiya](https://twitter.com/ombharatiya)

### Reference Work Credits
This guide was built on the foundation of the following people's work and ideas. Their courses, blogs, and open-source contributions made this guide possible:
- Hamel Husain: [@HamelHusain](https://x.com/HamelHusain) — [hamel.dev](https://hamel.dev/)
- Shreya Shankar: [@sh_reya](https://x.com/sh_reya) — [sh-reya.com](https://www.sh-reya.com/)
- Eugene Yan: [@eugeneyan](https://x.com/eugeneyan) — [eugeneyan.com](https://eugeneyan.com/)

---

*This guide was inspired by and builds upon the AI Evals for Engineers & PMs course by Hamel Husain and Shreya Shankar, extended with additional research, production-ready code examples, and multi-platform guides covering LangWatch, Langfuse, and the broader eval tooling ecosystem.*

*Author: Om Bharatiya | Created: February 2026*

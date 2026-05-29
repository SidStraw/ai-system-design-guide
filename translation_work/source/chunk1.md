# AI Evals For Engineers, PMs & QAs: Complete Study Guide

*Based on the Maven course by Hamel Husain & Shreya Shankar, enriched with hands-on examples, production-ready code, and platform-specific guides for LangWatch, Langfuse, and more*

**Who is this guide for?**
- **Engineers** building AI-powered products who need to systematically evaluate quality
- **Product Managers** who own the product experience and need to lead error analysis
- **QA Engineers** who need to build automated evaluation pipelines for AI systems
- **Anyone** who wants to learn how to evaluate AI applications without taking the full course

**What you'll learn:**
- How to set up observability for any AI application
- How to systematically find what's broken (error analysis)
- How to build automated evaluators (code-based and LLM judges)
- How to evaluate RAG systems, multi-step pipelines, and multi-turn conversations
- How to run production evals: guardrails, safety, and real-time monitoring
- How to use statistical correction to account for judge errors
- How to close the loop: turn eval results into system improvements
- How to do all of this with your observability platform of choice (LangWatch, Langfuse, Braintrust, LangSmith, or your own)

**Platform Examples:** This guide uses **LangWatch** (open-source, self-hosted or cloud) and **Langfuse** (open-source, cloud or self-hosted) as primary examples. The methodology is platform-agnostic — adapt it to whichever tool you use.

**LangWatch vs Langfuse:** Both are excellent open-source platforms with similar core capabilities. LangWatch offers simpler setup and built-in evaluators, while Langfuse provides more flexibility for custom pipelines and has a larger community. This guide shows both so you can choose based on your needs.

---

## Table of Contents

1. [What Are AI Evals and Why You Need Them](#chapter-1)
2. [Setting Up Observability](#chapter-2)
3. [Error Analysis: The Secret Sauce](#chapter-3)
4. [Building LLM-as-a-Judge Evaluators](#chapter-4)
5. [Code-Based Evaluators](#chapter-5)
6. [RAG System Evaluation](#chapter-6)
7. [Multi-Step Pipeline Evaluation](#chapter-7)
8. [Multi-Turn Conversation Evaluation](#chapter-8)
9. [Production Evals: Safety, Guardrails & Monitoring](#chapter-9)
10. [Statistical Correction with judgy](#chapter-10)
11. [Closing the Loop: From Evals to Improvements](#chapter-11)
12. [Human Annotation Best Practices](#chapter-12)
13. [Cost, Latency & Scaling Evals](#chapter-13)
14. [Practical Implementation Guide](#chapter-14)
15. [Common Mistakes to Avoid](#chapter-15)
16. [Tools and Resources](#chapter-16)

**Appendices:**
- [A: Glossary for PMs & QAs](#appendix-a)
- [B: Quick Reference](#appendix-b)
- [C: Complete Judge Prompts from Production](#appendix-c)
- [D: Pipeline State Evaluator Prompts](#appendix-d)
- [E: Judge Prompt Engineering Tips](#appendix-e)
- [F: Platform Methods Reference (LangWatch & Langfuse)](#appendix-f)
- [G: 30-Day Learning Path](#appendix-g)

---

<a name="chapter-1"></a>
## Chapter 1: What Are AI Evals and Why You Need Them

### Simple Definition

**Evals (Evaluations)** are systematic tests that check if your AI application is working correctly. Think of them like unit tests for traditional software, but for AI systems.

### Why Everyone Needs Evals

There's a debate in the AI community: some people say "just vibe check your app" (meaning: just use it yourself and see if it feels good). But here's the truth:

**Everyone needs evals.** The people who say they don't need evals are actually benefiting from evals that someone else did upstream.

Example: If you're building a coding assistant with GPT-4, OpenAI already tested GPT-4 on massive code benchmarks. So you can "vibe check" your app. But for most applications that aren't simple uses of foundation models, you need your own evals.

### The Three Core Truths About Evals

1. **You can't improve what you don't measure**
   - Generic metrics like "helpfulness score" won't catch specific problems
   - You need application-specific evals

2. **Error analysis is the most important step**
   - More important than LLM judges
   - More important than fancy observability tools
   - This is where you actually learn what's broken

3. **PMs and QAs must own error analysis, not just engineers**
   - Engineers know if code works
   - PMs know if the product experience is good
   - QAs know how to systematically break things
   - You have the domain expertise
   - This is product work, not just technical work

### The AI Development Cycle is the Scientific Method

Building great AI products requires a rigorous evaluation process. In many ways, AI development IS the scientific method:

1. **Observe** - Trace your AI's behavior (Chapter 2)
2. **Hypothesize** - Identify what's broken through error analysis (Chapter 3)
3. **Experiment** - Build evaluators and test changes (Chapters 4-9)
4. **Measure** - Calculate metrics and correct for bias (Chapter 10)
5. **Iterate** - Improve based on data, not hunches (Chapter 11)

### What Can Go Wrong Without Evals?

Your demo works great. Then production happens:

- Users hit edge cases you never thought of
- Text messages contain typos and unusual formatting
- Dates are formatted differently than expected
- AI tries to handle requests it should hand off to humans
- Small prompt changes break things that were working

**Example from real production data:**
```
User: "I need a one bedroom with the bathroom NOT connected"
AI: Returns apartments with connected bathrooms (WRONG!)
User: "I do NOT want a bathroom connected to the room"
AI: "I'll check on that" but never actually checks
PLUS: AI used markdown formatting (* asterisks *) in a text message
```

Three different problems in one interaction! Without proper logging and evaluation, you'd never catch these patterns.

### For PMs: Why This Is Your Job

**Wrong approach:** "This is technical AI stuff, let engineering figure it out"

**Right approach:** PMs should lead error analysis because:
1. You understand user needs
2. You have product taste
3. You have domain expertise
4. This is product work disguised as technical work

**The teams shipping the best AI products have PMs who've personally reviewed hundreds or thousands of traces.**

### For QAs: Your New Superpower

Traditional QA involves test cases with expected outputs. AI QA is different:
1. Outputs are non-deterministic (same input can give different outputs)
2. "Correct" is often subjective
3. Edge cases are virtually infinite
4. You need automated evaluators that scale

But the core QA mindset - systematic testing, edge case thinking, regression prevention - is exactly what AI evals need. QAs who learn evals become incredibly valuable.

---

<a name="chapter-2"></a>
## Chapter 2: Setting Up Observability

### What is a Trace?

A **trace** is a complete recording of everything your AI did to respond to a user. It's like a detailed log that shows:

1. **System prompt** (instructions given to the AI)
2. **User messages** (what the person asked)
3. **Tool calls** (functions the AI tried to use)
4. **Tool responses** (what those functions returned)
5. **Assistant responses** (what the AI said back)
6. **All context** (everything the LLM saw when making decisions)

### Example of a Complete Trace

```
=== TRACE ID: abc123 ===

SYSTEM PROMPT:
"You are a helpful property management assistant..."

USER MESSAGE:
"I need a one bedroom with the bathroom not connected"

TOOL CALL:
get_availability(bedrooms=1, bathroom_connected=None)

TOOL RESPONSE:
[
  {unit: "A101", bedrooms: 1, bathroom_connected: True},
  {unit: "B205", bedrooms: 1, bathroom_connected: True}
]

ASSISTANT RESPONSE:
"I found these apartments: A101 and B205..."
(Used markdown: ** ** in text message)
```

### What Information to Capture

**Minimum requirements:**
- Input (user message)
- Output (AI response)
- Timestamp
- Unique ID for the interaction

**Better to include:**
- System prompts used
- Tool calls and their results
- Model parameters (temperature, max_tokens, etc.)
- Token counts
- Latency (response time)
- Cost per request

**Best practice:**
- User context (session history)
- Error messages if any occurred
- Model version used
- Feature flags active at the time

### Choosing an Observability Platform

| Tool | Type | Best For | Cost |
|------|------|----------|------|
| **LangWatch** | Open source, cloud or self-hosted | Simple setup, built-in evaluators, great UX | Free tier + paid |
| **Langfuse** | Open source, cloud or self-hosted | Custom pipelines, large community | Free tier + paid |
| **Braintrust** | Cloud | Excellent UI, team collaboration | Paid |
| **LangSmith** | Cloud | LangChain users | Paid |
| **Build Your Own** | Custom | Learning, custom needs | Free |

**LangWatch vs Langfuse comparison:**
- **Setup:** LangWatch is simpler (3-line integration), Langfuse requires more configuration
- **Evaluators:** LangWatch has 40+ built-in evaluators, Langfuse requires custom implementation
- **Flexibility:** Langfuse is more flexible for custom workflows, LangWatch is more opinionated
- **Community:** Langfuse has a larger community and more integrations
- **UI:** Both have excellent UIs; LangWatch focuses on analytics, Langfuse on workflow

All of these support the same core concepts: traces, spans, datasets, evaluations, and experiments. The methodology in this guide works with any of them.

### Setting Up LangWatch (Open-Source, Cloud or Self-Hosted)

LangWatch is an open-source LLM observability and analytics platform. It provides tracing, evaluation, datasets, experiments, and 40+ built-in evaluators.

#### Install and Configure

```bash
pip install langwatch
```

```python
# Set your API key (get one at langwatch.ai or self-host)
import os
os.environ["LANGWATCH_API_KEY"] = "lw_..."  # or set in .env file
```

**Cloud vs Self-Hosted:**
- **Cloud:** Sign up at [langwatch.ai](https://langwatch.ai), get API key, done in 5 minutes
- **Self-Hosted:** Run `docker-compose up` with their Docker setup, point to your own instance

#### Instrument Your Application (Auto-Tracing)

LangWatch supports auto-instrumentation for most frameworks:

```python
import langwatch

# Initialize LangWatch
langwatch.init()

# Your existing OpenAI code now gets traced automatically!
import openai
client = openai.OpenAI()

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "You are a recipe assistant."},
        {"role": "user", "content": "How do I make pancakes?"}
    ],
    temperature=0.7
)
# This call is automatically captured by LangWatch!
```

**Framework Support:**
- OpenAI (automatic)
- LangChain (automatic)
- LlamaIndex (automatic)
- Anthropic Claude (automatic)
- Any custom LLM (manual spans)

#### Add Custom Spans with Decorators

```python
import langwatch

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

**Comparison with Langfuse:**
Both use decorators, but LangWatch's `@langwatch.span()` is simpler than Langfuse's `@observe()`. LangWatch automatically categorizes spans by type, while Langfuse requires explicit `as_type` parameters.

### Setting Up Langfuse (Open-Source, Cloud or Self-Hosted)

Langfuse provides tracing, evaluation, datasets, experiments, and prompt management. It offers a managed cloud and a self-hosted option.

#### Install and Configure

```bash
pip install langfuse openai
```

```python
# Set environment variables (or pass to constructor)
# LANGFUSE_SECRET_KEY="sk-lf-..."
# LANGFUSE_PUBLIC_KEY="pk-lf-..."
# LANGFUSE_HOST="https://cloud.langfuse.com"  # or your self-hosted URL
```

#### Instrument Your Application (Drop-In Replacement)

```python
# Just change your import — everything else stays the same!
from langfuse.openai import OpenAI

client = OpenAI()

# This call is automatically traced by Langfuse
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "You are a recipe assistant."},
        {"role": "user", "content": "How do I make pancakes?"}
    ],
    temperature=0.7
)
```

#### Add Custom Spans with Decorators

```python
from langfuse import observe

@observe()
def my_pipeline(question):
    """Parent trace for the whole pipeline"""
    sql = generate_sql(question)
    results = execute_query(sql)
    return synthesize_answer(question, results)

@observe(as_type="generation")
def generate_sql(question):
    """Tracked as a generation (LLM call)"""
    return client.chat.completions.create(...)
```

### Creating and Managing Prompts

Both platforms support versioned prompt management:

#### LangWatch

```python
import langwatch

# Create a prompt template
langwatch.prompts.create(
    name="recipe-assistant-v1",
    template=[
        {"role": "system", "content": "You are a recipe assistant..."},
        {"role": "user", "content": "{{question}}"}
    ],
    model="gpt-4o-mini",
    temperature=0.7
)

# Use at runtime
prompt = langwatch.prompts.get("recipe-assistant-v1")
messages = prompt.render(question="How do I make pancakes?")
response = client.chat.completions.create(messages=messages, **prompt.settings)
```

**LangWatch advantage:** Simpler API, automatic parameter management (temperature, model stored with prompt).

#### Langfuse

```python
from langfuse import get_client

langfuse = get_client()

langfuse.create_prompt(
    name="recipe-assistant",
    type="chat",
    prompt=[
        {"role": "system", "content": "You are a recipe assistant..."},
        {"role": "user", "content": "{{query}}"},
    ],
    labels=["production"],
)

# Use at runtime
prompt = langfuse.get_prompt("recipe-assistant", type="chat")
compiled = prompt.compile(query="How do I make pancakes?")
```

**Langfuse advantage:** More mature prompt management, better versioning UI, labels for organization.

### Uploading Test Datasets

#### LangWatch

```python
import langwatch
import pandas as pd

df = pd.DataFrame({
    "query": [
        "Suggest a quick vegan breakfast recipe",
        "I have chicken and rice. What can I cook?",
        "Give me a dessert recipe with chocolate",
    ]
})

dataset = langwatch.datasets.create(
    name="recipe-queries",
    dataframe=df,
)
```

**LangWatch advantage:** Direct pandas DataFrame support, simpler API.

#### Langfuse

```python
from langfuse import get_client

langfuse = get_client()

langfuse.create_dataset(name="recipe-queries")

for query in ["Suggest a quick vegan breakfast recipe",
              "I have chicken and rice. What can I cook?",
              "Give me a dessert recipe with chocolate"]:
    langfuse.create_dataset_item(
        dataset_name="recipe-queries",
        input={"query": query},
    )
```

**Langfuse advantage:** More control over individual items, better for incremental additions.

### Key Principle

**Without traces, you can't do evals.** This is your foundation. Set this up first before anything else.

**For PMs/QAs:** You don't need to write the instrumentation code. Ask your engineers to set up tracing, then use the web UI to review traces visually. Both LangWatch (`langwatch.ai` or your self-hosted URL) and Langfuse (`cloud.langfuse.com` or your self-hosted URL) provide UIs that let you browse, search, and annotate traces without writing any code.

**Platform Choice Guidance:**
- Choose **LangWatch** if: You want the fastest setup, built-in evaluators, and focus on analytics
- Choose **Langfuse** if: You need maximum flexibility, have complex custom workflows, or want the largest community
- Use **both**: They complement each other - LangWatch for quick evals, Langfuse for deep workflow customization

---


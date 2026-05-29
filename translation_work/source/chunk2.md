<a name="chapter-3"></a>
## Chapter 3: Error Analysis: The Secret Sauce

### What is Error Analysis?

Error analysis is the **systematic process** of:
1. Reviewing traces (logs of AI interactions)
2. Taking notes on problems you see
3. Categorizing those problems
4. Counting how often each type of problem occurs

**This is THE most important skill** in building reliable AI products.

Most teams skip straight to building fancy dashboards or LLM judges. That's backwards. You need to understand what's wrong before you can measure it.

### Why PMs and QAs MUST Do This (Not Just Engineers)

**Wrong approach:**
"This is technical AI stuff, let engineering figure it out"

**Right approach:**
PMs and QAs should lead error analysis because:

1. **You understand user needs** - Engineers don't know if a "connected bathroom" vs "disconnected bathroom" matters to users
2. **You have product taste** - You know what good experiences look like
3. **You have domain expertise** - You understand business requirements
4. **This is product work** - Disguised as technical work, but it's really about product quality

**Real impact:**
The teams shipping the best AI products have PMs who've personally reviewed hundreds or thousands of traces.

### Step 1: Generate Diverse Test Queries

Before you can review traces, you need diverse test inputs. A powerful technique for this is **dimensional sampling**.

#### Define Key Dimensions

Identify 3-4 dimensions that matter for your product:

```python
DIMENSIONS = {
    "dietary_restriction": [
        "vegan", "vegetarian", "gluten-free", "keto", "no restrictions"
    ],
    "cuisine_type": [
        "Italian", "Asian", "Mexican", "Mediterranean", "American"
    ],
    "meal_type": [
        "breakfast", "lunch", "dinner", "snack", "dessert"
    ],
    "skill_level": [
        "beginner", "intermediate", "advanced"
    ],
}

# Total possible combinations: 5 x 5 x 5 x 3 = 375
```

#### Generate Random Combinations

```python
import random

random.seed(42)
dimension_tuples = []

for i in range(25):  # Generate 25 diverse tuples
    tuple_data = {
        "dietary_restriction": random.choice(DIMENSIONS["dietary_restriction"]),
        "cuisine_type": random.choice(DIMENSIONS["cuisine_type"]),
        "meal_type": random.choice(DIMENSIONS["meal_type"]),
        "skill_level": random.choice(DIMENSIONS["skill_level"]),
    }
    dimension_tuples.append(tuple_data)
```

#### Convert Tuples to Natural Language Queries Using an LLM

You can use any LLM to convert dimension tuples into realistic queries. Here are platform-specific approaches:

**With LangWatch (built-in generation):**

```python
import langwatch

QUERY_GEN_PROMPT = """Convert this dimension tuple into a realistic user query
for a Recipe Bot. Be creative and vary your style.

Dimension tuple: {tuple_description}

Generate 1 unique, realistic query:"""

queries = []
for t in dimension_tuples:
    result = langwatch.completion(
        prompt=QUERY_GEN_PROMPT.format(tuple_description=str(t)),
        model="gpt-4o-mini",
        temperature=0.9
    )
    queries.append(result.text)
```

**With any LLM (platform-agnostic):**

```python
import openai

client = openai.OpenAI()

QUERY_GEN_PROMPT = """Convert this dimension tuple into a realistic user query
for a Recipe Bot. Be creative and vary your style.

Dimension tuple: {tuple_description}

Generate 1 unique, realistic query:"""

queries = []
for t in dimension_tuples:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": QUERY_GEN_PROMPT.format(
            tuple_description=str(t)
        )}],
        temperature=0.9
    )
    queries.append(response.choices[0].message.content)
```

**With Langfuse (manual tracking):**

```python
from langfuse.openai import OpenAI

client = OpenAI()  # Auto-traced

queries = []
for t in dimension_tuples:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": QUERY_GEN_PROMPT.format(
            tuple_description=str(t)
        )}],
        temperature=0.9
    )
    queries.append(response.choices[0].message.content)
```

**Example conversions:**

| Dimension Tuple | Generated Query |
|---|---|
| vegan, Italian, dinner, beginner | "Hey, I'm new to cooking and vegan. Can you suggest an easy Italian dinner?" |
| gluten-free, any, dessert, intermediate | "I'm looking for a gluten-free dessert that's a bit of a challenge to make" |
| keto, American, breakfast, advanced | "Give me a complex keto breakfast recipe, American style" |

**For PMs/QAs:** This dimensional approach ensures you test the full space of user needs. Without it, you'll only test the obvious cases and miss edge cases where users combine unexpected requirements.

### Step 2: Review 100 Traces and Take Notes (Open Coding)

**The process (30 seconds per trace):**

1. Open your trace viewer (LangWatch dashboard, Langfuse UI, or any tool)
2. Look at the first trace
3. Scan through it:
   - Read the user message
   - Check if AI called the right tools
   - Look at what the tools returned
   - Read the assistant's response
   - Note any problems you see

**Example notes from a real error analysis session:**

```
TRACE #1:
"Told user it would check on bathrooms but didn't do it.
Did not follow user instructions.
Rendered markdown in a text message."

TRACE #2:
"Returned properties outside user's price range.
Tool call had correct parameters but didn't filter results."

TRACE #3:
"Good response. No issues."

TRACE #4:
"Failed to hand off to human when user asked for same-day tour.
Policy violation."
```

**Rules for error analysis:**

1. **Don't try to catch everything** - Just note the most important things
2. **Don't debate every trace** - Think quickly, write it down, move on
3. **Skip the system prompt** - If it's usually the same, you don't need to read it every time
4. **Get into a flow state** - This should feel fast, not tedious

**Time commitment:**
- First trace: 45 seconds
- After 10 traces: 25 seconds each
- After 50 traces: 20 seconds each
- **Total time for 100 traces: ~45 minutes**

**Platform-Specific Note:**
- **LangWatch:** Use the "Annotations" feature to add notes directly to traces in the UI
- **Langfuse:** Use the "Comments" feature to add notes to traces

### Step 3: Categorize Errors Using Axial Coding

Now you have 40-50 notes scattered across traces. Time to organize them.

This process is called **"axial coding"** (a research method from sociology). You're grouping similar errors into categories.

#### Using an LLM to Help Discover Categories

Export your notes, then use this prompt:

```python
prompt = f"""
You are analyzing Recipe Bot failures. Look at these examples where
a user queried the bot, the bot responded, and an analyst described
what went wrong.

EXAMPLES:
{combined_df.to_json(orient="records", lines=True)}

Based on the patterns you see in the analyst's descriptions,
create 4-6 systematic failure mode labels.

Each label should:
- Be short and clear (2 words max)
- Capture a distinct type of failure pattern
- Be applicable to multiple traces

Respond with a list:
["label1", "label2", "label3", "label4", "label5", "label6"]
"""
```

**Example results from a real recipe bot evaluation:**

```
["Dietary Ignored", "Formatting Error", "Complexity Mismatch",
 "Meal Type Mismatch", "Ingredient Omission", "Skill Level Misalignment"]
```

#### Refine Categories to Be Specific and Actionable

**Problem:** Generic LLM suggestions are too vague!

"Temporal issues" - what does that mean?
"Quality issues" - too generic!

**Better categories (specific and actionable):**

1. **Dietary Ignored** - Bot suggests ingredients that violate dietary restrictions
2. **Formatting Error** - Markdown in SMS, wrong structure
3. **Complexity Mismatch** - Recipe too hard/easy for stated skill level
4. **Meal Type Mismatch** - Suggests dinner when asked for breakfast
5. **Ingredient Omission** - Doesn't include unique ingredients user asked for
6. **Skill Level Misalignment** - Advanced techniques for beginners

**Your categories need to be specific enough that someone else could label errors using them.**

### Step 4: Label Your Errors with LLM Assistance

This step works with any LLM. Use batch processing if your platform supports it:

```python
CLASSIFICATION_PROMPT = """Look at this Recipe Bot interaction and the
analyst's description. Apply the most appropriate failure mode label.

USER QUERY: {input_query}
BOT RESPONSE: {bot_response}
ANALYST'S ISSUE DESCRIPTION: {issue_description}

AVAILABLE LABELS:
{failure_mode_labels}

Respond with just the label name."""

# Run classification on each error note (use your platform's batch API
# or loop with any LLM client)
```

**With LangWatch (batch evaluation):**

```python
import langwatch

results = langwatch.evaluate.batch(
    dataset=error_notes_df,
    evaluator="custom_classifier",
    prompt_template=CLASSIFICATION_PROMPT,
    model="gpt-4o-mini"
)
```

**With Langfuse (manual iteration):**

```python
from langfuse.openai import OpenAI

client = OpenAI()

for note in error_notes:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": CLASSIFICATION_PROMPT.format(**note)}],
        temperature=0
    )
    note["label"] = response.choices[0].message.content
```

### Step 5: Count and Prioritize

**Count how many times each category appears:**

```python
label_counts = results["output"].value_counts()
```

**Example results from a real evaluation:**

| Category | Count | Percentage |
|----------|-------|------------|
| Complexity Mismatch | 2 | 22% |
| Meal Type Mismatch | 2 | 22% |
| Ingredient Omission | 2 | 22% |
| Dietary Ignored | 1 | 11% |
| Formatting Error | 1 | 11% |
| Skill Level Misalignment | 1 | 11% |

### Why This Changes Everything

**Before error analysis:**
- You're paralyzed
- Don't know what to fix first
- Can't prioritize

**After error analysis:**
- Clear priorities based on frequency
- Understanding of severity (frequency vs. impact)
- Evidence for stakeholder discussions
- Concrete list of what to build evals for

**Example prioritization discussion:**

```
"Dietary restriction violations happen in 11% of cases, but when
they occur, we could harm users with allergies. This is HIGH-SEVERITY.

Formatting issues happen in 11% of cases, but they're just
annoying, not dangerous. This is LOW-SEVERITY.

Let's fix dietary adherence first, then complexity matching."
```

### The "Theoretical Saturation" Concept

**When to stop reviewing traces?**

In qualitative research, there's a concept called "theoretical saturation" - when you stop finding new types of errors.

- Review your first 50 traces: You find 10 different error types
- Review next 25 traces: You find 2 new error types
- Review next 25 traces: You find 0 new error types
- **Stop here!** You've reached saturation

You don't need to review 1000 traces if you're not finding new patterns after 100.

### For PMs/QAs: Your Error Analysis Checklist

1. Ask engineering to set up tracing (LangWatch, Langfuse, or any tool)
2. Open the trace viewer UI
3. Browse 100 traces, taking quick notes on problems
4. Use an LLM to help categorize your notes into 4-6 failure modes
5. Count occurrences of each failure mode
6. Create a prioritized list considering both frequency and severity
7. Present findings to your team with data-backed recommendations
8. Repeat monthly with new traces to catch new failure patterns

---


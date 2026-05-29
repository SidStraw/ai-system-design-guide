<a name="chapter-4"></a>
## 第四章：建構 LLM 作為評測器的評估系統

### 什麼是 LLM 評測器？

**LLM 評測器**是一種用來評估其他 AI 輸出結果的 AI。它讀取追蹤記錄並為其評分。

**為何使用它？**
- 自動化大規模評估
- 提供一致的判斷
- 比人工審查快得多

**挑戰所在：**
大多數人建構評測器的方式是錯的。他們的評測器會產生幻覺、遺漏問題，或製造虛假的信心。

### 何時使用 LLM 評測器

**適合使用 LLM 評測器的情境：**
- 主觀品質評估
- 政策合規性檢查
- 上下文理解
- 飲食規範遵循
- 語調適當性
- 多步驟推理檢查

**不適合使用 LLM 評測器的情境：**
- 格式驗證（改用程式碼）
- 必填欄位檢查（改用程式碼）
- 簡單模式匹配（改用程式碼）
- 精確字串匹配（改用程式碼）

**基本原則：** 如果可以用 if/else 陳述式來表達，就用程式碼。如果需要判斷力，就用 LLM。

### LLM 評測器完整工作流程

建構可靠的 LLM 評測器需要嚴謹的 7 步驟工作流程：

#### 概述：流程管線

```
1. Generate traces (run your AI on test queries)
2. Label a subset manually (or with a powerful LLM)
3. Split into Train / Dev / Test sets
4. Develop your judge prompt using Train examples
5. Validate on Dev set (iterate until good)
6. Final evaluation on Test set (unbiased metrics)
7. Run on all traces + correct with judgy
```

### 步驟一：生成追蹤記錄

在多樣化的測試查詢上執行你的 AI 系統以建立追蹤記錄。使用你的平台自動化埋點（參見第二章）來自動捕獲所有資訊。

### 步驟二：標記基準真相資料

將 150-200 筆追蹤記錄標記為 PASS 或 FAIL。你可以手動進行（最精確），也可以使用強大的 LLM：

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

**各平台標記方式：**

**使用 LangWatch（內建評估器）：**

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

**使用 Langfuse（自訂實作）：**

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

**LangWatch 優勢：** 40 個以上的內建評估器，節省常見使用情境的時間。
**Langfuse 優勢：** 對自訂評估邏輯擁有完全的控制權。

### 步驟三：分割資料（訓練集／開發集／測試集）

這一步至關重要，卻經常被跳過！你需要三個獨立的資料集：

- **訓練集（約 15%）：** 用於為評測器提示詞選取少樣本示例
- **開發集（約 40%）：** 用於迭代和改進評測器提示詞
- **測試集（約 45%）：** 僅使用**一次**，用於最終的無偏差評估

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

**為何需要分層抽樣？** 你需要在每個分割中同時包含 PASS 和 FAIL 的範例。若不進行分層，你可能會得到一個全為 PASS 範例的開發集，使其無法用於測試失敗情境的偵測。

### 步驟四：建構評測器提示詞

你的評測器提示詞需要**四個關鍵部分：**

#### 第一部分：角色與領域定義

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

#### 第二部分：明確的評估標準

```
EVALUATION CRITERIA:
- PASS: The recipe clearly adheres to the dietary preferences
  with appropriate ingredients and preparation methods
- FAIL: The recipe contains ingredients or methods that violate
  the dietary preferences
- Consider both explicit ingredients AND cooking methods
```

#### 第三部分：少樣本示例（來自你的訓練集！）

這就是訓練集發揮作用的地方。從中選取 1-3 個正確判斷的示例：

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

#### 第四部分：輸出格式

```
Now evaluate the following:
Query: {query}
Dietary Restriction: {dietary_restriction}
Recipe Response: {response}

RETURN YOUR EVALUATION IN JSON FORMAT:
"label": "PASS" or "FAIL",
"explanation": "Detailed explanation citing specific ingredients or methods"
```

### 為何二元評分效果最好

**有些人想要 1-5 分制或百分比評分。請不要這樣做。**

**使用二元評分（PASS/FAIL）：**
- 只需驗證兩件事
- 決策邊界清晰
- 更易於除錯
- 更容易向利害關係人解釋

**使用 1-5 分制：**
- 需要驗證每個分數的對齊情況
- 2 分和 3 分之間有什麼差別？
- 需要 5 倍的工作量來驗證
- 業務決策本來就是二元的

**請記住：** 要麼你解決了問題，要麼沒有。要麼它壞了，要麼沒壞。

### 步驟五：在開發集上驗證

在開發集上執行你的評測器，並與基準真相進行比較。以下是各平台的操作方式：

#### 評估器函數（平台無關）

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

#### 執行實驗

**使用 LangWatch：**

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

**LangWatch 優勢：** 內建指標計算，無需手動計算混淆矩陣。

**使用 Langfuse：**

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

**Langfuse 優勢：** 對評估邏輯有更多控制，更適合複雜的自訂指標。

### 真正重要的指標

**大多數人只看「一致性」：**

```
Agreement = (Judge agrees with me) / (Total traces)
Example: 90% agreement
```

**為何這具有誤導性：**

如果失敗情況只佔 10% 的時間，一個總是說「通過」的評測器，在完全沒有用的情況下，仍可獲得 90% 的準確率！

**你真正需要的兩個指標：**

#### 1. TPR（真陽性率）- 召回率

**「當實際上是 PASS 時，評測器正確說 PASS 的頻率有多高？」**

```
TPR = True Positives / (True Positives + False Negatives)
```

#### 2. TNR（真陰性率）- 特異性

**「當實際上是 FAIL 時，評測器正確說 FAIL 的頻率有多高？」**

```
TNR = True Negatives / (True Negatives + False Positives)
```

### 真實結果：為何迭代很重要

**經過仔細的提示詞迭代（生產品質的評測器）：**

```
Test Set Performance:
  True Positive Rate (TPR): 95.7%
  True Negative Rate (TNR): 100.0%
  Balanced Accuracy: 97.8%
  Total predictions: 33
  Correct predictions: 32
  Overall Accuracy: 97.0%
```

**首次嘗試（迭代前）：**

```
Test Set Performance:
  True Positive Rate (TPR): 90.1%
  True Negative Rate (TNR): 22.2%  <-- TOO LOW!
  Accuracy: 84.0%
```

請注意，首次嘗試的 TNR 僅為 22.2%——這意味著當食譜實際上違反飲食限制時，評測器只有 22% 的時間能夠發現。這非常危險（試想一下，告訴糖尿病患者某道食譜是安全的，但其實不是）。經過仔細的提示詞迭代後，評測器達到了 100% 的 TNR。

### 目標指標

**良好的評測器：**
- TPR > 80%
- TNR > 80%

**優秀的評測器：**
- TPR > 90%
- TNR > 90%

**兩者都必須很高！** TPR=95% 但 TNR=40% 的評測器毫無用處，因為你會錯過大多數真實的失敗案例。

### 迭代你的評測器提示詞

**你的第一個提示詞不會完美。這是正常的。**

**流程：**

1. **測試你的評測器**，在開發集上執行
2. **計算 TPR 和 TNR**
3. **查看錯誤：**
   - 它在哪裡遺漏了真實的失敗？（假陰性）
   - 它在哪裡誤報了？（假陽性）
4. **更新提示詞：**
   - 將遺漏的場景添加到評估標準中
   - 將誤報場景添加到「非失敗」部分
   - 添加 1-2 個正確判斷的示例
5. **再次在開發集上測試**
6. **重複，直到兩個指標都超過 80%**
7. **然後在測試集上測試一次，以獲得最終的無偏差指標**

### 步驟六：在測試集上進行最終評估

一旦評測器在開發集上表現良好，就在測試集上執行**一次**：

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

### 步驟七：大規模在所有追蹤記錄上執行

一旦驗證通過，就在**所有**生產追蹤記錄上執行你的評測器：

**使用 LangWatch（具有內建並發功能的批次評估）：**

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

**LangWatch 優勢：** 自動並發管理、內建快取、進度追蹤。

**使用 Langfuse（在資料集上實驗）：**

```python
result = langfuse.run_experiment(
    name="full-evaluation",
    data=all_traces_data,
    task=judge_task,
    evaluators=[accuracy_evaluator],
    max_concurrency=20,
)
```

**使用原生 OpenAI（平台無關）：**

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

**範例結果：** 1000 筆追蹤記錄的原始通過率 = 84.4%

但此原始率未考慮評測器本身的錯誤。第 10 章將介紹如何使用 `judgy` 函式庫來進行校正。

### LLM 評測器在不同領域的應用

食譜機器人只是一個例子。以下展示相同的方法論如何應用於其他領域：

**客服機器人：**
```
Criterion: "Did the agent follow the refund policy correctly?"
PASS: Agent offered refund within 30-day window per policy
FAIL: Agent denied valid refund or offered refund outside policy
```

**程式碼生成助手：**
```
Criterion: "Does the generated code actually solve the user's problem?"
PASS: Code compiles, handles edge cases, follows the user's constraints
FAIL: Code has syntax errors, misses requirements, or uses deprecated APIs
```

**醫療資訊機器人：**
```
Criterion: "Does the response include appropriate disclaimers?"
PASS: Includes "consult your doctor" and avoids specific diagnoses
FAIL: Provides diagnosis-like statements without medical disclaimers
```

**電子商務搜尋：**
```
Criterion: "Are the recommended products relevant to the query?"
PASS: Products match stated preferences (size, color, price range)
FAIL: Products violate stated filters or preferences
```

結構始終相同：定義標準、撰寫 PASS/FAIL 定義、添加少樣本示例、使用 TPR/TNR 進行驗證。

---

<a name="chapter-3"></a>
## 第三章：錯誤分析：致勝關鍵

### 什麼是錯誤分析？

錯誤分析是一種**系統化的流程**，包含：
1. 審閱 trace（AI 互動的日誌）
2. 記錄你發現的問題
3. 將問題分類
4. 統計各類問題出現的頻率

**這是**打造可靠 AI 產品**最重要的技能**。

大多數團隊會直接跳去建立花俏的儀表板或 LLM 評審機制。這樣的順序是錯的。你必須先了解問題所在，才能加以衡量。

### 為什麼 PM 和 QA 必須親自做（而不只是交給工程師）

**錯誤做法：**
「這是 AI 技術的事，讓工程師去搞定就好」

**正確做法：**
PM 和 QA 應該主導錯誤分析，原因如下：

1. **你了解使用者需求** — 工程師不知道「有附屬衛浴」和「無附屬衛浴」對使用者來說有沒有差異
2. **你有產品品味** — 你知道好的體驗應該是什麼樣子
3. **你有領域專業** — 你了解業務需求
4. **這是產品工作** — 表面上像技術工作，但本質上是產品品質的問題

**實際影響：**
打造出最優秀 AI 產品的團隊，都有親自審閱過數百甚至數千條 trace 的 PM。

### 步驟一：產生多樣化的測試查詢

在審閱 trace 之前，你需要多樣化的測試輸入。一個強大的技術是**維度抽樣（dimensional sampling）**。

#### 定義關鍵維度

找出對你產品最重要的 3 到 4 個維度：

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

#### 產生隨機組合

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

#### 使用 LLM 將元組轉換為自然語言查詢

你可以使用任何 LLM 將維度元組轉換為真實的查詢。以下是各平台的做法：

**使用 LangWatch（內建生成功能）：**

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

**使用任何 LLM（不限平台）：**

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

**使用 Langfuse（手動追蹤）：**

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

**範例轉換：**

| 維度元組 | 生成的查詢 |
|---|---|
| vegan, Italian, dinner, beginner | 「嘿，我剛開始學做菜，而且是素食者。你能推薦一道簡單的義大利晚餐嗎？」 |
| gluten-free, any, dessert, intermediate | 「我在找一道有點難度的無麩質甜點食譜」 |
| keto, American, breakfast, advanced | 「給我一道複雜的生酮美式早餐食譜」 |

**給 PM/QA：** 這種維度化的方式能確保你涵蓋所有使用者需求的空間。若沒有這樣做，你只會測試到顯而易見的情境，而遺漏使用者將意想不到的需求組合在一起的邊緣案例。

### 步驟二：審閱 100 條 Trace 並做筆記（開放編碼）

**流程（每條 trace 約 30 秒）：**

1. 開啟你的 trace 查看器（LangWatch 儀表板、Langfuse UI 或任何工具）
2. 查看第一條 trace
3. 快速瀏覽：
   - 閱讀使用者訊息
   - 確認 AI 是否呼叫了正確的工具
   - 查看工具回傳的內容
   - 閱讀助理的回應
   - 記錄你發現的任何問題

**真實錯誤分析過程中的範例筆記：**

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

**錯誤分析的規則：**

1. **不要試圖記錄所有問題** — 只記錄最重要的事項
2. **不要在每條 trace 上糾結** — 快速思考、記下來、繼續往下
3. **略過系統提示** — 如果內容大多相同，不需要每次都讀
4. **進入心流狀態** — 這個過程應該感覺快速流暢，而不是枯燥乏味

**時間投入：**
- 第一條 trace：45 秒
- 第 10 條後：每條 25 秒
- 第 50 條後：每條 20 秒
- **100 條 trace 的總時間：約 45 分鐘**

**各平台注意事項：**
- **LangWatch：** 使用「Annotations」功能直接在 UI 中為 trace 添加筆記
- **Langfuse：** 使用「Comments」功能為 trace 添加筆記

### 步驟三：使用軸向編碼對錯誤進行分類

現在你手上有 40 到 50 條散落在各個 trace 中的筆記，是時候整理它們了。

這個流程稱為**「軸向編碼（axial coding）」**（源自社會學的研究方法）。你要將類似的錯誤歸納成同一個類別。

#### 使用 LLM 協助發掘類別

匯出你的筆記，然後使用以下提示：

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

**真實食譜機器人評估的範例結果：**

```
["Dietary Ignored", "Formatting Error", "Complexity Mismatch",
 "Meal Type Mismatch", "Ingredient Omission", "Skill Level Misalignment"]
```

#### 將類別細化，使其具體且可操作

**問題：** LLM 給出的通用建議太模糊了！

「時間性問題」— 這到底是什麼意思？
「品質問題」— 太籠統了！

**更好的類別（具體且可操作）：**

1. **飲食需求被忽略（Dietary Ignored）** — 機器人推薦了違反飲食限制的食材
2. **格式錯誤（Formatting Error）** — 在 SMS 中使用 Markdown，或結構錯誤
3. **難度不符（Complexity Mismatch）** — 食譜對使用者的技能等級來說太難或太簡單
4. **餐點類型不符（Meal Type Mismatch）** — 被問早餐卻推薦晚餐
5. **食材遺漏（Ingredient Omission）** — 未納入使用者指定的特殊食材
6. **技能等級不匹配（Skill Level Misalignment）** — 對初學者使用進階技巧

**你的類別必須具體到讓其他人也能根據它們為錯誤貼上標籤。**

### 步驟四：借助 LLM 為錯誤貼上標籤

這個步驟適用於任何 LLM。如果你的平台支援，可以使用批次處理：

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

**使用 LangWatch（批次評估）：**

```python
import langwatch

results = langwatch.evaluate.batch(
    dataset=error_notes_df,
    evaluator="custom_classifier",
    prompt_template=CLASSIFICATION_PROMPT,
    model="gpt-4o-mini"
)
```

**使用 Langfuse（手動迭代）：**

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

### 步驟五：統計並排定優先順序

**統計每個類別出現的次數：**

```python
label_counts = results["output"].value_counts()
```

**真實評估的範例結果：**

| 類別 | 次數 | 百分比 |
|----------|-------|------------|
| Complexity Mismatch | 2 | 22% |
| Meal Type Mismatch | 2 | 22% |
| Ingredient Omission | 2 | 22% |
| Dietary Ignored | 1 | 11% |
| Formatting Error | 1 | 11% |
| Skill Level Misalignment | 1 | 11% |

### 為什麼這能改變一切

**錯誤分析之前：**
- 你不知從何下手
- 不知道該先修復什麼
- 無法排定優先順序

**錯誤分析之後：**
- 根據頻率建立清晰的優先順序
- 了解嚴重程度（頻率與影響的權衡）
- 為利害關係人討論提供依據
- 具體列出需要建立評估機制的項目

**優先順序討論範例：**

```
"Dietary restriction violations happen in 11% of cases, but when
they occur, we could harm users with allergies. This is HIGH-SEVERITY.

Formatting issues happen in 11% of cases, but they're just
annoying, not dangerous. This is LOW-SEVERITY.

Let's fix dietary adherence first, then complexity matching."
```

### 「理論飽和度」的概念

**何時應該停止審閱 trace？**

在質性研究中，有一個概念叫做「理論飽和度（theoretical saturation）」— 即當你不再發現新類型的錯誤時。

- 審閱前 50 條 trace：你發現了 10 種不同的錯誤類型
- 再審閱 25 條：你發現了 2 種新的錯誤類型
- 再審閱 25 條：你沒有發現任何新的錯誤類型
- **停在這裡！** 你已達到飽和

如果在審閱 100 條後就找不到新模式，你不需要再審閱 1000 條。

### 給 PM/QA：你的錯誤分析檢查清單

1. 請工程師設置追蹤系統（LangWatch、Langfuse 或任何工具）
2. 開啟 trace 查看器 UI
3. 瀏覽 100 條 trace，快速記錄問題
4. 使用 LLM 協助將筆記歸納為 4 到 6 種故障模式
5. 統計每種故障模式的出現次數
6. 根據頻率和嚴重程度建立優先順序清單
7. 以數據支撐的建議向團隊呈現發現
8. 每月使用新的 trace 重複此流程，以發現新的故障模式

---

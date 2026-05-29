<a id="ai-evals-for-engineers-pms--qas-complete-study-guide"></a>
# AI 評測：工程師、PM 與 QA 完整學習指南

*本指南基於 Hamel Husain 與 Shreya Shankar 的 Maven 課程，並加入實作範例、可投入生產環境的程式碼，以及針對 LangWatch、Langfuse 等平台的專屬使用指南*

**本指南適合哪些人？**
- **工程師**：正在打造 AI 驅動產品並需要系統性評估品質的開發者
- **產品經理（PM）**：負責產品體驗並需要主導錯誤分析的人員
- **QA 工程師**：需要為 AI 系統建立自動化評測流程的測試人員
- **任何人**：想學習如何在不參加完整課程的情況下評測 AI 應用程式

**你將學到什麼：**
- 如何為任何 AI 應用程式設置可觀測性
- 如何系統性地找出問題所在（錯誤分析）
- 如何建立自動化評測器（基於程式碼與 LLM 裁判）
- 如何評測 RAG 系統、多步驟流程與多輪對話
- 如何執行生產環境評測：防護機制、安全性與即時監控
- 如何運用統計修正來應對裁判誤差
- 如何形成閉環：將評測結果轉化為系統改進
- 如何透過你選擇的可觀測性平台完成上述所有工作（LangWatch、Langfuse、Braintrust、LangSmith 或你自己的平台）

**平台範例：** 本指南以 **LangWatch**（開源、可自架或使用雲端版）和 **Langfuse**（開源、雲端版或可自架）作為主要範例。本指南的方法論與平台無關——可自行調整至你所使用的工具。

**LangWatch 與 Langfuse 比較：** 兩者都是具備相似核心功能的優秀開源平台。LangWatch 設置更簡單且內建評測器，而 Langfuse 在自訂流程方面更具彈性，且社群規模更大。本指南同時介紹兩者，讓你根據自身需求做出選擇。

---

<a id="table-of-contents"></a>
## 目錄

1. [什麼是 AI 評測以及為何需要它](#chapter-1)
2. [設置可觀測性](#chapter-2)
3. [錯誤分析：核心關鍵](#chapter-3)
4. [建立 LLM-as-a-Judge 評測器](#chapter-4)
5. [基於程式碼的評測器](#chapter-5)
6. [RAG 系統評測](#chapter-6)
7. [多步驟流程評測](#chapter-7)
8. [多輪對話評測](#chapter-8)
9. [生產環境評測：安全性、防護機制與監控](#chapter-9)
10. [使用 judgy 進行統計修正](#chapter-10)
11. [形成閉環：從評測到改進](#chapter-11)
12. [人工標注最佳實踐](#chapter-12)
13. [成本、延遲與評測擴展](#chapter-13)
14. [實作指南](#chapter-14)
15. [常見錯誤與規避方法](#chapter-15)
16. [工具與資源](#chapter-16)

**附錄：**
- [A：PM 與 QA 術語表](#appendix-a)
- [B：快速參考指南](#appendix-b)
- [C：來自生產環境的完整裁判提示詞](#appendix-c)
- [D：流程狀態評測器提示詞](#appendix-d)
- [E：裁判提示詞工程技巧](#appendix-e)
- [F：平台方法參考（LangWatch 與 Langfuse）](#appendix-f)
- [G：30 天學習路徑](#appendix-g)

---

<a name="chapter-1"></a>
<a id="chapter-1"></a>
## 第一章：什麼是 AI 評測以及為何需要它

<a id="simple-definition"></a>
### 簡單定義

**評測（Evals）** 是一種系統性測試，用以檢驗你的 AI 應用程式是否運作正常。可以把它想成是傳統軟體的單元測試，但應用於 AI 系統。

<a id="why-everyone-needs-evals"></a>
### 為何所有人都需要評測

AI 社群中存在一場爭論：有些人說「只要用感覺測試你的應用程式就好」（意思是：自己用看看，感覺不錯就行）。但事實是：

**所有人都需要評測。** 那些聲稱不需要評測的人，其實是在受益於別人在上游所做的評測。

範例：如果你正在用 GPT-4 建立一個程式碼助理，OpenAI 已經在大量程式碼基準上測試過 GPT-4 了。所以你可以「用感覺測試」你的應用程式。但對於大多數非單純使用基礎模型的應用程式，你需要自己的評測。

<a id="the-three-core-truths-about-evals"></a>
### 評測的三大核心真理

1. **無法量測的，就無法改進**
   - 像「有用性分數」這樣的通用指標無法發現特定問題
   - 你需要針對應用程式的專屬評測

2. **錯誤分析是最重要的步驟**
   - 比 LLM 裁判更重要
   - 比花俏的可觀測性工具更重要
   - 這才是真正找出問題所在的地方

3. **PM 和 QA 必須主導錯誤分析，而不只是工程師**
   - 工程師知道程式碼是否正常運作
   - PM 知道產品體驗是否良好
   - QA 知道如何系統性地找出問題
   - 你擁有領域專業知識
   - 這是產品工作，不只是技術工作

<a id="the-ai-development-cycle-is-the-scientific-method"></a>
### AI 開發週期就是科學方法

打造優秀的 AI 產品需要嚴謹的評測流程。在許多方面，AI 開發本身就是科學方法：

1. **觀察** — 追蹤 AI 的行為（第二章）
2. **假設** — 透過錯誤分析找出問題所在（第三章）
3. **實驗** — 建立評測器並測試變更（第四至九章）
4. **量測** — 計算指標並修正偏差（第十章）
5. **迭代** — 根據資料而非直覺進行改進（第十一章）

<a id="what-can-go-wrong-without-evals"></a>
### 沒有評測會出什麼問題？

你的 demo 效果很好。然後生產環境來了：

- 使用者遇到你從未想到的邊界案例
- 簡訊中包含錯字和不尋常的格式
- 日期格式與預期不同
- AI 試圖處理本應轉交給人工的請求
- 小幅修改提示詞就導致原本正常的功能失效

**真實生產資料範例：**
```
User: "I need a one bedroom with the bathroom NOT connected"
AI: Returns apartments with connected bathrooms (WRONG!)
User: "I do NOT want a bathroom connected to the room"
AI: "I'll check on that" but never actually checks
PLUS: AI used markdown formatting (* asterisks *) in a text message
```

一次互動中出現了三種不同的問題！沒有適當的日誌記錄和評測，你永遠不會發現這些規律。

<a id="for-pms-why-this-is-your-job"></a>
### 致 PM：為何這是你的工作

**錯誤的做法：** 「這是技術性的 AI 相關工作，讓工程師去搞定」

**正確的做法：** PM 應該主導錯誤分析，因為：
1. 你了解使用者需求
2. 你對產品有品味
3. 你擁有領域專業知識
4. 這是以技術工作為外表的產品工作

**交付最佳 AI 產品的團隊，其 PM 都親自審閱過數百甚至數千條 trace。**

<a id="for-qas-your-new-superpower"></a>
### 致 QA：你的新超能力

傳統 QA 包含具有預期輸出的測試案例。AI 的 QA 不同：
1. 輸出是不確定的（相同輸入可能產生不同輸出）
2. 「正確」通常是主觀的
3. 邊界案例幾乎是無窮無盡的
4. 你需要能夠擴展的自動化評測器

但 QA 的核心思維——系統性測試、邊界案例思考、迴歸預防——正是 AI 評測所需要的。學會評測的 QA 工程師會變得極具價值。

---

<a name="chapter-2"></a>
<a id="chapter-2"></a>
<a id="setting-up-observability"></a>
## 第二章：設置可觀測性

<a id="what-is-a-trace"></a>
### 什麼是 Trace？

**Trace** 是 AI 為回應使用者所做的一切完整記錄。就像一份詳細的日誌，顯示：

1. **系統提示詞**（給予 AI 的指令）
2. **使用者訊息**（使用者詢問的內容）
3. **工具呼叫**（AI 嘗試使用的函式）
4. **工具回應**（那些函式回傳的內容）
5. **助理回應**（AI 的回覆）
6. **所有上下文**（LLM 在做決策時所看到的一切）

<a id="example-of-a-complete-trace"></a>
### 完整 Trace 範例

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

<a id="what-information-to-capture"></a>
### 應捕獲哪些資訊

**最低需求：**
- 輸入（使用者訊息）
- 輸出（AI 回應）
- 時間戳記
- 互動的唯一 ID

**建議納入：**
- 所使用的系統提示詞
- 工具呼叫及其結果
- 模型參數（temperature、max_tokens 等）
- Token 計數
- 延遲（回應時間）
- 每次請求的成本

**最佳實踐：**
- 使用者上下文（對話歷史）
- 若有發生的錯誤訊息
- 使用的模型版本
- 當時啟用的功能旗標

<a id="choosing-an-observability-platform"></a>
### 選擇可觀測性平台

| 工具 | 類型 | 最適用於 | 費用 |
|------|------|----------|------|
| **LangWatch** | 開源、雲端或自架 | 設置簡單、內建評測器、絕佳使用者體驗 | 免費方案＋付費方案 |
| **Langfuse** | 開源、雲端或自架 | 自訂流程、龐大社群 | 免費方案＋付費方案 |
| **Braintrust** | 雲端 | 優秀的使用者介面、團隊協作 | 付費 |
| **LangSmith** | 雲端 | LangChain 使用者 | 付費 |
| **自行建置** | 客製化 | 學習、自訂需求 | 免費 |

**LangWatch 與 Langfuse 比較：**
- **設置：** LangWatch 更簡單（3 行整合），Langfuse 需要更多設定
- **評測器：** LangWatch 內建 40 個以上的評測器，Langfuse 需要自訂實作
- **彈性：** Langfuse 對自訂工作流程更靈活，LangWatch 則更有既定主張
- **社群：** Langfuse 擁有更龐大的社群和更多整合方式
- **使用者介面：** 兩者都有優秀的介面；LangWatch 專注於分析，Langfuse 專注於工作流程

所有這些平台都支援相同的核心概念：trace、span、資料集、評測和實驗。本指南的方法論適用於其中任何一個。

<a id="setting-up-langwatch-open-source-cloud-or-self-hosted"></a>
### 設置 LangWatch（開源、雲端或自架）

LangWatch 是一個開源的 LLM 可觀測性與分析平台。它提供追蹤、評測、資料集、實驗，以及 40 個以上的內建評測器。

<a id="install-and-configure"></a>
#### 安裝與設定

```bash
pip install langwatch
```

```python
# Set your API key (get one at langwatch.ai or self-host)
import os
os.environ["LANGWATCH_API_KEY"] = "lw_..."  # or set in .env file
```

**雲端版與自架版：**
- **雲端版：** 在 [langwatch.ai](https://langwatch.ai) 註冊、取得 API 金鑰，5 分鐘內完成
- **自架版：** 使用其 Docker 設定執行 `docker-compose up`，並指向你自己的執行個體

<a id="instrument-your-application-auto-tracing"></a>
#### 為應用程式加入追蹤（自動追蹤）

LangWatch 支援大多數框架的自動追蹤：

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

**框架支援：**
- OpenAI（自動）
- LangChain（自動）
- LlamaIndex（自動）
- Anthropic Claude（自動）
- 任何自訂 LLM（手動 span）

<a id="add-custom-spans-with-decorators"></a>
#### 使用裝飾器新增自訂 Span

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

**與 Langfuse 比較：**
兩者都使用裝飾器，但 LangWatch 的 `@langwatch.span()` 比 Langfuse 的 `@observe()` 更簡單。LangWatch 自動依類型分類 span，而 Langfuse 需要明確指定 `as_type` 參數。

<a id="setting-up-langfuse-open-source-cloud-or-self-hosted"></a>
### 設置 Langfuse（開源、雲端或自架）

Langfuse 提供追蹤、評測、資料集、實驗與提示詞管理。它提供託管雲端版和自架選項。

<a id="install-and-configure-langfuse"></a>
#### 安裝與設定

```bash
pip install langfuse openai
```

```python
# Set environment variables (or pass to constructor)
# LANGFUSE_SECRET_KEY="sk-lf-..."
# LANGFUSE_PUBLIC_KEY="pk-lf-..."
# LANGFUSE_HOST="https://cloud.langfuse.com"  # or your self-hosted URL
```

<a id="instrument-your-application-drop-in-replacement"></a>
#### 為應用程式加入追蹤（即插即用替換）

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

<a id="add-custom-spans-with-decorators-langfuse"></a>
#### 使用裝飾器新增自訂 Span

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

<a id="creating-and-managing-prompts"></a>
### 建立與管理提示詞

兩個平台都支援版本化的提示詞管理：

<a id="langwatch-prompts"></a>
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

**LangWatch 優勢：** 更簡潔的 API，自動管理參數（溫度、模型與提示詞一起儲存）。

<a id="langfuse-prompts"></a>
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

**Langfuse 優勢：** 更成熟的提示詞管理、更好的版本控制介面、用於組織管理的標籤功能。

<a id="uploading-test-datasets"></a>
### 上傳測試資料集

<a id="langwatch-datasets"></a>
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

**LangWatch 優勢：** 直接支援 pandas DataFrame，API 更簡潔。

<a id="langfuse-datasets"></a>
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

**Langfuse 優勢：** 對個別項目有更多控制，更適合增量新增。

<a id="key-principle"></a>
### 核心原則

**沒有 trace，就無法進行評測。** 這是你的基礎。在做任何其他事情之前，先設置好這一點。

**致 PM/QA：** 你不需要自己撰寫追蹤程式碼。請工程師設置追蹤，然後使用網頁介面視覺化審閱 trace。LangWatch（`langwatch.ai` 或你的自架 URL）和 Langfuse（`cloud.langfuse.com` 或你的自架 URL）都提供了讓你無需撰寫任何程式碼，即可瀏覽、搜尋和標注 trace 的介面。

**平台選擇指引：**
- 選擇 **LangWatch** 如果：你想要最快速的設置、內建評測器，並專注於分析
- 選擇 **Langfuse** 如果：你需要最大彈性、有複雜的自訂工作流程，或想要最大的社群支援
- **兩者都用**：它們相輔相成——LangWatch 用於快速評測，Langfuse 用於深度工作流程自訂

---

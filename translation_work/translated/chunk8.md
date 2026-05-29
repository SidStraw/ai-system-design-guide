<a name="appendix-f"></a>
## 附錄 F：平台方法參考（LangWatch 與 Langfuse）

### LangWatch

#### 追蹤

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

#### 查詢 Span

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

#### 資料集

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

#### 評估器

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

#### 實驗

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

#### 提示詞管理

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

#### 追蹤

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

#### 查詢追蹤

```python
from langfuse import get_client

langfuse = get_client()

traces = langfuse.api.trace.list(limit=100, tags=["production"])
trace = langfuse.api.trace.get("trace_id")
```

#### 資料集

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

#### 實驗

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

#### 分數（評估結果）

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

#### 提示詞管理

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
## 附錄 G：30 天學習路徑

### 第一週：基礎（工程師、PM 或 QA）

| 天 | 活動 | 時間 | 角色重點 |
|-----|----------|------|------------|
| 1 | 選擇你的平台（LangWatch 或 Langfuse），安裝它 | 1 小時 | 全部 |
| 2 | 為你的應用程式加入自動追蹤 | 2 小時 | 工程師 |
| 2 | 瀏覽追蹤查看器 UI，以視覺方式理解追蹤 | 1 小時 | PM/QA |
| 3 | 以維度取樣建立測試資料集 | 2 小時 | 全部 |
| 4 | 將資料集上傳至平台，執行第一個實驗 | 1 小時 | 全部 |
| 5 | 查看 50 筆追蹤，記錄筆記（開放式編碼） | 1 小時 | 全部 |
| 6 | 使用 LLM 分類錯誤（軸向編碼） | 1 小時 | 全部 |
| 7 | 優先排序：頻率 × 嚴重性矩陣 | 30 分鐘 | 全部 |

### 第二週：程式碼型評估

| 天 | 活動 | 時間 | 角色重點 |
|-----|----------|------|------------|
| 8 | 針對你的主要問題建立 2 個程式碼型評估 | 2 小時 | 工程師 |
| 8 | 以白話文定義評估標準 | 1 小時 | PM/QA |
| 9 | 以已知好/壞案例測試評估 | 1 小時 | 全部 |
| 10 | 對所有追蹤執行評估，計算失敗率 | 1 小時 | 全部 |
| 11-14 | 根據結果迭代 | 2 小時 | 全部 |

### 第三週：LLM 裁判

| 天 | 活動 | 時間 | 角色重點 |
|-----|----------|------|------------|
| 15 | 標記 100-150 筆追蹤作為基準真相 | 3 小時 | 全部 |
| 16 | 分割為訓練/開發/測試集 | 30 分鐘 | 工程師 |
| 17 | 撰寫第一個含少量示例的裁判提示詞 | 2 小時 | 全部 |
| 18 | 在開發集上驗證，計算 TPR/TNR | 1 小時 | 全部 |
| 19 | 迭代提示詞直到指標 > 80% | 2 小時 | 全部 |
| 20 | 在測試集上進行最終測試 | 30 分鐘 | 全部 |
| 21 | 對所有追蹤執行裁判 + 用 judgy 修正 | 1 小時 | 全部 |

### 第四週：進階主題與生產環境

| 天 | 活動 | 時間 | 角色重點 |
|-----|----------|------|------------|
| 22 | RAG 評估——檢索指標 + 答案品質（第 6 章） | 2 小時 | 工程師 |
| 23 | 多步驟管道評估（第 7 章） | 2 小時 | 工程師 |
| 24 | 多輪對話評估（第 8 章） | 2 小時 | 工程師 |
| 25 | 安全評估——提示詞注入、PII 洩漏（第 9 章） | 2 小時 | 全部 |
| 26 | 建立回歸測試套件（第 11 章） | 2 小時 | 工程師 |
| 27 | 人工標注校正——測量標注者間一致性（第 12 章） | 1 小時 | 全部 |
| 28 | 成本優化——分層評估、取樣策略（第 13 章） | 1 小時 | 全部 |
| 29 | 建立監控儀表板 + 自動化評估執行 | 2 小時 | 工程師 |
| 30 | 記錄評估套件，向利害關係人呈現，規劃維護計畫 | 2 小時 | 全部 |

---

## 學到的教訓

在生產環境中實作完整評估管道的真實教訓：

**關於建立裁判（第 4、10 章）**

1. **LLM 作為裁判功能強大，但需要防護機制** — 若缺乏適當驗證，裁判可能會自信地給出錯誤答案。請務必對照基準真相進行驗證。

2. **你必須對照基準真相測試評估器** — 一個看起來合理但 TNR=22% 的裁判實際上是有害的——它會漏掉大多數真實失敗。

3. **訓練/開發/測試集分割能建立信心** — 沒有它們，你就是在自欺欺人地評估裁判品質。這是不可妥協的。

4. **迭代裁判提示詞至關重要** — 第一個提示詞永遠不夠好。至少計劃進行 3-5 次迭代。技巧請參閱附錄 E。

5. **先解釋後裁決是第一名技巧** — 要求裁判在標記前先推理，比任何其他單一改變都能提升更多準確性。

**關於流程與方法論（第 3、11、12 章）**

6. **錯誤分析才是真正的工作** — 若你沒有靜下心來檢視自己的失敗，再花俏的工具也無濟於事。開放式編碼 → 軸向編碼 → 優先排序，這是有效的工作流程。

7. **人工標注者的分歧比你想像的多** — 在信任你的基準真相之前，先測量標注者間一致性（Cohen's kappa）。若人類無法達成共識，裁判也不會。

8. **閉環才是區分優秀團隊與卓越團隊的關鍵** — 執行評估只是工作的一半。另一半是系統性地將失敗轉化為改進，並防止回歸。

**關於生產環境與規模化（第 9、13 章）**

9. **安全評估不是可選的** — 提示詞注入、PII 洩漏和越獄偵測應在你擔心品質評估之前就開始運行。

10. **先花費高昂，再優化** — 使用 GPT-4o/Claude Sonnet 建立你的性能上限，然後測試較便宜的模型是否能達到。通常可以。

11. **取樣勝過窮舉評估** — 以嚴格的統計方法評估 10% 的追蹤，比用糟糕的裁判評估 100% 能給出更好的答案。

12. **良好的可觀測性工具讓工作流程快 10 倍** — 在一個平台（LangWatch、Langfuse 等）中整合追蹤、評估、資料集和實驗，與拼湊自訂腳本相比，可節省大量時間。

**關於平台選擇**

13. **LangWatch 求速，Langfuse 求深** — LangWatch 透過內建評估器在數小時內給你結果。Langfuse 為複雜的自訂邏輯提供最大控制。許多團隊兩者都使用。

14. **內建評估器節省數週的開發時間** — LangWatch 的 40+ 個內建評估器涵蓋了大多數常見使用案例。若你在重新發明安全檢查或 RAG 指標，你是在浪費時間。

15. **社群對長期成功至關重要** — Langfuse 更大的社群意味著更多整合、更多範例、更多支援。LangWatch 更簡單的 API 意味著更快速的入門。

---

## 結論

AI 評估不只是「測試」——它是一種涉及工程、產品管理和品質保證的產品開發方法論。

**重點摘要：**

1. **每個人都需要評估** — 不只是大公司。若你的 AI 應用程式接觸到用戶，你就需要系統性的評估。
2. **從錯誤分析開始** — 在建立任何自動化之前，先靜下心來檢視你的失敗（第 3 章）。
3. **PM 和 QA 必須主導** — 錯誤分析和標準定義是產品/品質工作，不只是工程任務。
4. **逐步建立** — 從程式碼型評估開始，然後加入 LLM 裁判，再加入安全評估。不要試圖一次完成所有事情。
5. **測量重要的事** — 應用程式特定標準，而非泛用的「有用性」分數。
6. **TPR 和 TNR 兩者都要** — 一個能捕捉失敗但也會誤報的裁判是有害的。兩者都要測量。
7. **分割你的資料** — 訓練/開發/測試集是必要的。沒有它，你的裁判會過度擬合。
8. **修正偏差** — 使用統計修正（第 10 章）以獲得誠實的指標。
9. **閉環** — 不導向改進的評估是浪費的努力（第 11 章）。
10. **規劃規模化** — 從最好的模型開始，然後優化成本（第 13 章）。

**你的行動計劃（詳見附錄 G）：**

1. 第一週：設定可觀測性（LangWatch 或 Langfuse），進行錯誤分析
2. 第二週：建立 2-3 個核心程式碼型評估
3. 第三週：建立並驗證含適當訓練/開發/測試集分割的 LLM 裁判
4. 第四週：進階主題——RAG 評估、多輪評估、安全評估、自動化
5. 持續：每週 30 分鐘維護 + 回歸測試

**平台決策：**
- 若你想快速開始（設定 < 30 分鐘）並使用內建評估器，請選擇 **LangWatch**
- 若你需要最大靈活性且有複雜的自訂工作流程，請選擇 **Langfuse**
- 若你想兼得兩者的優勢，請**兩者都使用**（許多團隊這樣做）

**記住：** 推出最佳 AI 產品的團隊是擁有最佳評估的團隊。不是最花俏的模型。不是最大的團隊。而是那些系統性地測量和改進的人。

從今天開始。未來的你會感謝你。

---

## 學習資源

### 平台文件與學習中心

- **LangWatch 文件**：[docs.langwatch.ai](https://docs.langwatch.ai)
- **LangWatch GitHub**：[github.com/langwatch/langwatch](https://github.com/langwatch/langwatch)
- **Langfuse 文件**：[langfuse.com/docs](https://langfuse.com/docs)
- **Langfuse GitHub**：[github.com/langfuse/langfuse](https://github.com/langfuse/langfuse)
- **Maven 課程（AI Evals for Engineers & PMs）**：[maven.com/parlance-labs/evals](https://maven.com/parlance-labs/evals)
- **HuggingFace 評估指南**：[github.com/huggingface/evaluation-guidebook](https://github.com/huggingface/evaluation-guidebook)

### 研究與思想領導力

- **OpenAI Evals 平台**：[evals.openai.com](https://evals.openai.com/)
- **OpenAI Cookbook**（實務範例與指南）：[cookbook.openai.com](https://cookbook.openai.com/)
- **OpenAI 研究**：[openai.com/research](https://openai.com/research)
- **OpenAI 文件（評估）**：[platform.openai.com/docs/guides/evals](https://platform.openai.com/docs/guides/evals)
- **Anthropic 研究**：[anthropic.com/research](https://www.anthropic.com/research)
- **METR**（模型評估與威脅研究）：[metr.org](https://metr.org/)
- **Eugene Yan 談評估流程**：[eugeneyan.com/writing/eval-process](https://eugeneyan.com/writing/eval-process/)

### 塑造本指南的部落格

- **Hamel Husain 的部落格**：[hamel.dev](https://hamel.dev/) — 應用 AI 工程、LLM 評估深度探討
- **Shreya Shankar 的網站**：[sh-reya.com](https://www.sh-reya.com/) — LLM 資料系統研究、評估方法論
- **Maxim AI 文章**：[getmaxim.ai/articles](https://www.getmaxim.ai/articles) — 代理評估模式

### 開源工具與函式庫

| 工具 | 重點 | 授權 | 連結 |
|------|-------|---------|-------|
| **LangWatch** | 可觀測性與內建評估 | Apache 2.0 | [GitHub](https://github.com/langwatch/langwatch) · [文件](https://docs.langwatch.ai) |
| **Langfuse** | 自訂管道與追蹤 | MIT | [GitHub](https://github.com/langfuse/langfuse) · [文件](https://langfuse.com/docs) |
| **RAGAS** | RAG 專用評估 | Apache 2.0 | [GitHub](https://github.com/explodinggradients/ragas) · [文件](https://docs.ragas.io/) |
| **Comet Opik** | LLM 追蹤與評估 | Apache 2.0 | [GitHub](https://github.com/comet-ml/opik) · [網站](https://www.comet.com/site/products/opik/) |
| **judgy** | 統計偏差修正 | 開放 | [GitHub](https://github.com/ai-evals-course/judgy) |
| **Braintrust** | 實驗與記錄 | 部分 | [文件](https://www.braintrust.dev/docs) |
| **Galileo** | 幻覺偵測 | 專有 | [網站](https://www.galileo.ai/) |
| **Maxim** | 代理系統評估 | 專有 | [網站](https://www.getmaxim.ai/) |

### 策略比較矩陣

| 公司 | 重點 | 開源 | 最適合 | 獨特優勢 |
|---------|-------|-------------|----------|-----------------|
| **LangWatch** | 可觀測性 + 內建評估 | 是（Apache 2.0） | 快速設置、分析 | 40+ 個內建評估器、自動分析 |
| **Langfuse** | 自訂管道 | 是（MIT） | 資料主權、靈活性 | 可自主託管，完全掌控資料 |
| **Anthropic** | 安全性 / 紅隊測試 | 部分 | 前沿風險 | 憲法分類器、多次嘗試對抗性測試 |
| **OpenAI** | 準備度 / 商業 | 評估工具包 | 企業情境 | 中小企業探測、情境評估 |
| **RAGAS** | RAG 專用 | 是（Apache 2.0） | RAG 管道 | 無參考指標、合成測試資料生成 |
| **Maxim** | 代理系統 | 否 | 多代理應用程式 | 模擬框架、無程式碼評估 |
| **Braintrust** | 實驗 | 部分 | 早期團隊 | 協作設計、快速迭代 |
| **Galileo** | 幻覺 | 否 | 品質保證 | ChainPoll、即時監控 |
| **Comet Opik** | LLM 追蹤與評估 | 是（Apache 2.0） | 端到端可觀測性 | 框架整合、線上評估規則 |
| **METR** | 災難性風險 | 研究 | 政策指導 | 自主能力評估 |

### 聯絡我
- Om Bharatiya：[@ombharatiya](https://twitter.com/ombharatiya)

### 參考工作致謝
本指南建立在以下人士的工作和想法之上。他們的課程、部落格和開源貢獻使本指南成為可能：
- Hamel Husain：[@HamelHusain](https://x.com/HamelHusain) — [hamel.dev](https://hamel.dev/)
- Shreya Shankar：[@sh_reya](https://x.com/sh_reya) — [sh-reya.com](https://www.sh-reya.com/)
- Eugene Yan：[@eugeneyan](https://x.com/eugeneyan) — [eugeneyan.com](https://eugeneyan.com/)

---

*本指南受 Hamel Husain 和 Shreya Shankar 的 AI Evals for Engineers & PMs 課程啟發並在其基礎上構建，並延伸了額外研究、生產就緒的程式碼範例，以及涵蓋 LangWatch、Langfuse 和更廣泛評估工具生態系統的多平台指南。*

*作者：Om Bharatiya | 建立日期：2026 年 2 月*

<a name="chapter-10"></a>
## 第十章：使用 judgy 進行統計校正

### 問題所在：您的評審並非完美

即使是優秀的評審也會犯錯。若您的評審具備以下指標：
- TPR = 95.7%（漏判 4.3% 的真實通過案例）
- TNR = 100%（從不漏判真實失敗案例）

那麼評審所得出的原始通過率就會略有偏差。

### 什麼是 judgy？

[judgy](https://github.com/ai-evals-course/judgy) 是一個 Python 函式庫，使用統計方法校正評審誤差。它需要以下輸入：

1. **測試標籤**（來自您已標記資料的真實答案）
2. **測試預測**（評審對已標記資料的判斷結果）
3. **未標記預測**（評審對所有生產環境追蹤記錄的判斷結果）

並回傳帶有置信區間的校正後成功率。

### 如何使用 judgy

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

### 實際結果：校正前後對比

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

**為何校正很重要：** 原始比率（84.4%）低估了真實表現，因為評審存在輕微的偽陰性傾向（TPR=95.7%，而非 100%）。校正後的比率（88.2%）已將此偏差納入考量。

### 平台整合

**平台無關性：** `judgy` 可與任何平台的結果搭配使用。匯出您的測試集結果與生產環境預測，然後執行校正：

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

### 給 PM 的建議：如何呈報結果

向利害關係人報告時：

```
"Our Recipe Bot correctly follows dietary restrictions 88% of the time,
with 95% confidence that the true rate is between 84% and 99%.

This means approximately 12% of recipes may contain ingredients that
violate the user's stated dietary preferences. For high-risk diets
(diabetic-friendly, nut-free), we recommend additional safeguards."
```

這比「我們測試過，感覺沒問題」要可信得多。

---

<a name="chapter-11"></a>
## 第十一章：閉環——從評估到改進

### 最常見的失敗：只量測不行動

許多團隊建立了完善的評估套件，卻從未系統性地利用結果改善系統。評估只有在驅動行動時才有價值。

### 改進循環

```
1. Run evals → identify top failure mode
2. Root-cause the failure (is it prompt? retrieval? tool? data?)
3. Implement a fix (change prompt, add guardrail, fix tool)
4. Run evals again → confirm improvement, check for regressions
5. Repeat with the next failure mode
```

### 追溯失敗根因

當評估識別出失敗時，詢問失敗**發生在管線的哪個環節**：

| 失敗位置 | 症狀 | 典型修復方式 |
|---|---|---|
| **系統提示詞** | 語調錯誤、缺少功能、違反政策 | 編輯提示詞、新增範例、增加限制條件 |
| **檢索** | 文件錯誤、上下文缺失 | 改善分塊方式、重新排序、擴展查詢 |
| **工具呼叫** | 選錯工具、參數錯誤 | 改善工具描述、新增驗證機制 |
| **生成** | 幻覺、格式錯誤、忽略上下文 | 少量範例、結構化輸出、調整溫度 |
| **後處理** | 截斷、編碼問題、格式錯誤 | 修復解析程式碼、新增驗證 |

### 回歸測試

每次修復某個問題時，都有可能破壞其他功能。請建立回歸測試：

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

**平台對回歸測試的支援：**

**LangWatch：** 內建回歸測試套件，可自動與基準執行結果進行比較。

**Langfuse：** 透過資料集進行手動追蹤，回歸偵測需要自訂邏輯。

### 使用評估進行模型比較

評估是否要切換模型時（例如 GPT-4o vs. Claude vs. Gemini）：

```python
MODELS = ["gpt-4o", "claude-sonnet-4-5-20250929", "gemini-2.0-flash"]

for model in MODELS:
    results = run_eval_suite(model=model, test_set=test_data)
    print(f"{model}: TPR={results['tpr']:.1%}, TNR={results['tnr']:.1%}, "
          f"cost=${results['cost']:.2f}, latency={results['latency_p50']:.0f}ms")
```

### 給 PM 的建議：改進劇本

每個評估週期結束後，製作一份簡單報告：

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
## 第十二章：人工標注最佳實踐

### 人工標籤勝過 LLM 標籤的情境

- **模糊案例**——即使是專家也意見分歧，您需要記錄這種分歧
- **高風險領域**（醫療、法律、金融），錯誤會帶來真實後果
- **新型失敗模式**——您的 LLM 評審尚未受過訓練而無法偵測的情況
- **真實答案校準**——即使您以大規模 LLM 標記為主，也應手動驗證一個樣本

### 標注者間一致性

若兩個人對同一標籤意見相左，代表您的評估標準還不夠清晰。

**流程：**
1. 讓 2-3 人獨立標注同 50 筆追蹤記錄
2. 計算一致率（相同比例）
3. 若一致率 < 80%，標準需要更具體
4. 討論分歧、更新標準、重新標注

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

### 標籤品質 > 標籤數量

**50 個高品質標籤勝過 500 個低品質標籤。** 請投入時間於：
1. 清晰、書面的標注指引，並附上範例
2. 邊緣案例說明文件（「若遇到 X，請標記為 Y，因為……」）
3. 定期校準會議，讓標注者一起討論分歧

### 給 PM／QA 的建議：您是最佳標注者

PM 和 QA 往往能提供比工程師更好的標籤，因為：
- 您知道良好使用者體驗的樣貌
- 您了解產品的政策與限制
- 您從使用者的角度思考，而非從程式碼的角度

---

<a name="chapter-13"></a>
## 第十三章：成本、延遲與評估擴展

### 成本問題

以 GPT-4o 作為評審，對 10,000 筆追蹤記錄進行評估的代價高昂。以下是管控成本的方法：

### 策略一：使用更便宜的模型作為評審

並非每個評估都需要最佳模型：

| 評審模型 | 費用（每 1K 筆追蹤記錄） | 適用時機 |
|---|---|---|
| GPT-4o / Claude Opus | ~$5-15 | 複雜的主觀判斷、安全關鍵情境 |
| GPT-4o-mini / Claude Haiku | ~$0.50-1.50 | 明確標準、定義清晰的評分規則 |
| 程式碼判斷 | $0 | 格式檢查、模式比對、驗證 |

**小提示：** 先用強力模型驗證您的評審提示詞，再測試較便宜的模型是否能達到相近的 TPR/TNR。通常可以。

### 策略二：抽樣而非全量評估

您不需要評估每一筆追蹤記錄：

```python
import random

def sample_traces(traces, sample_rate=0.1, min_sample=100):
    """Sample a fraction of traces for evaluation"""
    sample_size = max(int(len(traces) * sample_rate), min_sample)
    return random.sample(traces, min(sample_size, len(traces)))

# 10% sample of 50,000 daily traces = 5,000 evals
# Statistical confidence is still high with proper sampling
```

### 策略三：分層評估

對所有追蹤記錄執行廉價評估，只對樣本執行昂貴評估：

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

### 策略四：快取重複評估

若相同輸入多次出現，快取評估結果：

```python
import hashlib

eval_cache = {}

def cached_eval(trace, eval_fn):
    key = hashlib.md5(str(trace['input'] + trace['output']).encode()).hexdigest()
    if key not in eval_cache:
        eval_cache[key] = eval_fn(trace)
    return eval_cache[key]
```

**平台對快取的支援：**

**LangWatch：** 內建評估快取，自動去除重複的相同追蹤記錄。

**Langfuse：** 需要手動快取，但支援透過 metadata 自訂快取鍵值。

### 即時防護措施的延遲考量

| 檢查類型 | 典型延遲 | 適合即時使用？ |
|---|---|---|
| 正則表達式／程式碼檢查 | <1ms | 是 |
| 嵌入向量相似度 | 10-50ms | 是 |
| 小型 LLM（Haiku 等級） | 200-500ms | 勉強（會增加明顯延遲） |
| 大型 LLM（GPT-4o 等級） | 1-3s | 否（僅適用於離線） |

---

<a name="chapter-14"></a>
## 第十四章：實用實施指南

### 評估的前兩週計畫

### 第一週：奠定基礎

#### 第 1-2 天：設定日誌記錄（4 小時）

**目標：** 捕捉每次 AI 互動的追蹤記錄。

選擇您的平台並進行設定：

**LangWatch：**
```bash
pip install langwatch
# Sign up at langwatch.ai or run self-hosted Docker
```

```python
import langwatch
langwatch.init()  # That's it! Auto-instrumentation enabled
```

**Langfuse：**
```bash
pip install langfuse openai
# Sign up at cloud.langfuse.com or self-host
```

```python
from langfuse.openai import OpenAI  # Drop-in replacement
client = OpenAI()  # Auto-traced
```

接著對您的應用程式進行儀器化（完整範例請參見第二章）。

**交付成果：** 每次 AI 互動皆已記錄，並可在您的可觀測性 UI 中查看。

#### 第 3 天：手動錯誤分析（3 小時）

**目標：** 審查 100 筆追蹤記錄並做筆記。

1. 開啟您的追蹤記錄檢視器（LangWatch 或 Langfuse UI）
2. 瀏覽追蹤記錄
3. 在試算表或 CSV 中記錄問題
4. 每筆追蹤記錄預算 30-60 秒

**交付成果：** 從 100 筆追蹤記錄中得到 40-50 條錯誤筆記。

#### 第 4 天：分類錯誤（2 小時）

**目標：** 將您的筆記分組為 5-6 個類別。

1. 匯出您的筆記
2. 使用 LLM 建議類別
3. 將類別精鍊為具體且可執行的內容
4. 為每條筆記標記類別
5. 統計出現次數

**交付成果：** 已依優先順序排列的問題清單，附帶頻率資料。

#### 第 5-7 天：建立第一個評估（6 小時）

**目標：** 建立一個程式碼評估和一個 LLM 評審。

**程式碼評估（第 5 天）：** 挑選出現頻率最高的客觀問題。

**LLM 評審（第 6-7 天）：**
1. 撰寫含標準與範例的評審提示詞
2. 將 50-100 筆追蹤記錄標記為真實答案
3. 分為訓練集／開發集／測試集
4. 在開發集上驗證（反覆修改提示詞，直到 TPR/TNR > 80%）
5. 在測試集上測試以獲取最終指標

**交付成果：** 兩個可對新追蹤記錄執行的有效評估。

### 第二週：自動化與監控

#### 第 8-9 天：自動化評估執行

**With LangWatch：**
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

**With Langfuse：**
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

#### 第 10-11 天：設定警報

```python
def check_for_degradation(current_rate, historical_avg, threshold=1.5):
    """Alert if failure rate spikes"""
    return current_rate > historical_avg * threshold

# Example alert
if check_for_degradation(today_failure_rate, avg_failure_rate):
    send_slack_alert("Eval failure rate spiked!")
```

**LangWatch：** 內建警報功能，可在指標超過閾值時透過電子郵件、Slack 或 webhook 發送通知。

**Langfuse：** 自訂警報需要與您的監控系統整合。

#### 第 12-14 天：儀表板

**LangWatch：** 內建分析儀表板，無需設定。

**Langfuse：** 使用其 API 建立自訂儀表板：
```python
# Fetch recent scores
scores = langfuse.api.score.list(limit=1000, from_timestamp=last_week)

# Aggregate and visualize
failure_rates = aggregate_by_day(scores)
plot_dashboard(failure_rates)
```

### 持續進行：每週 30 分鐘

**每週一（15 分鐘）：**
1. 檢查您的可觀測性 UI 是否有異常
2. 審查上週的任何警報
3. 記錄模式

**每月（2 小時）：**
1. 對 50 筆新追蹤記錄進行錯誤分析
2. 尋找新的失敗模式
3. 必要時新增評估
4. 淘汰從未觸發的評估

**重大變更後（1 小時）：**
1. 執行完整評估套件
2. 與基準進行比較
3. 調查任何回歸情況

---

<a name="chapter-15"></a>
## 第十五章：應避免的常見錯誤

### 錯誤 #1：跳過錯誤分析

**常見做法：** 直接跳到建立 LLM 評審或儀表板。
**為何有問題：** 您還不知道要量測什麼。
**修正方式：** 永遠從錯誤分析開始。花時間審查追蹤記錄。

### 錯誤 #2：僅使用一致率進行驗證

**常見做法：** 「我的評審與人類的一致率達 90%，可以上線了！」
**為何有問題：** 在失敗案例稀少時，一個永遠說「通過」的評審也能達到 90% 一致率。
**修正方式：** 永遠分別計算 TPR 與 TNR。兩者都必須高。

### 錯誤 #3：PM／QA 委外處理錯誤分析

**常見做法：** 「這是技術性工作，讓工程師審查日誌吧。」
**為何有問題：** 工程師不具備產品直覺或領域專業知識。
**修正方式：** PM 和 QA 必須親自進行錯誤分析。這是核心的產品／品質工作。

### 錯誤 #4：未拆分資料（訓練集／開發集／測試集）

**常見做法：** 使用所有已標記資料來建立並測試評審。
**為何有問題：** 您是在對測試資料過擬合，您的指標毫無意義。
**修正方式：** 使用 15%／40%／45% 的拆分比例。在最終評估前絕對不要碰測試集。

### 錯誤 #5：上線後才開始進行評估

**常見做法：** 建立產品、上線，然後才開始思考評估。
**修正方式：** 在建立產品的同時建立評估，而非事後才做。

### 錯誤 #6：建立過多評估

**常見做法：** 「讓我們對所有事情都建立評估！」
**修正方式：** 從最大問題的 2-3 個評估開始。只有在需要時才新增。
**規則：** 若一個評估在 3 個月內從未觸發，請移除它。

### 錯誤 #7：TNR 過低（忽視假陽性）

**常見做法：** 「我的評估能抓到所有真實問題（TPR=95%），夠用了。」
**為何有問題：** 若它也頻繁誤報（TNR=22%，就像初次嘗試的天真作法），您將會忽視它。
**修正方式：** TPR 與 TNR 都必須高。TNR 低意味著評估毫無用處。

### 錯誤 #8：不測試評估本身

**常見做法：** 撰寫評估後假設它能正常運作，直接對所有資料執行。
**修正方式：** 在部署前，用已知的好案例與壞案例測試您的評估。

### 錯誤 #9：複製貼上評審提示詞

**常見做法：** 「這個 LLM 評審提示詞對別人有用，我也用吧。」
**修正方式：** 針對**您的**產品、**您的**政策、**您的**使用者撰寫評估。

### 錯誤 #10：不對系統提示詞進行版本控制

**常見做法：** 直接在生產環境編輯系統提示詞。
**修正方式：** 使用平台的提示詞管理功能（LangWatch、Langfuse 等）進行版本控制。在每筆追蹤記錄中記錄使用了哪個版本。

### 錯誤 #11：未校正評審偏差

**常見做法：** 將評審的原始通過率直接作為真實比率呈報。
**修正方式：** 使用 judgy 校正評審誤差，並呈報置信區間。

### 錯誤 #12：過早過度工程化

**常見做法：** 在審查任何一筆追蹤記錄之前，就建立一個分散式評估平台。
**修正方式：** 從簡單開始。CSV + Python 腳本 + 任何可觀測性工具。只有在簡單方法無法應付時才增加複雜度。

---

<a name="chapter-16"></a>
## 第十六章：工具與資源

### 可觀測性平台

| 工具 | 類型 | 最適用於 | 費用 |
|------|------|----------|------|
| **LangWatch** | 開源，雲端或自架 | 簡易設定、內建評估器、優秀的分析功能 | 免費方案 + 付費 |
| **Langfuse** | 開源，雲端或自架 | 自訂管線、最大彈性、龐大社群 | 免費方案 + 付費 |
| **Braintrust** | 雲端 | 出色的 UI、團隊協作 | 付費 |
| **LangSmith** | 雲端 | LangChain 使用者 | 付費 |
| **自行建立** | 自訂 | 學習、客製化需求 | 免費 |

### 評估框架

- **LangWatch Evaluators** - 40+ 種內建評估器，涵蓋安全性、品質、RAG 及自訂領域
- **Langfuse Evals** - 內建 LLM-as-Judge、透過 SDK 自訂評估器
- **Simple Evals**（OpenAI）- 輕量級模型評分評估
- **RAGAS** - 專為 RAG 評估設計
- **DeepEval** - 完整的評估框架

### 核心函式庫

- **judgy** - LLM 評審的統計偏差校正：[github.com/ai-evals-course/judgy](https://github.com/ai-evals-course/judgy)
- **rank_bm25** - BM25 檢索，用於 RAG 系統
- **LiteLLM** - 統一的 LLM API 介面

### 平台比較矩陣

| 功能 | LangWatch | Langfuse | 備注 |
|---------|-----------|----------|-------|
| **設定時間** | 5 分鐘（3 行程式碼） | 15 分鐘（更多設定） | LangWatch: langwatch.init() |
| **內建評估器** | 40+ | 0（全部自訂） | LangWatch 節省大量開發時間 |
| **自訂評估器** | 支援（裝飾器） | 支援（完整 SDK） | 兩者皆支援自訂邏輯 |
| **分析儀表板** | 內建，自動化 | 需自行建立 | LangWatch：零設定分析 |
| **費用追蹤** | 自動 | 手動標記 | LangWatch 追蹤每個模型的費用 |
| **社群規模** | 成長中 | 龐大、成熟 | Langfuse 有更多整合 |
| **自架** | Docker（簡單） | Docker（較複雜） | 兩者皆完全開源 |
| **提示詞管理** | 支援 | 支援（較成熟） | Langfuse 有更豐富的版本控制 UI |
| **快取** | 內建 | 手動 | LangWatch 自動快取重複評估 |
| **批次評估** | 原生 API | 透過實驗 | LangWatch：大批次更簡單 |
| **即時評估** | 支援 | 透過 scores API | 兩者皆可，LangWatch 設定更快 |

**選擇 LangWatch 的時機：**
- 您希望快速起步（設定 < 10 分鐘）
- 您需要常見使用案例的內建評估器
- 您希望無需設定就能獲得自動分析
- 您偏好「開箱即用」的固定意見工具

**選擇 Langfuse 的時機：**
- 您需要自訂工作流程的最大彈性
- 您有複雜的評估邏輯
- 您希望擁有最大的社群與整合生態系統
- 您偏好自行建立儀表板與分析

**為何不兩者並用？**
許多團隊同時使用兩者：LangWatch 用於快速評估與分析，Langfuse 用於深度客製化。它們是互補的，而非競爭關係。

### 核心原則（重新整理）

1. **從簡單開始** - 不要過度工程化
2. **先進行錯誤分析** - 永遠如此
3. **PM 和 QA 必須參與** - 這是產品／品質工作
4. **TPR 與 TNR 都重要** - 而非只看一致率
5. **盡可能使用程式碼評估** - 必要時才用 LLM 評審
6. **測試您的評估** - 它們也可能有 bug
7. **拆分您的資料** - 訓練集／開發集／測試集不容妥協
8. **校正偏差** - 使用 judgy 以獲得誠實的指標
9. **對提示詞進行版本控制** - 追蹤何時發生了什麼變更
10. **根據資料迭代** - 而非靠直覺

---

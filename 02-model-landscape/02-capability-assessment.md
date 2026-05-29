<a id="capability-assessment"></a>
# 能力評估

本章說明如何針對你的特定使用情境評估與比較模型能力。通用 benchmarks 很少能完整說明問題；本指南會幫助你進行更有意義的評估。

<a id="table-of-contents"></a>
## 目錄

- [為什麼 benchmarks 還不夠](#why-benchmarks-are-not-enough)
- [評估維度](#evaluation-dimensions)
- [建立自訂評估](#building-custom-evaluations)
- [常見評估陷阱](#common-evaluation-pitfalls)
- [實務評估流程](#practical-assessment-process)
- [內部 Elo 制評估](#internal-elo-based-evaluation)
- [推理校準與效率](#reasoning-calibration)
- [模型 A/B 測試](#ab-testing-models)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="why-benchmarks-are-not-enough"></a>
## 為什麼 benchmarks 還不夠

<a id="the-benchmark-problem"></a>
### Benchmark 的問題

公開 benchmarks（MMLU、HumanEval、GSM8K）有其限制：

| 問題 | 影響 |
|-------|--------|
| 訓練資料污染 | 模型可能早已看過測試題 |
| 任務不匹配 | Benchmarks 可能無法反映你的使用情境 |
| 彙總分數掩蓋差異 | 模型 A 整體勝過 B，但在你的領域可能反而輸掉 |
| 為 benchmark 最佳化 | 模型可能針對 benchmark 而非真實任務做最佳化 |
| 過時 | Benchmarks 往往落後模型能力演進 |

<a id="what-benchmarks-tell-you"></a>
### Benchmarks 能告訴你的事

```
Benchmark results tell you: "Model X scored 88% on MMLU"

What you need to know: "Will Model X correctly answer my 
customers' questions about our product documentation?"
```

**經驗法則：** 先用 benchmarks 做初步篩選，再進行自己的評估。

---

<a id="evaluation-dimensions"></a>
## 評估維度

<a id="dimension-1-task-performance-dec-2025"></a>
### 維度 1：任務表現（2025 年 12 月）

| 任務類型 | 評估方法 | 關鍵指標 |
|-----------|---------------------|------------|
| **Autonomous Coding** | CWE／SWE-bench（Verified） | 自主解決 issue 的比例 |
| **Long-Horizon Planning** | Agentic Loop 測試 | 10+ 步計畫的成功率 |
| **Reasoning Depth** | Thinking mode 分析 | CoT 各步驟間的邏輯一致性 |
| **Long Context RAG** | Needle-in-a-Haystack（2M+） | 大規模下的 recall 效率 |
| **Native Multimodal** | 交錯的 Vision／Voice／Text | 各模態之間的同步準確率 |

<a id="dimension-2-agentic-mastery"></a>
### 維度 2：Agentic 掌控力

模型使用工具與遵循多步指令的能力有多好？

```python
def evaluate_agentic_flow(agent, task_environment):
    """
    Measure success on 'Autonomous Agent' tasks:
    1. Plan generation
    2. Tool selection accuracy
    3. Error recovery
    4. Feedback loop utilization
    """
    results = []
    for scenario in task_environment.scenarios:
        traj = agent.run(scenario.goal)
        results.append({
            "success": traj.reached_goal,
            "steps": len(traj.steps),
            "tool_errors": traj.count_invalid_tool_calls()
        })
    return aggregate(results)
```

<a id="dimension-3-reasoning-reliability"></a>
### 維度 3：推理可靠性

「Thinking」模式相較標準生成，是否真的提升了輸出正確率？

| 模式 | Accuracy（Math） | Accuracy（Code） | 平均延遲 | Tokens / Output |
|------|-----------------|-----------------|-------------|-----------------|
| **Standard** | 72% | 68% | 1.2s | 400 |
| **Thinking** | 94% | 89% | 12.5s | 2400 |
| **Hybrid** | 可變 | 可變 | 使用者定義 | 可設定 |

<a id="reasoning-calibration"></a>
### 推理校準

**「過度思考」問題：**
模型經常在一個只需要 10 tokens 回答的問題上，花費 2000+ 個「thinking」tokens（例如：「2+2 等於多少？」）。

**Principal 級細節：**
請依照 **Logic Efficiency** 來評估模型：`Accuracy / (Inference Tokens)`。
2025 年的正式環境系統開始使用 **Model Arbitration**：先由小模型（Gemini 3 Flash）判斷查詢是否需要「Thinking」模式。這能避免簡單查詢承受 10 倍的延遲／成本懲罰。

---

<a id="internal-elo-based-evaluation"></a>
## 內部 Elo 制評估

**不再只依賴靜態 rubric。**
Rubrics（1-5 分量表）容易出現「評審疲勞」與「分數漂移」。到了 2025 年底，許多系統改用 **Pairwise Elo** 來維護內部 golden set 排行。

**工作流程：**
1. **Blind Side-by-Side：** 模型 A 與模型 B 對同一查詢生成答案。
2. **Judge：** 由「Ultra」模型（GPT-5.2 或人類）選出勝者。
3. **Elo Update：** 更新內部排行榜。

```python
def update_elo(winner_elo, loser_elo, k=32):
    expected_winner = 1 / (1 + 10 ** ((loser_elo - winner_elo) / 400))
    new_winner_elo = winner_elo + k * (1 - expected_winner)
    new_loser_elo = loser_elo + k * (0 - (1 - expected_winner))
    return new_winner_elo, new_loser_elo
```

**為何它有效：** 它提供的是 **相對** 排名，對評審風格變化或模型版本更替的韌性遠高於絕對分數。

<a id="dimension-4-context-recall-dec-2025"></a>
### 維度 4：Context Recall（2025 年 12 月）

在 2M+ 的 context windows 出現後，單純的「needle-in-a-haystack」已經不夠。我們現在量測的是整個視窗中的 **Contextual Reasoning**。

| 指標 | 量測方式 | 目標 |
|--------|-------------|--------|
| **Window Recall** | 在 90% 視窗深度下的事實 recall | > 98% |
| **Cross-Doc Reasoning** | 將 Doc A（位置 10k）與 Doc B（位置 1M）串接推理 | > 90% |
| **Contextual Noise Resistance** | 當視窗中 90% 都是不相關「填充」內容時的正確率 | > 95% |

---

<a id="building-custom-evaluations"></a>
## 建立自訂評估

<a id="step-1-define-evaluation-criteria"></a>
### 步驟 1：定義評估標準

```python
evaluation_criteria = {
    "correctness": {
        "weight": 0.4,
        "description": "Is the answer factually correct?",
        "scale": [1, 2, 3, 4, 5],
        "rubric": {
            5: "Completely correct, no errors",
            4: "Mostly correct, minor issues",
            3: "Partially correct, some errors",
            2: "Mostly incorrect",
            1: "Completely wrong or nonsensical"
        }
    },
    "relevance": {
        "weight": 0.3,
        "description": "Does the answer address the question?",
        "scale": [1, 2, 3, 4, 5]
    },
    "completeness": {
        "weight": 0.2,
        "description": "Are all parts of the question addressed?",
        "scale": [1, 2, 3, 4, 5]
    },
    "conciseness": {
        "weight": 0.1,
        "description": "Is the answer appropriately concise?",
        "scale": [1, 2, 3, 4, 5]
    }
}
```

<a id="step-2-create-test-set"></a>
### 步驟 2：建立測試集

```python
test_set = [
    {
        "id": "q001",
        "query": "What is the refund policy for subscription cancellation?",
        "context": "[relevant documentation]",
        "ground_truth": "Full refund within 30 days, prorated after",
        "difficulty": "easy",
        "category": "policy"
    },
    {
        "id": "q002",
        "query": "How do I integrate the API with a Python async application?",
        "context": "[API documentation]",
        "ground_truth": "[expected code pattern]",
        "difficulty": "medium",
        "category": "technical"
    },
    # ... 50-100+ test cases
]
```

**測試集指引：**
- 覆蓋所有主要使用情境
- 納入 easy、medium、hard 範例
- 在不同分類之間取得平衡
- 包含 edge cases
- 具備清楚的 ground truth 答案

<a id="step-3-implement-evaluation"></a>
### 步驟 3：實作評估

```python
class ModelEvaluator:
    def __init__(self, models: list[str], test_set: list[dict]):
        self.models = models
        self.test_set = test_set
        self.results = {}
    
    def evaluate_all(self):
        for model in self.models:
            self.results[model] = self.evaluate_model(model)
        return self.results
    
    def evaluate_model(self, model: str) -> dict:
        scores = []
        latencies = []
        
        for case in self.test_set:
            start = time.time()
            response = self.generate(model, case)
            latency = time.time() - start
            latencies.append(latency)
            
            # Score using LLM judge or human
            score = self.score_response(case, response)
            scores.append(score)
        
        return {
            "mean_score": mean(scores),
            "score_by_category": self.group_by_category(scores),
            "p50_latency": percentile(latencies, 50),
            "p99_latency": percentile(latencies, 99)
        }
    
    def score_response(self, case: dict, response: str) -> float:
        # Option 1: LLM-as-judge
        return self.llm_judge(case, response)
        
        # Option 2: Exact match
        # return exact_match(response, case["ground_truth"])
        
        # Option 3: Semantic similarity
        # return cosine_sim(embed(response), embed(case["ground_truth"]))
```

<a id="step-4-llm-as-judge"></a>
### 步驟 4：LLM-as-Judge

```python
def llm_judge(case: dict, response: str) -> dict:
    prompt = f"""Evaluate this response to a customer query.

Query: {case['query']}
Expected Answer: {case['ground_truth']}
Model Response: {response}

Rate the response on these criteria (1-5 scale):
1. Correctness: Is it factually accurate?
2. Relevance: Does it answer the question?
3. Completeness: Are all aspects covered?
4. Conciseness: Is it appropriately brief?

Output JSON:
{{"correctness": X, "relevance": X, "completeness": X, "conciseness": X, "reasoning": "..."}}
"""
    
    result = judge_model.generate(prompt)
    return parse_json(result)
```

---

<a id="common-evaluation-pitfalls"></a>
## 常見評估陷阱

<a id="pitfall-1-small-test-set"></a>
### 陷阱 1：測試集太小

**問題：** 20 個測試案例不足以做可靠比較。

**解法：** 目標至少 100+ 個案例，並依難度與類別分層取樣。

<a id="pitfall-2-ambiguous-ground-truth"></a>
### 陷阱 2：Ground Truth 含糊不清

**問題：** 「合理」答案被誤判為錯。

```
Query: "What is the capital of Australia?"
Ground truth: "Canberra"
Model answer: "The capital of Australia is Canberra."
Exact match: FAIL (but clearly correct)
```

**解法：** 使用 semantic matching 或 LLM judge，而不是 exact match。

<a id="pitfall-3-evaluation-set-leakage"></a>
### 陷阱 3：評估集外洩

**問題：** 開發與評估使用同一批案例。

**解法：** 保留一份 held-out test set，絕不拿來做 prompt tuning。

<a id="pitfall-4-ignoring-variance"></a>
### 陷阱 4：忽略變異性

**問題：** 每個測試只跑一次，忽略模型隨機性。

**解法：** 在 temperature > 0 下重複執行多次，並回報信賴區間。

<a id="pitfall-5-cost-blindness"></a>
### 陷阱 5：忽視成本

**問題：** 最佳模型可能貴上 10 倍。

**解法：** 一律回報經品質校正後的成本。

```python
def quality_adjusted_cost(model_results):
    return {
        model: {
            "quality": results["mean_score"],
            "cost_per_1k": results["cost_per_1k_queries"],
            "quality_per_dollar": results["mean_score"] / results["cost_per_1k"]
        }
        for model, results in model_results.items()
    }
```

---

<a id="practical-assessment-process"></a>
## 實務評估流程

<a id="week-1-setup-and-initial-filtering"></a>
### 第 1 週：建置與初步篩選

```
Day 1-2: Define evaluation criteria and create test set
Day 3-4: Benchmark 4-6 candidate models
Day 5: Analyze results, filter to top 2-3
```

<a id="week-2-deep-evaluation"></a>
### 第 2 週：深度評估

```
Day 1-2: Expand test set for top candidates
Day 3: Test edge cases and robustness
Day 4: Measure latency and throughput
Day 5: Calculate total cost of ownership
```

<a id="week-3-production-validation"></a>
### 第 3 週：正式環境驗證

```
Day 1-2: Shadow mode deployment
Day 3-4: A/B test if traffic allows
Day 5: Final decision and documentation
```

<a id="decision-template"></a>
### 決策範本

```markdown
## Model Evaluation Report

### Candidates Evaluated
- Model A: GPT-4o
- Model B: Claude 3.5 Sonnet
- Model C: Llama 3.1 70B

### Evaluation Results

| Metric | Model A | Model B | Model C |
|--------|---------|---------|---------|
| Overall Score | 4.2/5 | 4.3/5 | 3.9/5 |
| Category 1 | ... | ... | ... |
| P50 Latency | 450ms | 520ms | 180ms |
| Cost/1K queries | $0.85 | $1.10 | $0.25 |

### Recommendation
Model B (Claude 3.5 Sonnet) for quality-critical paths
Model C (Llama 3.1 70B) for high-volume, cost-sensitive paths

### Rationale
[Detailed reasoning]
```

---

<a id="ab-testing-models"></a>
## 模型 A/B 測試

<a id="when-to-a-b-test"></a>
### 什麼時候該做 A/B 測試

- 高流量（每天 1000+ 次查詢）
- 成功指標明確
- 可接受一定程度的品質波動風險
- 需要正式環境驗證

<a id="a-b-test-design"></a>
### A/B 測試設計

```python
class ModelABTest:
    def __init__(self, model_a: str, model_b: str, traffic_split: float = 0.5):
        self.model_a = model_a
        self.model_b = model_b
        self.traffic_split = traffic_split
        self.results = {"a": [], "b": []}
    
    def route_request(self, request_id: str) -> str:
        # Deterministic routing for consistency
        hash_val = hash(request_id) % 100
        if hash_val < self.traffic_split * 100:
            return self.model_a
        return self.model_b
    
    def record_outcome(self, request_id: str, metrics: dict):
        model = self.route_request(request_id)
        bucket = "a" if model == self.model_a else "b"
        self.results[bucket].append(metrics)
    
    def analyze(self):
        return {
            "model_a": {
                "name": self.model_a,
                "mean_score": mean([r["score"] for r in self.results["a"]]),
                "sample_size": len(self.results["a"])
            },
            "model_b": {
                "name": self.model_b,
                "mean_score": mean([r["score"] for r in self.results["b"]]),
                "sample_size": len(self.results["b"])
            },
            "p_value": self.calculate_significance()
        }
```

<a id="metrics-to-track"></a>
### 應追蹤的指標

| 指標類型 | 範例 |
|-------------|----------|
| 品質 | 使用者評分、專家審查、LLM judge |
| 互動 | 點擊率、停留時間、後續查詢 |
| 商業 | 轉換率、客服升級率、解決率 |
| 營運 | 延遲、錯誤、成本 |

---

<a id="interview-questions"></a>
## 面試題

<a id="q-how-would-you-evaluate-models-for-a-customer-support-chatbot"></a>
### Q：你會如何評估客服 chatbot 使用的模型？

**強答範例：**
我會分層進行評估：

**1. 離線評估（占 80% 心力）：**
- 從真實客服工單建立測試集（200+ 案例）
- 涵蓋所有類別：billing、technical、returns、general
- 納入 easy、medium、hard 難度
- 衡量：accuracy、helpfulness、safety

**2. 評估方法：**
- 主觀指標使用 LLM-as-judge
- 抽樣 20% 由人工審查
- 追蹤 instruction following（格式、長度）

**3. 指標：**
```python
metrics = {
    "resolution_accuracy": "Does answer solve the problem?",
    "safety": "No harmful/wrong advice?", 
    "tone": "Professional and empathetic?",
    "escalation_appropriate": "Knows when to involve human?"
}
```

**4. 正式環境驗證：**
- Shadow mode：執行新模型並比較輸出
- A/B test：將 10% 流量導向新模型
- 監控：CSAT、升級率、解決時間

<a id="q-what-is-wrong-with-using-mmlu-to-compare-models-for-your-use-case"></a>
### Q：為什麼用 MMLU 來比較你的使用情境中的模型並不理想？

**強答範例：**
MMLU 對特定使用情境有幾個問題：

**1. 領域不匹配：** MMLU 測的是學術知識；我的客服 bot 需要的是產品知識。

**2. 格式不匹配：** MMLU 是選擇題；我的使用情境需要自由生成。

**3. 污染：** 模型可能已經在訓練中看過 MMLU 題目。

**4. 彙總分數掩蓋差異：** 模型 A 在 MMLU 勝過 B，但在我重視的類別可能表現更差。

**5. 沒有測試 context：** MMLU 不會測 RAG 或長上下文能力。

**更好的做法：** 
- 用 MMLU 做初步篩選（節省時間）
- 用自訂評估做最終決策
- 用真實使用情境資料測試
- 納入營運指標（延遲、成本）

---

<a id="references"></a>
## 參考資料

- Zheng et al. "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena" (2023)
- LMSYS Chatbot Arena: https://chat.lmsys.org/
- HELM: https://crfm.stanford.edu/helm/
- LMSys Evaluation: https://github.com/lm-sys/FastChat/tree/main/fastchat/llm_judge
- OpenAI Evals: https://github.com/openai/evals

---

*上一篇：[Model Taxonomy](01-model-taxonomy.md) | 下一篇：[Pricing and Costs](03-pricing-and-costs.md)*

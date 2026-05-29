<a id="ci-cd-for-llm-applications"></a>
# LLM 應用程式的 CI/CD

部署 LLM 應用程式時，必須針對模型評估、prompt 測試與品質閘門等 AI 專屬議題，調整傳統的 CI/CD 實務。

<a id="table-of-contents"></a>
## 目錄

- [LLM CI/CD 挑戰](#llm-cicd-challenges)
- [Pipeline 架構](#pipeline-architecture)
- [測試階段](#testing-stages)
- [品質閘門](#quality-gates)
- [部署策略](#deployment-strategies)
- [回滾程序](#rollback-procedures)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="llm-cicd-challenges"></a>
## LLM CI/CD 挑戰

<a id="what-makes-llm-deployments-different"></a>
### LLM 部署有何不同

| 傳統 CI/CD | LLM CI/CD |
|-------------------|-----------|
| 二元測試（通過/失敗） | 機率式評估 |
| 快速測試 | 緩慢且昂貴的評估 |
| 決定性輸出 | 非決定性輸出 |
| 只有程式碼變更 | Prompt + 模型 + 資料變更 |
| 版本控制很直觀 | Prompt 版本管理較複雜 |

<a id="change-types"></a>
### 變更類型

| 變更類型 | 風險 | 所需測試 |
|-------------|------|------------------|
| Prompt 文字 | 中 | 回歸測試 + 品質評估 |
| System prompt | 高 | 完整評估套件 |
| 模型版本 | 高 | 全面 benchmark |
| RAG index | 中 | Retrieval + 品質評估 |
| 參數（temp 等） | 低-中 | 品質抽樣 |

---

<a id="pipeline-architecture"></a>
## Pipeline 架構

<a id="full-pipeline"></a>
### 完整 Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                       LLM CI/CD PIPELINE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐                                               │
│  │   Commit     │                                               │
│  │   Trigger    │                                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │   Validate   │ ─── Prompt syntax, config validation         │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │ Unit Tests   │ ─── Fast, deterministic tests                │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │  Golden Set  │ ─── Known input/output pairs                 │
│  │    Tests     │                                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │   LLM Eval   │ ─── Quality scoring, regression detection    │
│  │   (Sampled)  │                                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │ Quality Gate │ ─── Pass/fail based on thresholds            │
│  └──────┬───────┘                                               │
│         │                                                        │
│    ┌────┴────┐                                                  │
│    ▼         ▼                                                  │
│ ┌──────┐ ┌───────┐                                             │
│ │Canary│ │Blocked│                                             │
│ │Deploy│ │       │                                             │
│ └──┬───┘ └───────┘                                             │
│    │                                                            │
│    ▼                                                            │
│ ┌──────────────┐                                               │
│ │  Production  │                                               │
│ │  Monitoring  │                                               │
│ └──────────────┘                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

<a id="testing-stages"></a>
## 測試階段

<a id="stage-1-static-validation"></a>
### 第 1 階段：靜態驗證

```python
class PromptValidator:
    def validate(self, prompt_config: dict) -> ValidationResult:
        errors = []
        
        # Required fields
        if not prompt_config.get("system_prompt"):
            errors.append("Missing system_prompt")
        
        # Template syntax
        try:
            Template(prompt_config["user_template"]).substitute({})
        except KeyError:
            pass  # Expected for templates with variables
        except ValueError as e:
            errors.append(f"Invalid template syntax: {e}")
        
        # Token limits
        system_tokens = count_tokens(prompt_config.get("system_prompt", ""))
        if system_tokens > 4000:
            errors.append(f"System prompt too long: {system_tokens} tokens")
        
        return ValidationResult(
            valid=len(errors) == 0,
            errors=errors
        )
```

<a id="stage-2-unit-tests"></a>
### 第 2 階段：單元測試

```python
class PromptUnitTests:
    def test_template_rendering(self):
        prompt = PromptTemplate(SYSTEM_PROMPT, USER_TEMPLATE)
        
        rendered = prompt.render(
            query="test query",
            context="test context"
        )
        
        assert "test query" in rendered
        assert "test context" in rendered
        assert len(rendered) < 10000  # Token limit
    
    def test_output_parsing(self):
        parser = OutputParser()
        
        valid_output = '{"answer": "test", "confidence": 0.9}'
        result = parser.parse(valid_output)
        assert result["answer"] == "test"
        
        invalid_output = "not json"
        with pytest.raises(ParseError):
            parser.parse(invalid_output)
```

<a id="stage-3-golden-set-tests"></a>
### 第 3 階段：Golden Set 測試

```python
class GoldenSetRunner:
    def __init__(self, golden_set: list[dict]):
        self.golden_set = golden_set
    
    async def run(self, llm_client) -> TestResults:
        results = []
        
        for example in self.golden_set:
            response = await llm_client.generate(example["input"])
            
            # Exact match for deterministic outputs
            if example.get("exact_match"):
                passed = response == example["expected"]
            # Contains check for flexible outputs
            elif example.get("must_contain"):
                passed = all(
                    phrase in response 
                    for phrase in example["must_contain"]
                )
            # LLM judge for quality
            else:
                passed = await self.judge_quality(
                    response, example["expected"]
                )
            
            results.append(TestResult(
                input=example["input"],
                expected=example["expected"],
                actual=response,
                passed=passed
            ))
        
        return TestResults(
            total=len(results),
            passed=sum(1 for r in results if r.passed),
            failed=[r for r in results if not r.passed]
        )
```

<a id="stage-4-llm-evaluation"></a>
### 第 4 階段：LLM 評估

```python
class LLMEvaluationStage:
    def __init__(self, eval_set: list[dict], sample_rate: float = 0.1):
        self.eval_set = eval_set
        self.sample_rate = sample_rate
        self.evaluator = LLMEvaluator()
    
    async def run(self, llm_client) -> EvalResults:
        # Sample for cost efficiency
        sample = random.sample(
            self.eval_set,
            int(len(self.eval_set) * self.sample_rate)
        )
        
        scores = []
        for example in sample:
            response = await llm_client.generate(example["input"])
            
            score = await self.evaluator.evaluate(
                query=example["input"],
                response=response,
                reference=example.get("reference"),
                criteria=["relevance", "accuracy", "helpfulness"]
            )
            scores.append(score)
        
        return EvalResults(
            sample_size=len(sample),
            avg_relevance=np.mean([s["relevance"] for s in scores]),
            avg_accuracy=np.mean([s["accuracy"] for s in scores]),
            avg_helpfulness=np.mean([s["helpfulness"] for s in scores])
        )
```

---

<a id="quality-gates"></a>
## 品質閘門

<a id="gate-configuration"></a>
### 閘門設定

```python
class QualityGate:
    def __init__(self, thresholds: dict):
        self.thresholds = thresholds
    
    def evaluate(self, results: dict) -> GateResult:
        failures = []
        
        # Golden set pass rate
        if results["golden_pass_rate"] < self.thresholds["golden_pass_rate"]:
            failures.append({
                "metric": "golden_pass_rate",
                "actual": results["golden_pass_rate"],
                "threshold": self.thresholds["golden_pass_rate"]
            })
        
        # Quality scores
        for metric in ["relevance", "accuracy", "helpfulness"]:
            if results.get(f"avg_{metric}", 0) < self.thresholds.get(metric, 0):
                failures.append({
                    "metric": metric,
                    "actual": results.get(f"avg_{metric}"),
                    "threshold": self.thresholds[metric]
                })
        
        # Regression detection
        if results.get("regression_detected"):
            failures.append({
                "metric": "regression",
                "details": results["regression_details"]
            })
        
        return GateResult(
            passed=len(failures) == 0,
            failures=failures
        )

# Example thresholds
QUALITY_THRESHOLDS = {
    "golden_pass_rate": 0.95,  # 95% of golden tests must pass
    "relevance": 4.0,          # Average score >= 4.0/5.0
    "accuracy": 4.0,
    "helpfulness": 3.5
}
```

---

<a id="deployment-strategies"></a>
## 部署策略

<a id="canary-deployment"></a>
### Canary 部署

```python
class CanaryDeployer:
    def __init__(
        self,
        initial_percentage: int = 5,
        increment: int = 10,
        bake_time_minutes: int = 30
    ):
        self.initial_percentage = initial_percentage
        self.increment = increment
        self.bake_time = bake_time_minutes
    
    async def deploy(self, new_version: str):
        # Start canary
        await self.router.set_canary(new_version, self.initial_percentage)
        
        percentage = self.initial_percentage
        while percentage < 100:
            # Wait for bake time
            await asyncio.sleep(self.bake_time * 60)
            
            # Check canary health
            metrics = await self.get_canary_metrics(new_version)
            
            if not self.is_healthy(metrics):
                await self.rollback(new_version)
                raise CanaryFailedError(metrics)
            
            # Increment traffic
            percentage = min(100, percentage + self.increment)
            await self.router.set_canary(new_version, percentage)
        
        # Full rollout
        await self.router.promote_canary(new_version)
```

<a id="shadow-deployment"></a>
### Shadow 部署

```python
class ShadowDeployer:
    async def shadow_test(
        self,
        new_version: str,
        duration_hours: int = 24
    ):
        # Run new version in shadow mode
        await self.enable_shadow(new_version)
        
        # Collect comparison data
        start = datetime.now()
        while datetime.now() - start < timedelta(hours=duration_hours):
            await asyncio.sleep(60)
            
            comparison = await self.compare_outputs()
            if comparison["divergence_rate"] > 0.1:
                await self.alert("High divergence in shadow test", comparison)
        
        # Analyze results
        return await self.generate_comparison_report(new_version)
```

---

<a id="rollback-procedures"></a>
## 回滾程序

<a id="automated-rollback"></a>
### 自動回滾

```python
class AutoRollback:
    def __init__(self, rollback_thresholds: dict):
        self.thresholds = rollback_thresholds
    
    async def monitor_and_rollback(self, version: str):
        while True:
            metrics = await self.get_live_metrics(version)
            
            # Check error rate
            if metrics["error_rate"] > self.thresholds["error_rate"]:
                await self.trigger_rollback(version, "error_rate_exceeded")
                return
            
            # Check latency
            if metrics["p99_latency"] > self.thresholds["p99_latency"]:
                await self.trigger_rollback(version, "latency_exceeded")
                return
            
            # Check quality (sampled)
            if metrics.get("quality_score", 5) < self.thresholds["quality_score"]:
                await self.trigger_rollback(version, "quality_degradation")
                return
            
            await asyncio.sleep(60)
    
    async def trigger_rollback(self, version: str, reason: str):
        previous = await self.get_previous_version()
        await self.router.rollback_to(previous)
        await self.alert(f"Auto-rollback from {version}: {reason}")
```

---

<a id="interview-questions"></a>
## 面試題

<a id="q-how-do-you-test-prompt-changes-before-production"></a>
### Q：在正式上線前，你如何測試 prompt 變更？

**強回答：**

「我會使用多階段測試 pipeline：

**第 1 階段：靜態驗證。** 檢查語法、token 限制與 template 錯誤。快速又便宜。

**第 2 階段：單元測試。** 驗證 template render、輸出解析與決定性行為。仍然很快。

**第 3 階段：Golden set 測試。** 使用必須通過的已知輸入／輸出配對。可抓出明顯的回歸。

**第 4 階段：LLM 評估。** 使用 LLM-as-judge 進行抽樣評估。衡量品質維度（relevance、accuracy）。成本較高，但能抓出細微問題。

**品質閘門：** 所有階段都必須達到門檻。Golden set 通過率 > 95%，品質分數 > 4.0/5.0。

**部署：** 先以 5% 流量做 Canary，烘焙 30 分鐘，監控指標，再逐步提高。

關鍵洞見是 LLM 輸出具有非決定性，因此測試必須採統計方法。我無法保證 100% 正確，但可以確保品質維持在可接受範圍內。」

<a id="q-what-triggers-should-cause-automatic-rollback"></a>
### Q：哪些觸發條件應該導致自動回滾？

**強回答：**

「我會設定多種回滾觸發條件：

**錯誤率：** 若錯誤率連續 5 分鐘超過 5%，就回滾。這能抓到明顯故障。

**延遲：** 若 P99 延遲連續 10 分鐘超過 SLA（例如 10 秒），就回滾。這能抓到效能回歸。

**品質分數：** 若抽樣品質分數低於 3.5/5.0，就回滾。這能抓到細微的品質下降。

**使用者訊號：** 若負面回饋率飆升到基準的 2 倍，就進一步調查並視情況回滾。

**實作：**
- 由 Prometheus 警示觸發回滾腳本
- 自動通知團隊
- 回滾到最近一個已知穩定版本
- 在完成調查前阻止後續部署

關鍵在於快速偵測與行動。生產環境中壞掉的 prompt 持續 10 分鐘還可接受；持續 10 小時就不行。」

---

<a id="references"></a>
## 參考資料

- ML Ops: https://ml-ops.org/
- LangSmith: https://docs.smith.langchain.com/

---

*上一章：[LLM 基礎設施](01-llm-infrastructure.md)*

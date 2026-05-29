<a id="ensemble-methods-for-llm-reliability"></a>
# 提升 LLM 可靠性的集成方法

集成方法對正式環境的可靠性至關重要。本章涵蓋可提升準確度並降低幻覺的多模型協作模式。

<a id="table-of-contents"></a>
## 目錄

- [為何集成很重要](#why-ensembles-matter)
- [評估型集成](#evaluation-ensembles)
- [生成型集成](#generation-ensembles)
- [多代理模式](#multi-agent-patterns)
- [集成與仲裁的差異](#ensemble-vs-arbitration)
- [成本與準確度的權衡](#cost-accuracy-tradeoffs)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="why-ensembles-matter"></a>
## 為何集成很重要

單一模型的輸出對高風險應用而言並不可靠：
- 模型會產生幻覺
- 推理可能有缺陷
- 輸出會隨 temperature 波動
- 單一 judge 的評估會有偏誤

集成透過冗餘與多樣性來提升可靠性。

<a id="ensemble-methods-taxonomy"></a>
### 集成方法分類

| 類別 | 目的 | 方法 |
|------|------|------|
| 評估 | 降低 judge 偏誤 | Panel of Judges、Pairwise Comparison |
| 生成 | 提升輸出品質 | Self-Consistency、Best-of-N |
| 驗證 | 降低幻覺 | Multi-Agent Debate、Fact Checking |
| 綜合 | 結合不同觀點 | Mixture of Agents |

---

<a id="evaluation-ensembles"></a>
## 評估型集成

<a id="panel-of-llm-judges-poll"></a>
### LLM Judges Panel（PoLL）

讓多個具多樣性的模型對同一輸出打分：

```python
class PanelOfJudges:
    """
    Production implementation of PoLL pattern.
    Key insight: Diversity of judges matters more than individual judge quality.
    """
    def __init__(self, judges: list, aggregation: str = "mean"):
        # Use diverse model families, not just different sizes
        # Good: [Claude, GPT-4, Gemini, Llama-70B]
        # Bad: [GPT-4, GPT-4-turbo, GPT-3.5] - same family bias
        self.judges = judges
        self.aggregation = aggregation
    
    async def evaluate(self, question: str, answer: str, rubric: str) -> dict:
        # Parallel evaluation for latency
        judgments = await asyncio.gather(*[
            judge.score(question, answer, rubric) 
            for judge in self.judges
        ])
        
        scores = [j["score"] for j in judgments]
        
        # Track inter-judge agreement for confidence
        agreement = 1 - (np.std(scores) / max(np.mean(scores), 0.01))
        
        if self.aggregation == "mean":
            final_score = np.mean(scores)
        elif self.aggregation == "median":  # More robust to outliers
            final_score = np.median(scores)
        elif self.aggregation == "trimmed_mean":  # Drop highest and lowest
            final_score = np.mean(sorted(scores)[1:-1])
        
        return {
            "score": final_score,
            "confidence": agreement,
            "individual_scores": scores,
            "needs_review": agreement < 0.7  # Flag for human review
        }
```

**適用時機：** 高風險評估、benchmark 建立、無法接受單一 judge 偏誤時。

<a id="pairwise-comparison-with-positional-debiasing"></a>
### 具位置去偏誤的成對比較

模型約有 60-70% 的機率偏好第一個選項。務必同時測試兩種順序：

```python
async def pairwise_compare_debiased(model, response_a: str, response_b: str, criteria: str) -> dict:
    """
    Critical: Models have significant positional bias.
    Always run both orderings and aggregate.
    """
    # Run both orderings in parallel
    result_ab, result_ba = await asyncio.gather(
        model.compare(first=response_a, second=response_b, criteria=criteria),
        model.compare(first=response_b, second=response_a, criteria=criteria)
    )
    
    # If A wins in both positions -> Strong signal for A
    if result_ab["winner"] == "first" and result_ba["winner"] == "second":
        return {"winner": "A", "confidence": "high"}
    
    # If B wins in both positions -> Strong signal for B
    elif result_ab["winner"] == "second" and result_ba["winner"] == "first":
        return {"winner": "B", "confidence": "high"}
    
    # Winner depends on position -> Positional bias detected
    else:
        return {
            "winner": "tie",
            "confidence": "low",
            "note": "Positional bias detected"
        }
```

---

<a id="generation-ensembles"></a>
## 生成型集成

<a id="self-consistency-majority-voting"></a>
### Self-Consistency（多數決）

產生多條推理路徑，並對最終答案投票：

```python
class SelfConsistencyDecoder:
    """
    Key parameters:
    - k (sample count): 5-10 for most tasks, 15-20 for hard math
    - temperature: 0.5-0.8 for reasoning tasks
    
    Too low temperature = not enough diversity
    Too high temperature = too much noise
    """
    
    def __init__(self, model, k: int = 7, temperature: float = 0.7):
        self.model = model
        self.k = k
        self.temperature = temperature
    
    async def generate_with_consistency(self, prompt: str) -> dict:
        # Generate k reasoning paths in parallel
        responses = await asyncio.gather(*[
            self.model.generate(prompt, temperature=self.temperature)
            for _ in range(self.k)
        ])
        
        # Extract final answers (task-specific)
        answers = [self.extract_answer(r) for r in responses]
        
        # Majority voting
        answer_counts = Counter(answers)
        majority_answer, majority_count = answer_counts.most_common(1)[0]
        
        # Confidence = proportion of votes for winner
        confidence = majority_count / self.k
        
        # Get best reasoning path that led to majority answer
        best_reasoning = self.select_best_reasoning(
            responses, answers, majority_answer
        )
        
        return {
            "answer": majority_answer,
            "confidence": confidence,
            "num_paths": self.k,
            "reasoning": best_reasoning,
            "vote_distribution": dict(answer_counts)
        }
    
    def extract_answer(self, response: str) -> str:
        # Task-specific answer extraction
        # For math: extract the final number
        # For code: extract the function
        # Implement based on your task
        pass
```

**最適合：** 數學、邏輯、具有可驗證答案的程式撰寫。準確度提升：5-15%。

<a id="best-of-n-with-reward-model"></a>
### 搭配 Reward Model 的 Best-of-N

產生 N 個候選答案，用 reward model 評分後回傳最佳者：

```python
class BestOfNSampler:
    """
    Key considerations:
    1. N selection: N=4-8 for interactive, N=16-64 for batch
    2. Reward model ensemble prevents reward hacking
    3. Monitor sample diversity - if too similar, BoN is wasted compute
    """
    
    def __init__(self, generator, reward_models: list, n: int = 8):
        self.generator = generator
        self.reward_models = reward_models  # Ensemble for robustness
        self.n = n
    
    async def generate_best(self, prompt: str) -> dict:
        # Generate N candidates in parallel
        candidates = await asyncio.gather(*[
            self.generator.generate(prompt, temperature=0.8)
            for _ in range(self.n)
        ])
        
        # Score with reward model ensemble
        scored_candidates = []
        for candidate in candidates:
            rm_scores = await asyncio.gather(*[
                rm.score(prompt, candidate) for rm in self.reward_models
            ])
            
            # Conservative aggregation prevents reward hacking
            # Use 25th percentile instead of mean
            conservative_score = np.percentile(rm_scores, 25)
            
            scored_candidates.append({
                "response": candidate,
                "score": conservative_score,
                "rm_agreement": 1 - np.std(rm_scores) / np.mean(rm_scores)
            })
        
        # Select best by conservative score
        best = max(scored_candidates, key=lambda x: x["score"])
        
        # Compute diversity metric
        diversity = self.compute_diversity(candidates)
        
        return {
            "response": best["response"],
            "score": best["score"],
            "n_sampled": self.n,
            "diversity_score": diversity,
            "low_diversity_warning": diversity < 0.3
        }
    
    def compute_diversity(self, candidates: list) -> float:
        # Embed candidates and compute average pairwise distance
        embeddings = [embed(c) for c in candidates]
        similarities = []
        for i in range(len(embeddings)):
            for j in range(i + 1, len(embeddings)):
                similarities.append(cosine_similarity(embeddings[i], embeddings[j]))
        return 1 - np.mean(similarities)  # Higher = more diverse
```

**最適合：** 開放式生成、創意任務。準確度提升：10-30%。

---

<a id="multi-agent-patterns"></a>
## 多代理模式

<a id="multi-agent-debate"></a>
### Multi-Agent Debate

讓多個模型反覆互相評論：

```python
class MultiAgentDebate:
    """
    Pattern: Multiple models debate to reduce hallucinations.
    
    Most effective when:
    1. Models have different biases (diverse model families)
    2. 2-3 rounds is optimal (more = diminishing returns)
    3. Explicit "devil's advocate" prompting improves results
    """
    
    def __init__(self, debaters: list, rounds: int = 2):
        self.debaters = debaters
        self.rounds = rounds
    
    async def debate(self, question: str) -> dict:
        # Round 0: Initial positions
        positions = await asyncio.gather(*[
            debater.generate(f"Answer this question with reasoning: {question}")
            for debater in self.debaters
        ])
        
        debate_history = [{"round": 0, "positions": positions}]
        
        # Debate rounds
        for round_num in range(1, self.rounds + 1):
            new_positions = []
            
            for i, debater in enumerate(self.debaters):
                other_positions = [p for j, p in enumerate(positions) if j != i]
                
                critique_prompt = f"""
Question: {question}

Your previous answer: {positions[i]}

Other perspectives:
{self.format_positions(other_positions)}

Consider the other perspectives. If they raise valid points, update your answer.
If you still disagree, explain why with specific reasoning.
Provide your final answer.
"""
                new_position = await debater.generate(critique_prompt)
                new_positions.append(new_position)
            
            positions = new_positions
            debate_history.append({"round": round_num, "positions": positions})
        
        # Final synthesis
        final_answer = await self.synthesize(question, debate_history)
        
        return {
            "answer": final_answer,
            "rounds": self.rounds,
            "consensus_reached": self.check_consensus(positions),
            "debate_history": debate_history
        }
```

**最適合：** 事實驗證、降低複雜回答中的幻覺。

<a id="mixture-of-agents-moa"></a>
### Mixture of Agents（MoA）

多個模型先輸出，再由聚合器分層整合：

```
┌─────────────────────────────────────────────────────────────────┐
│                    MIXTURE OF AGENTS (MoA)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Layer 1 (Proposers):                                           │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐            │
│  │ Claude  │  │  GPT-4  │  │ Gemini  │  │ Llama   │            │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘            │
│       │            │            │            │                   │
│       └────────────┴─────┬──────┴────────────┘                  │
│                          │                                       │
│  Layer 2 (Aggregator):   ▼                                      │
│  ┌──────────────────────────────────────────────────┐           │
│  │  "Given these perspectives: [R1, R2, R3, R4]    │           │
│  │   Synthesize the best answer..."                │           │
│  └────────────────────────┬─────────────────────────┘           │
│                           │                                      │
│                           ▼                                      │
│                    [Final Output]                                │
└─────────────────────────────────────────────────────────────────┘
```

```python
class MixtureOfAgents:
    def __init__(self, proposers: list, aggregator):
        self.proposers = proposers
        self.aggregator = aggregator
    
    async def generate(self, prompt: str) -> str:
        # Layer 1: Get diverse proposals
        proposals = await asyncio.gather(*[
            proposer.generate(prompt) for proposer in self.proposers
        ])
        
        # Layer 2: Aggregate
        aggregation_prompt = f"""
Given the following question and multiple expert responses, 
synthesize the best possible answer.

Question: {prompt}

Expert responses:
{self.format_proposals(proposals)}

Synthesize the best answer, combining the strongest elements from each response.
"""
        
        final_answer = await self.aggregator.generate(aggregation_prompt)
        return final_answer
```

**最適合：** 複雜綜合、報告生成、多領域問題。

---

<a id="ensemble-vs-arbitration"></a>
## 集成與仲裁的差異

<a id="conceptual-distinction"></a>
### 概念上的區別

| 面向 | 集成學習 | 模型仲裁 |
|------|----------|----------|
| **目標** | 結合所有輸出 | 選出單一最佳輸出 |
| **機制** | 聚合（投票、平均） | 選擇（評分、排序） |
| **關係** | 協作式 | 競爭式 |
| **最終輸出** | 由所有模型合成的結果 | 單一勝出模型的輸出 |
| **適用時機** | 想要穩健性、降低變異 | 想要最佳品質 |

<a id="decision-framework"></a>
### 決策框架

```
Is there a single "correct" answer format?
├── Yes (classification, math)
│   └── Use Ensemble (voting/averaging)
│
└── No (creative writing, open QA)
    └── Use Arbitration (best-of-N)
        └── Do you have reliable scoring?
            ├── Yes → Reward model selection
            └── No → LLM-as-judge or human
```

---

<a id="cost-accuracy-tradeoffs"></a>
## 成本與準確度的權衡

<a id="ensemble-cost-matrix"></a>
### 集成成本矩陣

| 方法 | 成本倍數 | 延遲 | 準確度提升 | 適用時機 |
|------|----------|------|------------|----------|
| 單一模型 | 1x | 1x | 基準線 | 低風險、高流量 |
| Self-Consistency k=3 | 3x | 1x（平行） | +5-8% | 推理、延遲敏感 |
| Self-Consistency k=10 | 10x | 1x（平行） | +10-15% | 數學、準確度關鍵 |
| Best-of-N（N=8） | 8x + scoring | 1x（平行） | +15-25% | 創意生成 |
| Panel of Judges（3） | 3x eval | 1x（平行） | 降低偏誤 | 評估任務 |
| Multi-Agent Debate | 6x | 3x | 幻覺 ↓ | 事實關鍵 |
| Mixture of Agents | 5-8x | 2x | 更佳綜合能力 | 複雜報告 |

<a id="when-not-to-use-ensembles"></a>
### 何時不要使用集成

| 情境 | 為什麼不適合 | 替代方案 |
|------|--------------|----------|
| 簡單事實查詢 | 沒有多樣性收益 | 單次 RAG 呼叫 |
| 需要 < 500ms 延遲 | 集成會增加延遲 | 單一模型 + 快取 |
| 成本是首要限制 | 集成會放大成本 | 模型蒸餾 |
| 模型高度相關 | 沒有多樣性就沒有收益 | 先取得多樣化模型 |

---

<a id="interview-questions"></a>
## 面試題

<a id="q-when-would-you-use-self-consistency-vs-best-of-n"></a>
### 問：你會在什麼情況下使用 Self-Consistency 而不是 Best-of-N？

**強回答：**

「兩者用途不同：

**Self-Consistency** 適合有可抽取、可驗證答案的任務：
- 數學題：抽出最終數字，用多數決
- 分類：對標籤投票
- 短答 QA：對答案投票

關鍵在於你能比較答案是否相等。Temperature 0.5-0.8 可以在維持連貫性的同時提供多樣性。大多數任務我會用 k=5-10。

**Best-of-N** 適合沒有單一正確答案的開放式生成：
- 創意寫作
- 解釋說明
- 可以有多種寫法的程式碼

這時我需要 reward model 或 judge 來對候選答案評分，因為我不能只看答案是否相等。N 通常設為 8-16。挑戰在於避免 reward hacking，因此我會使用 reward model 集成加上保守聚合。

我不會把 Self-Consistency 用在創意寫作（沒有可抽取答案），也不會把 Best-of-N 用在數學題（直接投票更簡單）。」

<a id="q-how-do-you-prevent-reward-hacking-in-best-of-n"></a>
### 問：你如何在 Best-of-N 中避免 reward hacking？

**強回答：**

「Reward hacking 是指模型利用 reward model 的弱點，而不是真正提升品質。

**我的緩解方式：**

1. **Reward model 集成**：使用 3 個以上、多樣化的 reward model。能騙過一個 RM 的樣本，不太可能同時騙過全部。

2. **保守聚合**：不要用平均分，而改用第 25 百分位數或最小值。這會選出在所有 RM 上都表現不錯的樣本，而不是只在單一 RM 上分數高。

3. **多樣性監控**：追蹤樣本多樣性。如果多樣性掉太低，模型可能正在利用狹窄的 reward hack。我會調整 temperature 或改用不同 prompt。

4. **人工校準**：定期驗證 RM 選出的樣本是否真的符合人類偏好。

5. **多維度評分**：從多個面向評分（品質、安全、相關性），要求各面向都達標，而不是只看單一綜合分數。

關鍵洞見是：任何單一 reward signal 都可能被操弄。集成會讓操弄難度大幅提高。」

---

<a id="references"></a>
## 參考資料

- Verga et al. "Replacing Judges with Juries: Evaluating LLM Generations with a Panel of Diverse Models" (2024)
- Wang et al. "Self-Consistency Improves Chain of Thought Reasoning" (2023)
- Du et al. "Improving Factuality and Reasoning in Language Models through Multiagent Debate" (2023)

---

*下一章：[延伸可靠性模式](02-reliability-patterns.md)*

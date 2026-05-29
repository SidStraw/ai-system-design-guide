<a id="model-selection-guide"></a>
# 模型選型指南

本章提供一個實務框架，協助你根據能力、成本、延遲與營運因素，為自己的使用情境挑選合適的 LLM。

<a id="table-of-contents"></a>
## 目錄

- [選型框架](#selection-framework)
- [能力比較](#capability-comparison)
- [使用情境對應](#use-case-mapping)
- [成本分析](#cost-analysis)
- [營運考量](#operational-considerations)
- [多模型策略](#multi-model-strategies)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="selection-framework"></a>
## 選型框架

<a id="decision-tree-dec-2025"></a>
### 決策樹（2025 年 12 月）

```
Start Here
    │
    ├── Need autonomous agents / long-horizon planning?
    │   └── Yes ─────────────────────────────────────────┐
    │   └── No ──┐                                       │
    │            │                                       ▼
    │            │                              ┌─────────────────┐
    │            │                              │ GPT-5.5 / Claude│
    │            │                              │ 4.5 Opus / o3   │
    │            │                              └─────────────────┘
    │            │
    ├── Need best software engineering / coding?
    │   └── Yes ─────────────────────────────────────────┐
    │   └── No ──┐                                       │
    │            │                                       ▼
    │            │                              ┌─────────────────┐
    │            │                              │ Claude Opus 4.7 │
    │            │                              │ Claude Sonnet 4.6│
    │            │                              └─────────────────┘
    │            │
    ├── Need to process massive context (>1M)?
    │   └── Yes ─────────────────────────────────────────┐
    │   └── No ──┐                                       │
    │            │                                       ▼
    │            │                              ┌─────────────────┐
    │            │                              │ Gemini 3.0 Pro  │
    │            │                              │ (2.5M context)  │
    │            │                              └─────────────────┘
    │            │
    ├── Cost-sensitive high volume?
    │   └── Yes ─────────────────────────────────────────┐
    │   └── No ──┐                                       │
    │            │                                       ▼
    │            │                              ┌─────────────────┐
    │            │                              │ Gemini 3 Flash /│
    │            │                              │ o4-mini         │
    │            │                              └─────────────────┘
    │            │
    └── Default: Production Choice
                 ▼
        ┌─────────────────┐
        │ Claude Sonnet 4.6│
        │ GPT-5.5-mini    │
        └─────────────────┘
```

<a id="key-selection-factors"></a>
### 關鍵選型因素

| 因素 | 權重 | 考量 |
|--------|--------|----------------|
| **Agentic Reliability** | 高 | Tool-calling 準確率、多步規劃能力 |
| **Context Recall** | 高 | 1M+ context 下 needle-in-a-haystack 表現 |
| **Rate Limit Ceiling** | 高 | **（Principal 細節）**：供應商能否承受你的 P99 吞吐而不發生 429？ |
| **Ecosystem Maturity** | 高 | 正式環境歷史、SDK 支援與 Enterprise SLA |
| **Cost / Output Token** | 中 | Agentic loops 會消耗 5x-10x 更多 tokens |

---

<a id="capability-comparison"></a>
## 能力比較

<a id="frontier-model-comparison-december-2025"></a>
### 前沿模型比較（2025 年 12 月）

| 模型 | 優勢 | 缺點 | Context | 最適合 |
|-------|-----------|------|---------|----------|
| **GPT-5.5** | Agentic 規劃、原生 omni | 成本高 | 512K | 多 agent 系統 |
| **Claude Opus 4.7** | 最先進 Software Engineering | 昂貴 | 400K | 複雜 codebases |
| **Claude Sonnet 4.6** | Hybrid reasoning 深度 | 尖峰延遲較高 | 200K | 通用正式環境 |
| **Gemini 3.0 Pro** | 2.5M context、多模態 | 有延遲尖峰 | 2.5M | 大型資料攝取 |
| **o3** | 極致邏輯／推理 | 成本／延遲高 | 128K | 數學、複雜除錯 |

<a id="budget-model-comparison"></a>
### 預算型模型比較

| 模型 | 成本（每 1M input/output） | 品質 | Context | 最適合 |
|-------|----------------------------|---------|---------|----------|
| **Gemini 3 Flash** | $0.05 / $0.20 | Frontier-tier | 1M | 高流量 RAG |
| **o4-mini** | $0.10 / $0.40 | Excellent | 128K | 快速推理任務 |
| **Llama 4 8B** | Self-hosted（H100/L40） | Strong | 128K | 裝置端、私有場景 |

<a id="open-source-models"></a>
### 開源模型

| 模型 | 參數 | 品質 | 最適合 |
|-------|------------|---------|----------|
| **Llama 4 70B** | 70B | Frontier-competitive | 通用開放首選 |
| **Nemotron 3 Ultra** | 500B MoE | Agentic mastery | 可擴展的開放 agents |
| **DeepSeek V3.2** | 671B MoE | Ultra performance | 以最低 TCO 達成 frontier 品質 |

---

<a id="use-case-mapping"></a>
## 使用情境對應

<a id="by-application-type-dec-2025"></a>
### 依應用類型（2025 年 12 月）

| 使用情境 | 建議模型 | 理由 |
|----------|-------------------|-----------|
| **Autonomous Dev** | Claude Opus 4.7, Claude Sonnet 4.6 | 「Claude Code」的 agentic 掌控力與已驗證 coding 能力 |
| **Enterprise RAG** | Gemini 3.0 Pro, Gemini 3 Flash | 2.5M context 可降低 retrieval 複雜度 |
| **Customer Support** | Gemini 3 Flash, GPT-5.5-mini | 近乎零延遲且推理能力強 |
| **Reasoning / Debug** | o3, DeepSeek-R1 | 在 code／logic 的「Thinking」模式中表現最佳 |
| **Video/Multimodal** | Gemini 3.0 Pro, GPT-5.5 | 原生支援交錯式多模態處理 |
| **Private Agent** | Llama 4 70B, Nemotron 3 | 最強的 open-weight agentic planning |

<a id="by-constraint"></a>
### 依限制條件

| 限制 | 方法 |
|------------|----------|
| **最大延遲 < 100ms** | Gemini 3 Flash、o4-mini，或 self-hosted Nano models |
| **Context > 1M tokens** | Gemini 3.0 Pro（原生 2.5M） |
| **零資料外洩** | 在內部 VPC 上部署 Llama 4 70B |
| **複雜工具使用** | Claude Opus 4.7 或 GPT-5.5（規劃準確率最佳） |

---

<a id="cost-analysis"></a>
## 成本分析

<a id="cost-modeling-dec-2025"></a>
### 成本建模（2025 年 12 月）

| 模型 | Input / 1M | Output / 1M | 備註 |
|-------|------------|-------------|-------|
| **GPT-5.5** | $5.00 | $20.00 | Agentic 溢價 |
| **Claude Opus 4.7** | $15.00 | $75.00 | 專注工程任務 |
| **Claude Sonnet 4.6** | $3.00 | $15.00 | 均衡選擇 |
| **Gemini 3.0 Pro** | $1.25 | $5.00 | 最有價值的 frontier 選項 |
| **Gemini 3 Flash** | $0.05 | $0.20 | 大規模 RAG 贏家 |
| **o4-mini** | $0.10 | $0.40 | 平價邏輯推理 |

<a id="cost-comparison-example"></a>
### 成本比較範例

假設每月 1M 次查詢，每次 1K input tokens + 500 output tokens：

| 量級 | GPT-5.5 | Claude Sonnet | Gemini 3 Pro | Gemini 3 Flash |
|--------|---------|---------------|--------------|----------------|
| 每月 10K 次查詢 | $150 | $105 | $37.50 | $1.50 |
| 每月 1M 次查詢 | $15,000 | $10,500 | $3,750 | $150 |

*2025 年洞察：Gemini 3 Flash 幾乎已把 RAG 商品化，在大規模場景下，長上下文處理甚至比傳統向量搜尋基礎設施更便宜。*

---

<a id="operational-considerations"></a>
## 營運考量

<a id="rate-limits-and-quotas"></a>
### Rate Limits 與配額

| Provider | Tier | RPM | TPM |
|----------|------|-----|-----|
| OpenAI（Tier 1） | Basic | 500 | 30K |
| OpenAI（Tier 5） | Enterprise | 10K | 10M |
| Anthropic（Tier 1） | Basic | 50 | 40K |
| Anthropic（Tier 4） | Enterprise | 4K | 400K |

<a id="reliability-patterns"></a>
### 可靠性模式

```python
class ReliableModelClient:
    def __init__(self):
        self.providers = {
            "primary": OpenAIClient(),
            "fallback1": AnthropicClient(),
            "fallback2": GoogleClient()
        }
    
    async def generate(self, prompt: str) -> str:
        for name, client in self.providers.items():
            try:
                return await client.generate(prompt)
            except RateLimitError:
                continue
            except ServiceError:
                continue
        
        raise AllProvidersUnavailable()
```

<a id="abstraction-layer"></a>
### 抽象層

```python
class LLMClient:
    """Unified interface for multiple providers."""
    
    def __init__(self, config: dict):
        self.default_model = config["default_model"]
        self.clients = self._init_clients(config)
    
    async def generate(
        self,
        messages: list[dict],
        model: str = None,
        **kwargs
    ) -> str:
        model = model or self.default_model
        client = self._get_client(model)
        
        # Normalize request format
        normalized = self._normalize_request(messages, kwargs)
        
        # Call provider
        response = await client.generate(**normalized)
        
        # Normalize response
        return self._normalize_response(response)
    
    def _normalize_request(self, messages: list[dict], kwargs: dict) -> dict:
        # Handle differences between providers
        # OpenAI uses 'messages', Anthropic uses 'messages' with different format
        pass
```

---

<a id="multi-model-strategies"></a>
## 多模型策略

<a id="model-routing"></a>
### 模型路由

```python
class ModelRouter:
    def __init__(self):
        self.classifier = QueryClassifier()
        self.models = {
            "simple": "gpt-4o-mini",
            "complex": "claude-3.5-sonnet",
            "code": "claude-3.5-sonnet",
            "long_context": "gemini-1.5-pro",
            "reasoning": "o1-mini"
        }
    
    async def route(self, query: str, context_length: int) -> str:
        # Classify query complexity
        query_type = await self.classifier.classify(query)
        
        # Override for long context
        if context_length > 100_000:
            return self.models["long_context"]
        
        return self.models[query_type]
```

<a id="cascade-pattern-2025-refinement"></a>
### 級聯模式（2025 年版精修）

**核心邏輯：** 能由 1B 模型完成的任務，就不要動用 70B 模型。用「Router」來評估信心分數。

```python
class ModelCascade:
    """The 'Efficiency First' Pattern."""
    
    async def generate_optimized(self, query: str):
        # 1. Draft check (SLM / Classifier)
        if is_simple_intent(query):
            return await gpt4o_mini.generate(query)
            
        # 2. Main Generation (Efficient model)
        response = await claude_sonnet.generate(query)
        
        # 3. Validation / Escalate
        if needs_verification(response):
            return await o3.generate(f"Verify this: {response}")
            
        return response
```

**Principal 級建議：** 實作「Semantic Fallback」——發生錯誤時，不要只重試同一模型，而是立即跳到更大的模型或不同供應商（OpenAI -> Anthropic），以避免相關性故障。

---

<a id="interview-questions"></a>
## 面試題

<a id="q-how-do-you-choose-between-gpt-4o-claude-and-gemini-for-a-production-application"></a>
### Q：在正式環境應用中，你會如何在 GPT-4o、Claude 與 Gemini 之間做選擇？

**強答範例：**

「我的選擇會取決於具體需求：

**對大多數正式環境工作負載來說**，我通常先從 Claude 3.5 Sonnet 或 GPT-4o 開始。兩者都是很好的通用模型。Sonnet 在 coding 上略有優勢，GPT-4o 則有較佳的生態整合。

**對長上下文應用來說**，Gemini 1.5 Pro 憑藉 100-200 萬 token context 明顯勝出。如果我要處理整個 codebase 或非常長的文件，Gemini 會是我的選擇。

**對成本敏感且高流量的場景**，我會選 GPT-4o-mini 或 Claude Haiku。它們便宜 10-20 倍，而且能很好處理直接明確的任務。

**我的實務做法：**
1. 先用 Sonnet 或 GPT-4o 做原型，驗證 use case
2. 針對**我自己的任務**做評估，而不是只看 benchmarks
3. 建立 abstraction layer，方便日後切換
4. 透過把簡單請求路由到便宜模型來最佳化成本

我絕不只依賴 benchmark 分數。某個模型在 MMLU 排名較低，仍可能在我的領域明顯更強。」

<a id="q-when-would-you-self-host-vs-use-api-providers"></a>
### Q：你會在什麼情況下自行託管，而不是使用 API 供應商？

**強答範例：**

「這本質上是控制權與營運負擔之間的取捨。

**適合使用 API 的情況：**
- 流量低於每月 1M 次查詢（成本交叉點之前）
- 需要立即用到最新模型
- 團隊缺乏 GPU 基礎設施專業能力
- 工作負載波動大、難以做容量規劃
- 上市時間非常關鍵

**適合自行託管的情況：**
- 資料不得離開基礎設施（合規）
- 流量超過每月 10M 次查詢（有明顯成本優勢）
- 需要 P99 低於 100ms 的延遲
- 需要自訂模型權重或 fine-tuning
- 需要完整控制模型行為

**Hybrid 往往最好：**
- 高流量且可預測的工作負載用 self-host
- 尖峰與特化模型用 API
- self-hosted 失效時，API 作為 fallback

自行託管的隱性成本包括：GPU 採購、工程時間、模型更新與監控。把 1-2 位專職基礎設施工程師也算進去。」

---

<a id="references"></a>
## 參考資料

- OpenAI API: https://platform.openai.com/
- Anthropic API: https://docs.anthropic.com/
- Google AI: https://ai.google.dev/
- LMSys Leaderboard: https://chat.lmsys.org/

---

*下一篇：[Fine-Tuning Guide](../03-fine-tuning/01-when-to-fine-tune.md)*

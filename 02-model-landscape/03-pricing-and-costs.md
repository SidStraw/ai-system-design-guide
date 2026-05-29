<a id="pricing-and-costs"></a>
# 定價與成本

理解 LLM 系統的成本結構，對正式環境規劃至關重要。本章涵蓋定價模型、成本最佳化策略，以及總持有成本分析。

<a id="table-of-contents"></a>
## 目錄

- [定價模型](#pricing-models)
- [目前 API 定價](#current-api-pricing)
- [成本計算](#cost-calculation)
- [成本最佳化策略](#cost-optimization-strategies)
- [Context Caching 經濟學](#context-caching-economics)
- [自行託管與 GPU 雲端套利](#self-hosting-economics)
- [總持有成本](#total-cost-of-ownership)
- [面試題](#interview-questions)
- [參考資料](#references)

---

<a id="pricing-models"></a>
## 定價模型

<a id="token-based-pricing"></a>
### 以 Token 計價

大多數 LLM API 會按 token 收費：

```
Cost = (input_tokens × input_rate) + (output_tokens × output_rate)
```

**關鍵觀察：**
- output tokens 通常比 input tokens 貴 2-5 倍
- 定價會依模型層級有顯著差異
- 部分供應商會提供 batch 折扣

<a id="tiered-pricing"></a>
### 分級定價

部分供應商提供用量折扣：

| 層級 | 每月支出 | 折扣 |
|------|---------------|----------|
| Standard | $0 - $5K | 0% |
| Growth | $5K - $50K | 10-20% |
| Enterprise | $50K+ | 客製談判 |

<a id="commitment-based-pricing"></a>
### 承諾型定價

預先購買 tokens 以換取折扣價：

```
Standard: $2.50 / 1M input tokens
Committed (1-year): $2.00 / 1M input tokens (20% savings)
```

---

<a id="current-api-pricing"></a>
## 目前 API 定價

<a id="may-2026-pricing"></a>
### 2026 年 5 月定價

> **最後驗證：2026 年 5 月 25 日。** 價格變動頻繁。請務必再次確認：[OpenAI](https://developers.openai.com/api/docs/pricing)、[Anthropic](https://platform.claude.com/docs/en/about-claude/pricing)、[Google](https://ai.google.dev/gemini-api/docs/pricing)、[xAI](https://docs.x.ai/developers/models)、[DeepSeek](https://api-docs.deepseek.com/quick_start/pricing)
>
> **2026 年生效的淘汰項目：** OpenAI 於 2026 年 2 月 13 日將 GPT-4o、GPT-4.1、GPT-4.1-mini、o4-mini 自 ChatGPT 下架；gpt-5.2-chat-latest 與 gpt-5.3-chat-latest 於 2026 年 5 月 8 日棄用；Realtime API Beta 於 2026 年 5 月 12 日移除；Sora app 於 2026 年 4 月 26 日關閉（API EOL 為 2026 年 9 月 24 日）。Google Vertex 於 2026 年 3 月 26 日淘汰 `gemini-3-pro-preview`；Project Mariner 於 2026 年 5 月 4 日關閉。Gemini 2.5 Pro／Flash 於 2026 年 6 月 17 日棄用。
>
> **價格變動：** DeepSeek 於 2026 年 5 月 22 日將 V4 Pro 的 75% 折扣改為 **永久**：自 2026 年 6 月 1 日起，新牌價降為原價的 25%（每 1M input/output 為 $0.435 / $0.87），且所有 DeepSeek 模型的 cache-hit input 價格已於 2026 年 4 月 26 日降至發布價的 1/10。DeepSeek V4 Flash（每 1M 為 $0.14 / $0.28，1M context）現在以極大優勢成為最便宜的 frontier-class API。

<a id="openai-gpt-5x-generation"></a>
#### OpenAI（GPT-5.x 世代）
| 模型 | Input / 1M | Output / 1M | 備註 |
|-------|------------|-------------|-------|
| **GPT-5.5** ⭐ NEW | $5.00 | $30.00 | 2026 年 4 月 23 日發布。1M context。全新多模態旗艦。 |
| **GPT-5.5 Instant** ⭐ NEW | check latest | check latest | 自 2026 年 5 月 5 日起成為 ChatGPT 與 `chat-latest` 的預設。高風險提示下幻覺減少 52.5%。 |
| **GPT-Realtime-2** ⭐ NEW | $32.00 (audio) | $64.00 (audio) | 2026 年 5 月 7 日發布。GPT-5 等級的即時語音。 |
| **GPT-Realtime-Translate** ⭐ NEW | (audio pricing) | (audio pricing) | 70+ 種輸入 → 13 種輸出語言。 |
| **GPT-5.4 Pro** | $30.00 | $180.00 | 最強推理；長上下文價格翻倍為 $60/$270 |
| **GPT-5.4** | $2.50 | $15.00 | 旗艦模型；原生 computer use；cached input 為 $1.25 |
| **GPT-5.4-mini** | $0.75 | $4.50 | GPT-5 層級最佳成本／效能 |
| **GPT-5.4-nano** | check latest | check latest | 最小的 GPT-5.4 版本；2026 年 3 月發布 |
| **GPT-4o** | $2.50 | $10.00 | 已於 2026 年 2 月 13 日自 ChatGPT 下架；API 可用性依情況而定 |
| **GPT-4o-mini** | $0.15 | $0.60 | 舊版；請確認 API 是否仍可使用 |

<a id="anthropic-claude-4x-generation"></a>
#### Anthropic（Claude 4.x 世代）
| 模型 | Input / 1M | Output / 1M | Context | 備註 |
|-------|------------|-------------|---------|-------|
| **Claude Opus 4.7** ⭐ NEW | $5.00 | $25.00 | 1M | 2026 年 4 月 16 日於 API、Bedrock、Vertex、Microsoft Foundry 發布。更高解析度 vision，SWE 表現更好。 |
| **Claude Opus 4.6** | $5.00 | $25.00 | 1M | 128K max output；adaptive thinking 以標準價格計費 |
| **Claude Sonnet 4.6** | $3.00 | $15.00 | 1M | 以更低成本涵蓋大多數 Opus 級任務 |
| **Claude Haiku 4.5** | $1.00 | $5.00 | 200K | Anthropic 最快模型；精確價格請查最新資料 |
| **Claude Mythos Preview** | n/a | n/a | - | 僅限約 11 個 Project Glasswing 合作夥伴；未公開販售。 |

> [!NOTE]
> **Claude 1M context 採標準定價：** Opus 4.6 與 Sonnet 4.6 以標準價格就包含完整 1M token context window——長上下文沒有額外溢價。Batch API 提供 50% 折扣。Cache hit 成本為標準 input 價格的 10%。

<a id="google-gemini-3x-generation"></a>
#### Google（Gemini 3.x 世代）
| 模型 | Input / 1M | Output / 1M | Context | 備註 |
|-------|------------|-------------|---------|-------|
| **Gemini 3.1 Pro** | $2.00 | $12.00 | 1M | 200K+ context：$4.00/$18.00 |
| **Gemini 3.1 Flash** | $0.10 | $3.00 | 1M | 最佳價格／效能；適合高流量 |
| **Gemini 2.5 Flash-Lite** | $0.10 | $0.40 | 1M | 2026 年 6 月棄用 |

> [!WARNING]
> **Gemini 2.5 棄用：** Gemini 2.5 Pro 與 2.5 Flash 預計於 2026 年 6 月 17 日棄用。請遷移至 Gemini 3.x 模型。

<a id="xai-grok"></a>
#### xAI（Grok）
| 模型 | Input / 1M | Output / 1M | Context | 備註 |
|-------|------------|-------------|---------|-------|
| **Grok 4** | $3.00 | $15.00 | 256K | 原生 tool use；即時搜尋 |
| **Grok 4.1 Fast** | $0.20 | $0.50 | 2M | 高流量、低成本 |
| **Grok 3 mini** | check latest | check latest | - | 更快，但較不精確 |

<a id="open-weight-models-via-api-may-2026"></a>
#### 透過 API 提供的 Open-Weight 模型（2026 年 5 月）
| 模型 | Input / 1M | Output / 1M | Context | 供應商範例 |
|-------|------------|-------------|---------|-------------------|
| **DeepSeek-V3.2** | $0.28 | $0.42 | 128K | DeepSeek API。98% cache-hit 折扣。透過 routing，實際費率可再降 10–30×。 |
| **DeepSeek V4 Pro** ⭐ NEW | $0.435 | $0.87 | 1M | DeepSeek API。75% 促銷折扣已改為 **永久**：自 2026 年 6 月 1 日起，新牌價為原始價格（$1.74 / $3.48）的 25%。Cache-hit input：$0.003625/M。1M tokens 下只需 V3.2 約 27% 算力／10% 記憶體。 |
| **DeepSeek V4 Flash** ⭐ NEW | $0.14 | $0.28 | 1M | DeepSeek API。Cache-hit input：$0.0028/M（98% 折扣）。13B-active MoE。目前最便宜的 frontier-class 1M-context API。 |
| **Mistral Medium 3.5** ⭐ NEW | $1.50 | check latest | 256K | Mistral API。整合 chat／reasoning／coding／vision；SWE-Bench Verified 77.6%。 |
| **Kimi K2.6** ⭐ NEW | check latest | check latest | - | Moonshot API。1T MoE / 32B active；agent swarm 可達 300 個 sub-agents。 |
| **Qwen 3.6-35B-A3B** ⭐ NEW | check latest | check latest | - | Apache 2.0 權重；可 self-host 或由 API 供應商託管。 |
| **Llama 4 Scout** | $0.11 | $0.34 | 10M | Together AI、Groq、Fireworks。注意：超過 32K 後有效 context 會快速下降。 |
| **Llama 4 Maverick** | $0.27 | $0.85 | 1M | Together AI、Groq、Fireworks。需要支援 MoE 的 serving。 |
| **DeepSeek-V3** | $0.25 | $1.10 | 128K | DeepSeek API、Together AI |
| **DeepSeek-R1** | $0.55 | $2.19 | 128K | DeepSeek API |
| **Mistral Large 3** | $0.50 | $1.50 | 256K | Mistral API、AWS Bedrock |
| **Llama 3.3 70B** | ~$0.10–0.20 | ~$0.30–0.60 | 128K | Groq、Together AI |
| **Qwen2.5-Coder-32B** | ~$0.50 | ~$1.00 | 32K | Together AI |
| **Gemma 4 (31B / 26B-A4B MoE / E4B / E2B)** ⭐ NEW | self-host | self-host | 256K | Apache 2.0。140+ 種語言；原生 vision/audio；function calling。 |

<a id="embedding-models-may-2026"></a>
#### Embedding 模型（2026 年 5 月）
| 模型 | 每 1M tokens 成本 | 維度 |
|-------|------------------|-----------|
| **Cohere Embed 4** ⭐ NEW | $0.10 | 256 / 512 / 1024 / 1536（Matryoshka） |
| **text-embedding-3-large** | $0.13 | 3072 |
| **text-embedding-3-small** | $0.02 | 1536 |
| **Voyage-3** | $0.06 | 1024 |
| **Cohere embed-v3** | $0.10 | 1024 |

> [!IMPORTANT]
> **Inference-time Compute 成本：** 對於具有「Extended Thinking」或 reasoning mode 的模型（GPT-5.4 Pro、Claude Opus 4.6），即使使用者看不到內部思考 tokens，你仍然要為其 **internal thinking tokens** 付費。這可能讓偏邏輯任務的單次請求成本提高 2x-10x。正式環境中一定要設定 `budget_tokens` 上限。

---

<a id="cost-calculation"></a>
## 成本計算

<a id="basic-cost-formula"></a>
### 基本成本公式

```python
def calculate_request_cost(
    input_tokens: int,
    output_tokens: int,
    model: str
) -> float:
    pricing = {
        "gpt-5.4": {"input": 2.50, "output": 15.00},
        "gpt-5.4-mini": {"input": 0.75, "output": 4.50},
        "claude-sonnet-4.6": {"input": 3.00, "output": 15.00},
        "claude-opus-4.6": {"input": 5.00, "output": 25.00},
        "gemini-3.1-flash": {"input": 0.10, "output": 3.00},
    }
    
    rates = pricing[model]
    cost = (
        (input_tokens / 1_000_000) * rates["input"] +
        (output_tokens / 1_000_000) * rates["output"]
    )
    return cost
```

<a id="example-cost-calculations"></a>
### 成本計算範例

**情境 1：RAG Chatbot**
```
Per request:
- System prompt: 500 tokens
- Retrieved context: 2,000 tokens
- User message: 100 tokens
- Response: 300 tokens

Input: 2,600 tokens, Output: 300 tokens

GPT-5.4 cost: (2600 × $2.50 + 300 × $15) / 1M = $0.0110 per request

At 10,000 requests/day:
Daily: $95
Monthly: $2,850
```

**情境 2：文件摘要**
```
Per document:
- Document: 8,000 tokens
- Summary: 500 tokens

GPT-5.4 cost: (8000 × $2.50 + 500 × $15) / 1M = $0.0275

1,000 documents: $27.50
10,000 documents: $275
```

<a id="monthly-cost-projection"></a>
### 每月成本預估

```python
def project_monthly_cost(
    requests_per_day: int,
    avg_input_tokens: int,
    avg_output_tokens: int,
    model: str
) -> dict:
    per_request = calculate_request_cost(
        avg_input_tokens, avg_output_tokens, model
    )
    
    daily = per_request * requests_per_day
    monthly = daily * 30
    yearly = monthly * 12
    
    return {
        "per_request": per_request,
        "daily": daily,
        "monthly": monthly,
        "yearly": yearly
    }

# Example
costs = project_monthly_cost(
    requests_per_day=50000,
    avg_input_tokens=2000,
    avg_output_tokens=400,
    model="gpt-5.4"
)
# Output: ~$18,750/month
```

---

<a id="cost-optimization-strategies"></a>
## 成本最佳化策略

<a id="strategy-1-model-routing"></a>
### 策略 1：模型路由

將請求導向合適的模型層級：

```python
class ModelRouter:
    def __init__(self):
        self.classifier = load_complexity_classifier()
    
    def route(self, query: str, context: str) -> str:
        complexity = self.classifier.predict(query)
        
        if complexity < 0.3:
            return "gpt-5.4-mini"  # Simple queries
        elif complexity < 0.7:
            return "gpt-5.4-mini"  # Medium, try cheap first
        else:
            return "gpt-5.4"  # Complex queries

    def route_with_fallback(self, query: str, context: str) -> str:
        # Try cheap model first
        response = self.try_model("gpt-5.4-mini", query, context)

        if self.is_quality_sufficient(response):
            return response

        # Fallback to expensive model
        return self.try_model("gpt-5.4", query, context)
```

**可能節省：** 在幾乎不影響品質的前提下節省 50-70%

<a id="strategy-2-prompt-optimization"></a>
### 策略 2：Prompt 最佳化

在不犧牲品質的前提下降低 token 數量：

```python
# Before: 2,500 tokens
system_prompt = """
You are a helpful customer support assistant for Acme Corp. 
You have access to our product documentation and should answer 
questions accurately and helpfully. Always be polite and professional.
If you don't know something, say so rather than making things up.
Format your responses clearly with bullet points when listing items.
[... more verbose instructions ...]
"""

# After: 800 tokens
system_prompt = """
You are Acme Corp's support assistant.
Rules:
- Answer from provided context only
- Admit uncertainty
- Use bullet points for lists
- Be concise
"""

# Savings: 1,700 tokens × $2.50/1M = $0.00425 per request
# At 10K requests/day: $42.50/day = $1,275/month
```

<a id="strategy-3-caching"></a>
### 策略 3：快取

對重複或相似查詢快取回應：

```python
class ResponseCache:
    def __init__(self, ttl_seconds: int = 3600):
        self.exact_cache = TTLCache(maxsize=10000, ttl=ttl_seconds)
        self.semantic_cache = SemanticCache(threshold=0.95)
    
    def get_or_generate(self, query: str, context: str) -> tuple[str, bool]:
        # Check exact cache
        cache_key = self.make_key(query, context)
        if cache_key in self.exact_cache:
            return self.exact_cache[cache_key], True  # Cache hit
        
        # Check semantic cache
        similar = self.semantic_cache.find_similar(query)
        if similar:
            return similar.response, True  # Semantic hit
        
        # Generate new response
        response = self.generate(query, context)
        self.exact_cache[cache_key] = response
        self.semantic_cache.add(query, response)
        
        return response, False  # Cache miss

# With 30% cache hit rate:
# Baseline: $3,000/month
# With caching: $2,100/month
# Savings: $900/month
```

<a id="strategy-4-batch-processing"></a>
### 策略 4：批次處理

將多個請求一起處理以提高效率：

```python
# Real-time: pay full price
for query in queries:
    response = model.generate(query)

# Batch API (OpenAI offers 50% discount):
batch_responses = model.batch_generate(queries)
# Cost: 50% of real-time pricing
```

<a id="strategy-5-output-length-control"></a>
### 策略 5：控制輸出長度

適當限制回應長度：

```python
# Reduce unnecessary output
response = model.generate(
    prompt=prompt,
    max_tokens=300,  # Limit output
    stop=["\n\n"]    # Stop at natural break
)

# Cost impact:
# Before: avg 500 output tokens = $0.0075 per request (GPT-5.4)
# After: avg 250 output tokens = $0.00375 per request
# Savings: 50% on output costs
```

<a id="cost-optimization-summary"></a>
### 成本最佳化摘要

| 策略 | 工作量 | 可能節省 |
|----------|--------|-------------------|
| 模型路由 | 中 | 50-70% |
| **Context Caching** | 低 | **60-90%（Input）** |
| Prompt 最佳化 | 低 | 20-40% |
| 回應快取 | 中 | 20-40% |
| 批次處理 | 低 | 50%（OpenAI／Anthropic） |

---

<a id="context-caching-economics"></a>
## Context Caching 經濟學

**RAG 的「黃金法則」（到 2026 年依然成立）。**
如果你有固定 system prompt，或共用知識庫前綴超過 10,000 tokens，**Context Caching** 就是必須的。

**損益兩平分析（Claude Sonnet 4.6）：**
- **標準 Input**：$3.00 / 1M tokens
- **Cached Input**：$0.30 / 1M tokens（90% 折扣）
- **Cache Write Fee**：$3.75 / 1M tokens（5 分鐘 TTL，1.25x）；$6.00（1 小時 TTL，2x）

`Break-even = (Write Fee) / (Standard Rate - Cached Rate) ≈ 1.4 requests (5-min) or 2.2 requests (1-hour)`

如果你的長前綴會被 **超過 2 位使用者** 重複使用，那麼快取它一定比每次原樣傳送更便宜。OpenAI 與 Anthropic 現在都提供 batch API 折扣（5 折），可與 caching 疊加。

---

<a id="self-hosting--gpu-cloud-arbitrage"></a>
<a id="self-hosting-economics"></a>
## 自行託管與 GPU 雲端套利

**Reserved 與 Serverless 的取捨：**

| 模型大小 | Serverless（RunPod／Together） | Reserved（Lambda／AWS） |
|------------|-----------------------------|-----------------------|
| **Burst Capacity** | 幾乎無上限（但有 cold starts） | 固定 |
| **Utilization** | 只為實際算力時間付費 | 24/7 固定成本 |
| **TCO Break-even**| **利用率 < 40% 時較划算** | **利用率 > 40% 時較划算** |

**Principal 級細節：**
「GPU Cloud Arbitrage」是指根據 **Spot Instance 可用性**，在不同供應商之間搬移正式環境工作負載。到了 2025-26 年，像 **Skypilot** 這類工具可自動化這件事，透過追逐全球「低需求」區域，最多可節省 60% 的 self-hosting 成本。MoE 模型的興起（Llama 4 Scout 可放進單張 H100，Maverick 約需 ~2x H100）也使 self-hosting 相較 dense 模型所需的 GPU 明顯下降。

<a id="when-self-hosting-makes-sense"></a>
### 什麼情況下適合自行託管

```
Break-even analysis:

API cost at scale:
- 1M requests/month
- 2,500 tokens average
- GPT-5.4: ~$37,500/month
- Claude Sonnet 4.6: ~$30,000/month

Self-hosted equivalent (Llama 4 Maverick via MoE):
- 2x H100 80GB: ~$6/hour × 730 = $4,380/month
- Engineering time: $5,000/month (0.5 FTE)
- Ops overhead: $2,000/month
- Total: ~$11,380/month

Savings vs GPT-5.4: $26,120/month = 70%
Savings vs Claude Sonnet 4.6: $18,620/month = 62%
```

<a id="self-hosting-cost-components"></a>
### 自行託管成本組成

| 項目 | 每月成本 | 備註 |
|-----------|--------------|-------|
| GPU 算力 | $5K-20K | 視模型大小而定 |
| 儲存 | $200-500 | 模型權重、日誌 |
| 網路 | $100-500 | Egress、負載平衡 |
| 工程人力 | $5K-15K | 部分 FTE 用於營運 |
| 監控 | $100-500 | 可觀測性工具 |

<a id="gpu-requirements-by-model-size"></a>
### 依模型大小的 GPU 需求

| 模型大小 | GPU 配置 | 預估月成本 |
|------------|------------|---------------------|
| 7B（INT4） | 1x A10G | $500-800 |
| 7B（FP16） | 1x A100 40GB | $1,500-2,500 |
| 70B（INT4） | 2x A100 80GB | $5,000-8,000 |
| 70B（FP16） | 4x A100 80GB | $10,000-15,000 |
| 405B（INT4） | 8x H100 | $20,000-30,000 |

<a id="decision-framework"></a>
### 決策框架

```
Choose API when:
- Volume < 100K requests/month
- No ML ops expertise
- Need highest quality (frontier models)
- Fast iteration needed

Choose self-hosting when:
- Volume > 500K requests/month
- Have ML infrastructure team
- Data privacy requirements
- Predictable, stable workload
- Custom fine-tuning needed
```

---

<a id="total-cost-of-ownership"></a>
## 總持有成本

<a id="tco-components"></a>
### TCO 組成

```python
def calculate_tco(scenario: dict) -> dict:
    # Direct costs
    api_or_compute = scenario["monthly_api_cost"]
    
    # Engineering costs
    development = scenario["dev_hours"] * scenario["engineer_rate"]
    maintenance = scenario["maintenance_hours"] * scenario["engineer_rate"]
    
    # Infrastructure
    vector_db = scenario["vector_db_cost"]
    monitoring = scenario["monitoring_cost"]
    
    # Indirect costs
    downtime_risk = scenario["expected_downtime_hours"] * scenario["revenue_per_hour"]
    
    monthly_tco = (
        api_or_compute +
        development / 12 +  # Amortized over year
        maintenance +
        vector_db +
        monitoring +
        downtime_risk
    )
    
    return {
        "monthly_tco": monthly_tco,
        "yearly_tco": monthly_tco * 12,
        "breakdown": {
            "llm": api_or_compute,
            "engineering": development / 12 + maintenance,
            "infrastructure": vector_db + monitoring,
            "risk": downtime_risk
        }
    }
```

<a id="example-tco-comparison"></a>
### TCO 比較範例

**情境：客服 Bot（每月 50K 次請求）**

| 成本項目 | API 型 | 自託管 |
|----------------|-----------|-------------|
| LLM 成本 | $5,000 | $3,000 |
| Vector DB | $70 | $200 |
| 工程（每月） | $500 | $3,000 |
| 監控 | $100 | $200 |
| **月總成本** | **$5,670** | **$6,400** |

*在這種規模下，因工程負擔較低，API 更便宜。*

**情境：大型 RAG（每月 2M 次請求）**

| 成本項目 | API 型 | 自託管 |
|----------------|-----------|-------------|
| LLM 成本 | $50,000 | $15,000 |
| Vector DB | $500 | $1,000 |
| 工程（每月） | $1,000 | $8,000 |
| 監控 | $200 | $500 |
| **月總成本** | **$51,700** | **$24,500** |

*在這種規模下，自行託管明顯更便宜。*

---

<a id="interview-questions"></a>
## 面試題

<a id="q-how-would-you-optimize-costs-for-a-high-volume-rag-application"></a>
### Q：你會如何為高流量 RAG 應用最佳化成本？

**強答範例：**
我會分層處理成本最佳化：

**1. 架構最佳化：**
- 模型路由：簡單查詢用便宜模型
- 快取：30-40% 的查詢可能可被快取
- Prompt 壓縮：盡量縮減 system prompt tokens

**2. 模型選擇：**
```
Simple queries (60%): GPT-5.4-mini at $0.003/request
Complex queries (40%): GPT-5.4 at $0.011/request
Weighted avg: $0.0062/request (vs $0.011 all GPT-5.4)
Savings: 44%
```

**3. 基礎設施：**
- 批次更新 embeddings（便宜 50%）
- 為 vector DB 選擇合適規模
- 能用就用 spot instances

**4. 監控：**
- 追蹤各查詢類型的單次成本
- 異常告警
- 定期成本檢視

<a id="q-when-would-you-recommend-self-hosting-vs-using-apis"></a>
### Q：你會在什麼情況下建議自行託管，而不是使用 API？

**強答範例：**
這取決於多個因素：

**流量門檻：**
- 低於每月 100K：幾乎一律 API
- 100K-500K：逐案評估
- 高於 500K：通常 self-hosting 會勝出

**團隊能力：**
- 沒有 ML ops 能力：無論規模都用 API
- 基礎設施團隊很強：可更早考慮 self-hosting

**品質需求：**
- 需要絕對最佳：API（frontier models）
- 只要夠好即可：self-hosted open models

**其他因素：**
- 資料隱私：可能迫使你 self-host
- 延遲控制：self-hosting 有更高掌控力
- Fine-tuning 需求：self-hosting 更容易客製化

**我的建議流程：**
1. 先用 API 取得最快迭代速度
2. 建立模型切換 abstraction layer
3. 當月支出超過 $10K 時評估 self-hosting
4. 在真正投入前，先以 shadow deployment 做試點

---

<a id="references"></a>
## 參考資料

- OpenAI Pricing: https://developers.openai.com/api/docs/pricing
- Anthropic Pricing: https://platform.claude.com/docs/en/about-claude/pricing
- Google AI Pricing: https://ai.google.dev/gemini-api/docs/pricing
- xAI Pricing: https://docs.x.ai/developers/models
- Mistral Pricing: https://docs.mistral.ai/getting-started/changelog
- Lambda Labs GPU Pricing: https://lambdalabs.com/service/gpu-cloud
- RunPod Pricing: https://www.runpod.io/pricing
- LLM Pricing Comparison: https://pricepertoken.com/

---

*上一篇：[Capability Assessment](02-capability-assessment.md) | 下一篇：[Model Selection Guide](04-model-selection-guide.md)*

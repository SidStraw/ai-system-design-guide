<a id="reliability-patterns"></a>
# 可靠性模式

正式環境中的 LLM 系統，除了基本重試邏輯外，還需要更穩健的可靠性模式。本章介紹用於建立具韌性 AI 應用的進階模式。

<a id="table-of-contents"></a>
## 目錄

- [可靠性挑戰](#reliability-challenges)
- [重試模式](#retry-patterns)
- [斷路器](#circuit-breaker)
- [艙壁模式](#bulkhead-pattern)
- [逾時策略](#timeout-strategies)
- [優雅降級](#graceful-degradation)
- [多供應商故障切換](#multi-provider-failover)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="reliability-challenges"></a>
## 可靠性挑戰

<a id="llm-specific-failure-modes"></a>
### LLM 特有的失效模式

| 失效模式 | 原因 | 影響 |
|--------------|-------|--------|
| 速率限制 | 超出配額 | 請求被拒絕 |
| 逾時 | 生成時間過長、網路問題 | 回應變慢或失敗 |
| 供應商中斷 | 基礎設施問題 | 完全失效 |
| 品質下降 | 模型更新、負載 | 輸出品質變差 |
| 上下文溢位 | 輸入過大 | 請求失敗 |
| 輸出格式錯誤 | 生成錯誤 | 解析失敗 |

<a id="reliability-targets"></a>
### 可靠性目標

| 等級 | 可用性 | p99 延遲 | 範例 |
|------|--------------|-------------|----------|
| 關鍵 | 99.99% | < 3s | 支付處理 |
| 標準 | 99.9% | < 10s | 客戶支援 |
| 盡力而為 | 99% | < 30s | 背景任務 |

---

<a id="retry-patterns"></a>
## 重試模式

<a id="exponential-backoff-with-jitter"></a>
### 帶 Jitter 的指數退避

```python
import random
import asyncio
from typing import TypeVar, Callable

T = TypeVar("T")

class RetryConfig:
    def __init__(
        self,
        max_retries: int = 3,
        base_delay: float = 1.0,
        max_delay: float = 60.0,
        exponential_base: float = 2.0,
        jitter: float = 0.5
    ):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay
        self.exponential_base = exponential_base
        self.jitter = jitter
    
    def get_delay(self, attempt: int) -> float:
        delay = min(
            self.base_delay * (self.exponential_base ** attempt),
            self.max_delay
        )
        # 加入 jitter 以避免驚群效應
        jitter_range = delay * self.jitter
        delay += random.uniform(-jitter_range, jitter_range)
        return max(0, delay)


async def retry_with_backoff(
    func: Callable[[], T],
    config: RetryConfig,
    retryable_exceptions: tuple = (Exception,)
) -> T:
    last_exception = None
    
    for attempt in range(config.max_retries + 1):
        try:
            return await func()
        except retryable_exceptions as e:
            last_exception = e
            
            if attempt == config.max_retries:
                break
            
            delay = config.get_delay(attempt)
            await asyncio.sleep(delay)
    
    raise last_exception
```

<a id="retryable-vs-non-retryable-errors"></a>
### 可重試與不可重試錯誤

```python
class LLMRetryPolicy:
    RETRYABLE = [
        RateLimitError,
        TimeoutError,
        ServiceUnavailableError,
        ConnectionError
    ]
    
    NOT_RETRYABLE = [
        AuthenticationError,
        InvalidRequestError,
        ContentPolicyViolation,
        ContextLengthExceeded
    ]
    
    @classmethod
    def should_retry(cls, error: Exception) -> bool:
        for retryable_type in cls.RETRYABLE:
            if isinstance(error, retryable_type):
                return True
        return False
    
    @classmethod
    def get_retry_after(cls, error: Exception) -> float | None:
        # 某些 rate limit 錯誤會包含 retry-after header
        if hasattr(error, "retry_after"):
            return error.retry_after
        return None
```

---

<a id="circuit-breaker"></a>
## 斷路器

<a id="implementation"></a>
### 實作

```python
from enum import Enum
from dataclasses import dataclass
from datetime import datetime, timedelta

class CircuitState(Enum):
    CLOSED = "closed"      # 正常運作
    OPEN = "open"          # 失敗中，拒絕請求
    HALF_OPEN = "half_open"  # 測試是否恢復

@dataclass
class CircuitBreakerConfig:
    failure_threshold: int = 5
    recovery_timeout: timedelta = timedelta(seconds=30)
    half_open_max_calls: int = 3
    success_threshold: int = 2  # 關閉斷路器所需的成功次數

class CircuitBreaker:
    def __init__(self, name: str, config: CircuitBreakerConfig):
        self.name = name
        self.config = config
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time: datetime | None = None
        self.half_open_calls = 0
    
    def can_execute(self) -> bool:
        if self.state == CircuitState.CLOSED:
            return True
        
        if self.state == CircuitState.OPEN:
            # 檢查恢復逾時是否已過
            if self._recovery_timeout_elapsed():
                self._transition_to_half_open()
                return True
            return False
        
        if self.state == CircuitState.HALF_OPEN:
            # 在半開狀態下只允許有限次呼叫
            return self.half_open_calls < self.config.half_open_max_calls
        
        return False
    
    def record_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.config.success_threshold:
                self._transition_to_closed()
        else:
            self.failure_count = 0
    
    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = datetime.now()
        
        if self.state == CircuitState.HALF_OPEN:
            self._transition_to_open()
        elif self.failure_count >= self.config.failure_threshold:
            self._transition_to_open()
    
    def _transition_to_open(self):
        self.state = CircuitState.OPEN
        self.success_count = 0
    
    def _transition_to_half_open(self):
        self.state = CircuitState.HALF_OPEN
        self.half_open_calls = 0
        self.success_count = 0
    
    def _transition_to_closed(self):
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
    
    def _recovery_timeout_elapsed(self) -> bool:
        if self.last_failure_time is None:
            return True
        return datetime.now() - self.last_failure_time >= self.config.recovery_timeout
```

<a id="usage-with-llm-client"></a>
### 搭配 LLM 用戶端的用法

```python
class ResilientLLMClient:
    def __init__(self):
        self.circuit_breakers = {
            "openai": CircuitBreaker("openai", CircuitBreakerConfig()),
            "anthropic": CircuitBreaker("anthropic", CircuitBreakerConfig()),
        }
    
    async def generate(self, prompt: str, provider: str = "openai") -> str:
        cb = self.circuit_breakers[provider]
        
        if not cb.can_execute():
            raise CircuitOpenError(f"{provider} 的斷路器已開啟")
        
        try:
            result = await self._call_provider(provider, prompt)
            cb.record_success()
            return result
        except RetryableError as e:
            cb.record_failure()
            raise
```

---

<a id="bulkhead-pattern"></a>
## 艙壁模式

<a id="isolating-resources"></a>
### 隔離資源

```python
import asyncio
from contextlib import asynccontextmanager

class Bulkhead:
    """
    隔離資源以防止級聯失效。
    """
    
    def __init__(
        self,
        name: str,
        max_concurrent: int,
        max_queued: int = 100
    ):
        self.name = name
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.queue_semaphore = asyncio.Semaphore(max_queued)
    
    @asynccontextmanager
    async def acquire(self, timeout: float = 30.0):
        # 檢查佇列容量
        if not self.queue_semaphore.locked():
            await self.queue_semaphore.acquire()
        else:
            raise BulkheadFullError(f"艙壁 {self.name} 的佇列已滿")
        
        try:
            # 等待執行槽位
            acquired = await asyncio.wait_for(
                self.semaphore.acquire(),
                timeout=timeout
            )
            self.queue_semaphore.release()
            
            try:
                yield
            finally:
                self.semaphore.release()
        except asyncio.TimeoutError:
            self.queue_semaphore.release()
            raise BulkheadTimeoutError(f"艙壁 {self.name} 已逾時")


class BulkheadedLLMClient:
    def __init__(self):
        # 為不同工作負載分離艙壁
        self.bulkheads = {
            "realtime": Bulkhead("realtime", max_concurrent=50),
            "batch": Bulkhead("batch", max_concurrent=200),
            "critical": Bulkhead("critical", max_concurrent=10)
        }
    
    async def generate(
        self,
        prompt: str,
        priority: str = "realtime"
    ) -> str:
        bulkhead = self.bulkheads[priority]
        
        async with bulkhead.acquire():
            return await self._call_llm(prompt)
```

---

<a id="timeout-strategies"></a>
## 逾時策略

<a id="layered-timeouts"></a>
### 分層逾時

```python
class TimeoutConfig:
    def __init__(
        self,
        connection_timeout: float = 5.0,
        read_timeout: float = 30.0,
        total_timeout: float = 60.0
    ):
        self.connection_timeout = connection_timeout
        self.read_timeout = read_timeout
        self.total_timeout = total_timeout


class TimeoutManager:
    def __init__(self, config: TimeoutConfig):
        self.config = config
    
    async def execute_with_timeout(self, func, *args, **kwargs):
        try:
            return await asyncio.wait_for(
                func(*args, **kwargs),
                timeout=self.config.total_timeout
            )
        except asyncio.TimeoutError:
            raise LLMTimeoutError(
                f"請求在 {self.config.total_timeout}s 後逾時"
            )
```

<a id="adaptive-timeouts"></a>
### 自適應逾時

```python
class AdaptiveTimeout:
    """
    根據觀察到的延遲調整逾時值。
    """
    
    def __init__(
        self,
        initial_timeout: float = 30.0,
        min_timeout: float = 10.0,
        max_timeout: float = 120.0,
        percentile: float = 0.99
    ):
        self.min_timeout = min_timeout
        self.max_timeout = max_timeout
        self.percentile = percentile
        self.latencies: list[float] = []
        self.current_timeout = initial_timeout
    
    def record_latency(self, latency: float):
        self.latencies.append(latency)
        
        # 保留最近 1000 筆觀測
        if len(self.latencies) > 1000:
            self.latencies = self.latencies[-1000:]
        
        # 將逾時更新為 percentile + 緩衝
        if len(self.latencies) >= 10:
            sorted_latencies = sorted(self.latencies)
            idx = int(len(sorted_latencies) * self.percentile)
            p99_latency = sorted_latencies[idx]
            
            # 加上 20% 緩衝
            new_timeout = p99_latency * 1.2
            self.current_timeout = max(
                self.min_timeout,
                min(self.max_timeout, new_timeout)
            )
    
    def get_timeout(self) -> float:
        return self.current_timeout
```

---

<a id="graceful-degradation"></a>
## 優雅降級

<a id="degradation-levels"></a>
### 降級層級

```python
class DegradationLevel(Enum):
    FULL = "full"           # 所有功能
    REDUCED = "reduced"     # 較少功能
    MINIMAL = "minimal"     # 僅核心功能
    CACHED = "cached"       # 僅回傳快取結果
    OFFLINE = "offline"     # 錯誤訊息

class GracefulDegrader:
    def __init__(self):
        self.current_level = DegradationLevel.FULL
        self.health_checker = HealthChecker()
    
    async def get_response(self, query: str) -> str:
        level = await self.health_checker.get_degradation_level()
        
        if level == DegradationLevel.FULL:
            return await self.full_pipeline(query)
        
        elif level == DegradationLevel.REDUCED:
            # 跳過昂貴操作
            return await self.reduced_pipeline(query)
        
        elif level == DegradationLevel.MINIMAL:
            # 較簡單的模型，不做檢索
            return await self.minimal_pipeline(query)
        
        elif level == DegradationLevel.CACHED:
            # 只回傳快取回應
            cached = await self.cache.get_similar(query)
            if cached:
                return cached
            return "我目前遇到一些問題，請稍後再試。"
        
        else:
            return "服務暫時無法使用。"
    
    async def full_pipeline(self, query: str) -> str:
        # RAG + 前沿模型 + 集成驗證
        context = await self.retrieve(query)
        response = await self.generate(query, context, model="gpt-4o")
        verified = await self.verify(response)
        return verified
    
    async def reduced_pipeline(self, query: str) -> str:
        # RAG + 較小模型，不做驗證
        context = await self.retrieve(query)
        return await self.generate(query, context, model="gpt-4o-mini")
    
    async def minimal_pipeline(self, query: str) -> str:
        # 直接生成，使用最小模型
        return await self.generate(query, None, model="gpt-4o-mini")
```

---

<a id="multi-provider-failover"></a>
## 多供應商故障切換

<a id="provider-manager"></a>
### 供應商管理器

```python
class ProviderManager:
    def __init__(self):
        self.providers = {
            "primary": OpenAIProvider(),
            "secondary": AnthropicProvider(),
            "tertiary": GoogleProvider()
        }
        self.health = {name: True for name in self.providers}
        self.priority_order = ["primary", "secondary", "tertiary"]
    
    async def generate(self, request: dict) -> str:
        for provider_name in self.priority_order:
            if not self.health[provider_name]:
                continue
            
            provider = self.providers[provider_name]
            
            try:
                result = await provider.generate(request)
                return result
            except RetryableError as e:
                # 標記為不健康，但繼續嘗試下一個供應商
                self.health[provider_name] = False
                asyncio.create_task(
                    self._health_check_later(provider_name)
                )
                continue
        
        raise AllProvidersUnavailableError()
    
    async def _health_check_later(self, provider_name: str):
        await asyncio.sleep(30)  # 重試前先等待
        try:
            await self.providers[provider_name].health_check()
            self.health[provider_name] = True
        except:
            # 安排下一次檢查
            asyncio.create_task(self._health_check_later(provider_name))
```

<a id="request-hedging"></a>
### 請求對沖

```python
class HedgedRequest:
    """
    對多個供應商發送平行請求，使用第一個回應。
    """
    
    def __init__(self, providers: list, hedge_delay: float = 2.0):
        self.providers = providers
        self.hedge_delay = hedge_delay
    
    async def generate(self, request: dict) -> str:
        # 啟動主要請求
        tasks = [asyncio.create_task(self.providers[0].generate(request))]
        
        try:
            # 在對沖延遲內等待主要請求
            result = await asyncio.wait_for(tasks[0], timeout=self.hedge_delay)
            return result
        except asyncio.TimeoutError:
            # 主要請求太慢，啟動對沖請求
            for provider in self.providers[1:]:
                tasks.append(asyncio.create_task(provider.generate(request)))
            
            # 回傳第一個成功結果
            done, pending = await asyncio.wait(
                tasks,
                return_when=asyncio.FIRST_COMPLETED
            )
            
            # 取消尚未完成的工作
            for task in pending:
                task.cancel()
            
            # 取得已完成工作中的結果
            for task in done:
                if task.exception() is None:
                    return task.result()
            
            # 全部失敗
            raise AllProvidersFailedError()
```

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-how-do-you-design-for-high-availability-in-llm-systems"></a>
### 問：你如何為 LLM 系統設計高可用性？

**強答：**

「我會使用多層可靠性機制：

**帶退避的重試：** 對暫時性失敗使用帶 jitter 的指數退避。重點是區分可重試錯誤（速率限制、逾時）與不可重試錯誤（驗證失敗、錯誤請求）。

**斷路器：** 如果某個供應商持續失敗，就在冷卻期內停止嘗試。這能避免把延遲浪費在已故障的供應商上，也給它時間恢復。

**多供應商故障切換：** 不要依賴單一供應商。我會配置主要／次要／第三供應商，並啟用自動故障切換。每個供應商都有自己的斷路器。

**優雅降級：** 定義當沒有任何 provider 可用時要怎麼做。與其完全失敗，不如回傳降級後的回應（較簡單的模型、快取結果）。

**艙壁隔離：** 隔離不同工作負載。批次處理暴增不應拖垮即時查詢。

關鍵洞見是：預設失敗一定會發生。LLM API 比傳統 API 更不可靠。設計時就要假設供應商會掛掉，因為它終究會掛。」

<a id="q-what-is-the-difference-between-circuit-breaker-and-retry"></a>
### 問：斷路器與重試有什麼差別？

**強答：**

「它們解決的是不同問題：

**重試**處理暫時性失敗。單一請求失敗時，再試一次。它假設失敗彼此獨立，下一次嘗試可能成功。

**斷路器**處理系統性失敗。如果很多請求都失敗，就完全停止嘗試。它假設下游系統不健康，而反覆嘗試只會浪費資源、拖慢恢復。

**它們如何協作：**
1. 請求失敗 → 進行帶退避的重試（第 1、2、3 次）
2. 如果所有重試都失敗 → 斷路器記錄一次失敗
3. 失敗達到 N 次後 → 斷路器開啟，立即拒絕請求
4. 逾時後 → 斷路器進入半開狀態，允許有限的測試請求
5. 如果測試成功 → 斷路器關閉，恢復正常運作

沒有斷路器時：在服務中斷期間，每個請求都要等所有重試跑完才失敗。延遲飆升、資源耗盡。

有斷路器時：偵測到中斷後，請求會快速失敗。系統仍保持回應能力，也能切換到替代方案。」

---

<a id="references"></a>
## 參考資料

- Microsoft Resilience Patterns: https://learn.microsoft.com/en-us/azure/architecture/patterns/
- Netflix Hystrix: https://github.com/Netflix/Hystrix

---

*上一章：[集成方法](02-ensemble-methods.md)*

<a id="access-control-for-llm-systems"></a>
# LLM 系統的存取控制

安全的存取控制，對多使用者與多租戶的 LLM 應用至關重要。本章涵蓋 authentication、authorization，以及資料隔離模式。

<a id="table-of-contents"></a>
## 目錄

- [存取控制需求](#access-control-requirements)
- [驗證模式](#authentication-patterns)
- [授權模型](#authorization-models)
- [租戶隔離](#tenant-isolation)
- [API Key 管理](#api-key-management)
- [稽核與合規](#audit-and-compliance)
- [面試問題](#interview-questions)
- [參考資料](#references)

---

<a id="access-control-requirements"></a>
## 存取控制需求

<a id="security-dimensions"></a>
### 安全面向

| 面向 | 描述 | 控制措施 |
|-----------|-------------|----------|
| **Authentication** | 是誰發出這個請求？ | API keys、OAuth、JWT |
| **Authorization** | 他們可以做什麼？ | RBAC、ABAC、policies |
| **Isolation** | 他們可以看到哪些資料？ | Tenant filtering、encryption |
| **Audit** | 他們做了什麼？ | Logging、compliance reports |

<a id="llm-specific-concerns"></a>
### LLM 特有考量

| 考量 | 風險 | 緩解方式 |
|---------|------|------------|
| Prompt injection | 繞過存取控制 | 輸入驗證 |
| 資料外洩 | 跨租戶暴露 | 嚴格過濾 |
| 模型輸出 | 暴露受保護資訊 | 輸出過濾 |
| 上下文污染 | 注入未授權資料 | 上下文驗證 |

---

<a id="authentication-patterns"></a>
## 驗證模式

<a id="api-key-authentication"></a>
### API Key 驗證

```python
class APIKeyAuthenticator:
    def __init__(self, key_store):
        self.key_store = key_store
    
    async def authenticate(self, api_key: str) -> AuthResult:
        if not api_key:
            return AuthResult(authenticated=False, error="Missing API key")
        
        # Hash the key for lookup
        key_hash = self.hash_key(api_key)
        
        # Look up in store
        key_record = await self.key_store.get(key_hash)
        
        if not key_record:
            return AuthResult(authenticated=False, error="Invalid API key")
        
        if key_record.expired:
            return AuthResult(authenticated=False, error="Expired API key")
        
        if key_record.revoked:
            return AuthResult(authenticated=False, error="Revoked API key")
        
        return AuthResult(
            authenticated=True,
            user_id=key_record.user_id,
            tenant_id=key_record.tenant_id,
            scopes=key_record.scopes
        )
    
    def hash_key(self, key: str) -> str:
        return hashlib.sha256(key.encode()).hexdigest()
```

<a id="jwt-with-scopes"></a>
### 帶有 Scopes 的 JWT

```python
class JWTAuthenticator:
    def __init__(self, public_key: str):
        self.public_key = public_key
    
    async def authenticate(self, token: str) -> AuthResult:
        try:
            payload = jwt.decode(
                token,
                self.public_key,
                algorithms=["RS256"],
                audience="llm-api"
            )
            
            return AuthResult(
                authenticated=True,
                user_id=payload["sub"],
                tenant_id=payload.get("tenant_id"),
                scopes=payload.get("scopes", []),
                expires_at=datetime.fromtimestamp(payload["exp"])
            )
        except jwt.ExpiredSignatureError:
            return AuthResult(authenticated=False, error="Token expired")
        except jwt.InvalidTokenError as e:
            return AuthResult(authenticated=False, error=str(e))
```

---

<a id="authorization-models"></a>
## 授權模型

<a id="role-based-access-control-rbac"></a>
### 角色式存取控制（RBAC）

```python
class RBACAuthorizer:
    ROLE_PERMISSIONS = {
        "admin": ["*"],
        "developer": ["generate", "embed", "fine_tune", "read_metrics"],
        "user": ["generate", "embed"],
        "viewer": ["read_metrics"]
    }
    
    def authorize(self, user: User, action: str) -> bool:
        permissions = self.ROLE_PERMISSIONS.get(user.role, [])
        
        if "*" in permissions:
            return True
        
        return action in permissions
```

<a id="attribute-based-access-control-abac"></a>
### 屬性式存取控制（ABAC）

```python
class ABACAuthorizer:
    def __init__(self, policy_engine):
        self.policy_engine = policy_engine
    
    async def authorize(
        self,
        subject: dict,       # Who (user attributes)
        action: str,         # What (operation)
        resource: dict,      # On what (resource attributes)
        context: dict        # When/where (environmental)
    ) -> AuthzResult:
        # Evaluate all applicable policies
        policies = await self.policy_engine.get_policies(action)
        
        for policy in policies:
            result = policy.evaluate(subject, action, resource, context)
            if result == PolicyResult.DENY:
                return AuthzResult(allowed=False, reason=policy.name)
            if result == PolicyResult.ALLOW:
                return AuthzResult(allowed=True)
        
        return AuthzResult(allowed=False, reason="No matching policy")
```

<a id="model-level-permissions"></a>
### 模型層級權限

```python
class ModelAccessControl:
    MODEL_TIERS = {
        "gpt-4o": ["enterprise", "professional"],
        "gpt-4o-mini": ["enterprise", "professional", "starter"],
        "claude-3.5-sonnet": ["enterprise"],
        "claude-3.5-haiku": ["enterprise", "professional", "starter"]
    }
    
    def can_access_model(self, user: User, model: str) -> bool:
        allowed_tiers = self.MODEL_TIERS.get(model, [])
        return user.tier in allowed_tiers
    
    def get_available_models(self, user: User) -> list[str]:
        return [
            model for model, tiers in self.MODEL_TIERS.items()
            if user.tier in tiers
        ]
```

---

<a id="tenant-isolation"></a>
## 租戶隔離

<a id="data-isolation-patterns"></a>
### 資料隔離模式

```python
class TenantIsolatedVectorStore:
    def __init__(self, vector_db):
        self.db = vector_db
    
    async def search(
        self,
        tenant_id: str,
        query_embedding: list[float],
        top_k: int = 10
    ) -> list[dict]:
        # CRITICAL: Always filter by tenant_id at database level
        results = await self.db.search(
            query_vector=query_embedding,
            top_k=top_k,
            filter={"tenant_id": {"$eq": tenant_id}}  # Mandatory filter
        )
        
        return results
    
    async def insert(
        self,
        tenant_id: str,
        documents: list[dict]
    ):
        # CRITICAL: Always include tenant_id in metadata
        for doc in documents:
            doc["metadata"]["tenant_id"] = tenant_id
        
        await self.db.insert(documents)
```

<a id="prompt-isolation"></a>
### Prompt 隔離

```python
class TenantAwarePromptBuilder:
    def build_prompt(
        self,
        tenant_id: str,
        user_query: str,
        context: list[dict]
    ) -> str:
        # Verify all context belongs to tenant
        for doc in context:
            if doc.get("tenant_id") != tenant_id:
                raise SecurityError("Cross-tenant context detected")
        
        # Build isolated prompt
        return f"""
[Tenant: {tenant_id}]
Context from tenant documents:
{self.format_context(context)}

User query: {user_query}
"""
```

<a id="cache-isolation"></a>
### 快取隔離

```python
class TenantIsolatedCache:
    def __init__(self, cache_backend):
        self.cache = cache_backend
    
    def _scoped_key(self, tenant_id: str, key: str) -> str:
        return f"tenant:{tenant_id}:{key}"
    
    async def get(self, tenant_id: str, key: str) -> any:
        return await self.cache.get(self._scoped_key(tenant_id, key))
    
    async def set(self, tenant_id: str, key: str, value: any, ttl: int = 3600):
        await self.cache.set(
            self._scoped_key(tenant_id, key),
            value,
            ttl=ttl
        )
```

---

<a id="api-key-management"></a>
## API Key 管理

<a id="key-lifecycle"></a>
### 金鑰生命週期

```python
class APIKeyManager:
    KEY_PREFIX = "llm_"
    
    async def create_key(
        self,
        user_id: str,
        tenant_id: str,
        name: str,
        scopes: list[str],
        expires_in_days: int = 365
    ) -> APIKey:
        # Generate secure key
        raw_key = self.KEY_PREFIX + secrets.token_urlsafe(32)
        key_hash = self.hash_key(raw_key)
        
        # Store metadata (not the raw key)
        key_record = APIKeyRecord(
            id=generate_id(),
            hash=key_hash,
            user_id=user_id,
            tenant_id=tenant_id,
            name=name,
            scopes=scopes,
            created_at=datetime.now(),
            expires_at=datetime.now() + timedelta(days=expires_in_days)
        )
        
        await self.store.save(key_record)
        
        # Return raw key only once (not stored)
        return APIKey(
            id=key_record.id,
            key=raw_key,  # Only returned on creation
            name=name,
            scopes=scopes,
            expires_at=key_record.expires_at
        )
    
    async def revoke_key(self, key_id: str, reason: str):
        await self.store.update(key_id, {
            "revoked": True,
            "revoked_at": datetime.now(),
            "revoke_reason": reason
        })
        
        await self.audit_log.log("api_key_revoked", {
            "key_id": key_id,
            "reason": reason
        })
```

<a id="key-rotation"></a>
### 金鑰輪替

```python
class KeyRotator:
    async def rotate_key(self, old_key_id: str) -> APIKey:
        old_key = await self.key_store.get(old_key_id)
        
        # Create new key with same permissions
        new_key = await self.key_manager.create_key(
            user_id=old_key.user_id,
            tenant_id=old_key.tenant_id,
            name=f"{old_key.name} (rotated)",
            scopes=old_key.scopes
        )
        
        # Grace period: old key still works temporarily
        await self.key_store.update(old_key_id, {
            "deprecated": True,
            "deprecated_at": datetime.now(),
            "grace_period_ends": datetime.now() + timedelta(days=7)
        })
        
        await self.notify_user(old_key.user_id, new_key)
        
        return new_key
```

---

<a id="audit-and-compliance"></a>
## 稽核與合規

<a id="audit-logging"></a>
### 稽核日誌

```python
class AuditLogger:
    async def log_request(
        self,
        request: LLMRequest,
        response: LLMResponse,
        auth: AuthResult
    ):
        audit_entry = {
            "timestamp": datetime.now().isoformat(),
            "request_id": request.id,
            "user_id": auth.user_id,
            "tenant_id": auth.tenant_id,
            "action": "llm_generate",
            "model": request.model,
            "input_tokens": response.usage.input_tokens,
            "output_tokens": response.usage.output_tokens,
            "cost": response.cost,
            "latency_ms": response.latency_ms,
            # Hash content for privacy
            "input_hash": self.hash_content(request.prompt),
            "output_hash": self.hash_content(response.content)
        }
        
        await self.audit_store.append(audit_entry)
```

<a id="compliance-reports"></a>
### 合規報告

```python
class ComplianceReporter:
    async def generate_report(
        self,
        tenant_id: str,
        start_date: datetime,
        end_date: datetime
    ) -> ComplianceReport:
        logs = await self.audit_store.query(
            tenant_id=tenant_id,
            start=start_date,
            end=end_date
        )
        
        return ComplianceReport(
            tenant_id=tenant_id,
            period=(start_date, end_date),
            total_requests=len(logs),
            unique_users=len(set(l["user_id"] for l in logs)),
            models_used=list(set(l["model"] for l in logs)),
            total_cost=sum(l["cost"] for l in logs),
            data_access_events=self.extract_data_access(logs),
            security_events=await self.get_security_events(tenant_id, start_date, end_date)
        )
```

---

<a id="interview-questions"></a>
## 面試問題

<a id="q-how-do-you-implement-multi-tenant-isolation-in-a-rag-system"></a>
### 問：你如何在 RAG 系統中實作多租戶隔離？

**強而有力的回答：**

「多租戶隔離需要縱深防禦：

**Vector database 層：**
- 每個向量都在 metadata 中包含 tenant_id
- 所有查詢都在資料庫層以 tenant_id 過濾
- 絕不在檢索後才過濾（資料已經先洩漏到記憶體）

**Cache 層：**
- 所有 cache keys 都以 tenant_id 為前綴
- Semantic cache 需限定在租戶範圍內
- 即使查詢完全相同，也不能出現跨租戶 cache hit

**Prompt 層：**
- 在納入前，先驗證 context documents 屬於提出請求的租戶
- 絕不混用多個租戶的 context

**Output 層：**
- 驗證回應中不含跨租戶資訊
- 將 output filtering 作為額外防護

**Audit：**
- 記錄所有帶有租戶上下文的存取
- 監控跨租戶存取嘗試

關鍵原則是：tenant_id 是每個資料存取點都必須具備的強制篩選條件，而不是可有可無的參數。」

<a id="q-how-do-you-manage-api-keys-for-an-llm-service"></a>
### 問：你如何為 LLM 服務管理 API keys？

**強而有力的回答：**

「安全的 API key 管理包括：

**建立：**
- 產生具密碼學強度的隨機金鑰
- 只儲存雜湊值，原始金鑰只回傳一次
- 關聯 user、tenant、scopes 與到期時間

**驗證：**
- 將傳入金鑰做雜湊，與儲存的雜湊值比較
- 檢查是否已過期與是否被撤銷
- 驗證 scopes 是否符合請求的操作

**輪替：**
- 支援具有寬限期的金鑰輪替
- 舊金鑰在轉換期間內仍可使用（7 天）
- 通知使用者即將到期

**安全性：**
- 對失敗的驗證嘗試做速率限制
- 一旦懷疑外洩就立即撤銷
- 稽核所有金鑰操作

**Scopes：**
- 細粒度控制：模型存取、操作類型、每日限制
- 預設採最小權限

關鍵原則是：永遠不要儲存原始金鑰、支援輪替、落實最小權限。」

---

<a id="references"></a>
## 參考資料

- OAuth 2.0: https://oauth.net/2/
- OWASP API Security: https://owasp.org/API-Security/

---

*上一篇：[Security Fundamentals](01-security-fundamentals.md)*

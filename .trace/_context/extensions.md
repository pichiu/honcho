# Stage 2.4 - Extension Points

## 1. 向量儲存後端（可插拔）

### 1.1 抽象基底類別

`src/vector_store/__init__.py:64-193` 定義了 `VectorStore` ABC：

```python
class VectorStore(ABC):
    @abstractmethod
    async def upsert_many(self, namespace, vectors) -> VectorUpsertResult: ...
    @abstractmethod
    async def query(self, namespace, embedding, *, top_k, filters, max_distance) -> list[VectorQueryResult]: ...
    @abstractmethod
    async def delete_many(self, namespace, ids) -> None: ...
    @abstractmethod
    async def delete_namespace(self, namespace) -> None: ...
    @abstractmethod
    async def close(self) -> None: ...
```

### 1.2 現有實作

| 後端 | 位置 | 類別 | 用途 |
|------|------|------|------|
| pgvector（預設）| SQLAlchemy ORM 直接操作 | - | 本地 + 雲端 PostgreSQL |
| Turbopuffer | `src/vector_store/turbopuffer.py` | `TurbopufferVectorStore` | 雲端向量 DB |
| LanceDB | `src/vector_store/lancedb.py` | `LanceDBVectorStore` | 本地嵌入 |

### 1.3 如何新增向量後端

1. 建立 `src/vector_store/my_store.py`，繼承 `VectorStore`
2. 實作所有抽象方法
3. 在 `_create_store_by_type()` (`src/vector_store/__init__.py:198-209`) 新增分支
4. 在 `VectorStoreSettings.TYPE` 的 Literal 中新增類型名稱（`src/config.py:581`）

### 1.4 向量命名空間格式

```
{NAMESPACE}.doc.{SHA256(workspace, observer, observed)[:43]}   ← 文件嵌入
{NAMESPACE}.msg.{SHA256(workspace)[:43]}                       ← 訊息嵌入
```

---

## 2. LLM Provider 切換點

### 2.1 支援的 Provider

```python
# src/utils/types.py
SupportedProviders = Literal["anthropic", "google", "openai", "custom", "vllm", "groq"]
```

### 2.2 每個功能的 Provider 設定

| 功能 | 設定 key | 備用 key |
|------|---------|---------|
| Deriver | `DERIVER_PROVIDER`, `DERIVER_MODEL` | `DERIVER_BACKUP_PROVIDER`, `DERIVER_BACKUP_MODEL` |
| Summary | `SUMMARY_PROVIDER`, `SUMMARY_MODEL` | 同上 |
| Dream | `DREAM_PROVIDER`, `DREAM_MODEL` | 同上 |
| Dialectic per-level | `DIALECTIC_LEVELS__{level}__PROVIDER` | `DIALECTIC_LEVELS__{level}__BACKUP_PROVIDER` |
| Deduction specialist | 使用 Dream provider + `DREAM_DEDUCTION_MODEL` | - |
| Induction specialist | 使用 Dream provider + `DREAM_INDUCTION_MODEL` | - |
| Embedding | `LLM_EMBEDDING_PROVIDER` | - |

### 2.3 OpenAI 相容端點

設定 `LLM_OPENAI_COMPATIBLE_BASE_URL` + `LLM_OPENAI_COMPATIBLE_API_KEY`，然後設定任何功能的 provider 為 `"custom"`，即可路由到 OpenRouter、Together、Fireworks、LiteLLM 等。

---

## 3. Webhook 事件系統

### 3.1 Webhook 事件定義

`src/webhooks/events.py` 定義了可訂閱的事件：

```python
class QueueEmptyEvent:
    workspace_id: str
    queue_type: str       # "representation" 或 "summary"
    session_id: str | None
    observer: str | None
    observed: str | None
```

### 3.2 如何新增 Webhook 事件

1. 在 `src/webhooks/events.py` 定義新事件類別
2. 在適當的業務邏輯中呼叫 `publish_webhook_event(event)`
3. 佇列化後由 `src/webhooks/webhook_delivery.py` 異步遞送到已登記的端點

### 3.3 Webhook 端點管理

- `POST /v3/workspaces/{workspace_id}/webhooks` — 登記 webhook 端點
- 每個 workspace 最多 `WEBHOOK_MAX_WORKSPACE_LIMIT`（10）個端點
- 使用 HMAC 簽名驗證（`WEBHOOK_SECRET`）

---

## 4. 可配置的 Dreamer 採樣策略

### 4.1 Surprisal 樹（src/dreamer/trees/）

`src/dreamer/trees/base.py` 定義基礎介面，可新增自訂採樣演算法：

| 樹類型 | 實作檔案 | 說明 |
|--------|---------|------|
| kdtree | sklearn_wrapper.py | scikit-learn KDTree |
| balltree | sklearn_wrapper.py | scikit-learn BallTree |
| rptree | rptree.py | 隨機投影樹 |
| covertree | covertree.py | Cover Tree |
| lsh | lsh.py | 局部敏感雜湊 |
| graph | graph.py | 圖形結構 |
| prototype | prototype.py | 原型法 |

設定 `DREAM_SURPRISAL__TREE_TYPE` 選擇。

---

## 5. 設定繼承層（Configuration Hierarchy）

Peer/Session/Workspace 級的設定允許開發者精細控制哪些 peer 被觀察、摘要功能是否啟用等。

### 5.1 SessionPeer 設定（session_peers 表的 configuration 欄位）

```python
class SessionPeerConfig(BaseModel):
    observe_me: bool | None = None       # 此 peer 是否被其他 peer 觀察
    observe_others: bool | None = None   # 此 peer 是否觀察其他 peer
```

### 5.2 Peer 設定（peers 表的 configuration 欄位）

```python
class PeerConfig(BaseModel):
    observe_me: bool | None = None       # 預設觀察設定
    reasoning: ...                        # 推理設定
    summary: ...                          # 摘要設定
    dream: ...                            # Dream 設定
    peer_card: ...                        # Peer card 設定
```

### 5.3 Workspace 設定（workspaces 表的 configuration 欄位）

Workspace 級設定作為所有 peer 的預設值。

---

## 6. Telemetry 系統（可選插件）

### 6.1 CloudEvents 遙測

`src/telemetry/events/` 定義各種事件類型：

| 事件類別 | 觸發時機 |
|---------|---------|
| `AgentToolConclusionsCreatedEvent` | 觀察記錄建立時 |
| `DialecticCompletedEvent` | Dialectic 查詢完成時 |
| `DreamRunEvent` | Dream 週期完成時 |
| `DeletionCompletedEvent` | 刪除操作完成時 |
| `SyncVectorsCompletedEvent` | 向量同步完成時 |

新增事件：
1. 在 `src/telemetry/events/` 建立新事件類別（繼承 `BaseEvent`）
2. 在業務邏輯中呼叫 `emit(event)`

### 6.2 Prometheus 指標

`src/telemetry/prometheus/metrics.py` 定義指標：
- `record_api_request()` — API 請求計數
- `record_messages_created()` — 訊息建立計數
- `record_dialectic_call()` — Dialectic 呼叫計數
- `record_deriver_queue_item()` — Deriver 佇列項目計數

---

## 7. 快取層（可選）

`src/cache/client.py` 使用 `cashews` 庫提供 Redis 快取。

設定 `CACHE_ENABLED=true` 啟用。停用時所有快取操作退化為 no-op。

快取鍵使用 `CACHE_NAMESPACE` 前綴，TTL 由 `CACHE_DEFAULT_TTL_SECONDS` 控制。

---

## 8. MCP Server（src/mcp/）

提供 Model Context Protocol 整合，允許 AI assistant（如 Claude）直接透過 MCP 協議與 Honcho 互動。

工具定義在 `src/mcp/src/tools/` 目錄。

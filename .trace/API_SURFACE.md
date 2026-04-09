# API_SURFACE.md — Honcho API 參考

> 基於 Honcho v3.0.5 | API 前綴：`/v3/` | 參考來源：src/routers/ + entry_points.md

---

## 1. 認證機制

### 1.1 JWT 格式（HS256）

```
Authorization: Bearer <jwt_token>
```

**JWT Payload 欄位**（`src/security.py` `JWTParams`）：

| 欄位 | 說明 | 範例 |
|------|------|------|
| `ad` | 是否為管理員（admin） | `true` |
| `w` | workspace 名稱（範圍限制） | `"my-app"` |
| `p` | peer 名稱（範圍限制） | `"alice"` |
| `s` | session 名稱（範圍限制） | `"chat-001"` |

**範圍規則**：
- `ad=true`：可操作所有資源
- `w=X`：只能操作 workspace X 的資源
- `w=X, p=Y`：只能操作 workspace X 中 peer Y 的資源
- `w=X, p=Y, s=Z`：只能操作特定 session 的資源

### 1.2 停用認證（開發環境）

```env
AUTH_USE_AUTH=false  # 所有請求視為 admin JWT，無需 Authorization header
```

### 1.3 建立 Scoped JWT

```
POST /v3/workspaces/{workspace_id}/keys
```

```json
// Request Body
{
  "peer_id": "alice",        // 可選：限制到特定 peer
  "session_id": "session-1"  // 可選：限制到特定 session
}

// Response
{
  "key": "eyJhbGci...",      // JWT token
  "workspace_id": "my-app",
  "peer_id": "alice"
}
```

---

## 2. 通用規範

### 2.1 分頁

所有列表端點使用 `fastapi-pagination`：

```
GET /v3/workspaces/{id}/peers?page=1&size=50
```

回應格式：
```json
{
  "items": [...],
  "total": 100,
  "page": 1,
  "size": 50,
  "pages": 2
}
```

**注意**：Honcho 的「列表」端點使用 `POST /list` 而非 `GET`，因為需要傳遞 filter body。

### 2.2 ID 格式

所有公開 ID 為 nanoid(21)，格式：`[A-Za-z0-9_-]{21}`

例：`8K_gLhXmFJ3N-qy7P2wVu`

### 2.3 錯誤回應

```json
{
  "detail": "Resource not found: peer alice does not exist in workspace my-app"
}
```

| HTTP 狀態碼 | 錯誤類型 |
|------------|---------|
| 400 | ValidationException |
| 401 | AuthenticationException |
| 404 | ResourceNotFoundException |
| 422 | Pydantic ValidationError（自動）|
| 500 | 內部錯誤（記錄到 Sentry）|

---

## 3. Workspace 端點

### `POST /v3/workspaces` — 建立或取得 Workspace

```json
// Request Body
{
  "name": "my-app",           // 可選（若 JWT 包含 workspace）
  "metadata": {},
  "configuration": {}
}

// Response 200（已存在）或 201（新建）
{
  "id": "8K_gLhXmFJ3N-qy7P2wVu",
  "name": "my-app",
  "metadata": {},
  "configuration": {},
  "created_at": "2025-01-01T00:00:00Z"
}
```

### `POST /v3/workspaces/list` — 列出 Workspaces（管理員）

```json
// Request Body（可選）
{
  "filters": {"name": "my-app"}
}

// Response
{
  "items": [{ "id": "...", "name": "my-app", ... }],
  "total": 1, "page": 1, "size": 50, "pages": 1
}
```

### `GET /v3/workspaces/{workspace_id}` — 取得 Workspace

### `PUT /v3/workspaces/{workspace_id}` — 更新 Workspace

```json
{
  "metadata": {"env": "production"},
  "configuration": {"observe_me": false}
}
```

### `DELETE /v3/workspaces/{workspace_id}` — 刪除 Workspace

### `POST /v3/workspaces/search` — 搜尋 Workspaces（管理員）

---

## 4. Peer 端點

**路由前綴**：`/v3/workspaces/{workspace_id}/peers`

### `POST /v3/workspaces/{workspace_id}/peers` — 建立或取得 Peer

```json
// Request Body
{
  "name": "alice",           // 可選（若 JWT 包含 peer_id）
  "metadata": {},
  "configuration": {
    "observe_me": true,
    "reasoning": {"enabled": true}
  }
}

// Response 200（已存在）或 201（新建）
{
  "id": "alice-nanoid-21chars",
  "name": "alice",
  "workspace_id": "my-app",
  "metadata": {},
  "configuration": {},
  "created_at": "2025-01-01T00:00:00Z"
}
```

### `POST /v3/workspaces/{workspace_id}/peers/list` — 列出 Peers

### `GET /v3/workspaces/{workspace_id}/peers/{peer_id}` — 取得 Peer

### `PUT /v3/workspaces/{workspace_id}/peers/{peer_id}` — 更新 Peer

### `DELETE /v3/workspaces/{workspace_id}/peers/{peer_id}` — 刪除 Peer

---

### 4.1 Dialectic Chat（重要端點）

### `POST /v3/workspaces/{workspace_id}/peers/{peer_id}/chat`

自然語言查詢 peer 的記憶，由 DialecticAgent 處理。

```json
// Request Body
{
  "query": "What does Alice prefer for breakfast?",
  "session_id": "optional-session-id",    // 注入對話歷史
  "observer_id": "agent-1",               // 觀察者（提問方）
  "options": {
    "stream": false,                       // 是否串流回應（SSE）
    "reasoning": {
      "level": "low"                       // minimal/low/medium/high/max
    }
  }
}

// Response（stream=false）
{
  "content": "Alice prefers oatmeal with berries for breakfast.",
  "session_id": "optional-session-id",
  "usage": {
    "input_tokens": 1234,
    "output_tokens": 56
  }
}
```

**串流回應**（`stream=true`）：

Server-Sent Events 格式：
```
data: {"type": "content_block_delta", "delta": {"text": "Alice "}}
data: {"type": "content_block_delta", "delta": {"text": "prefers "}}
...
data: {"type": "message_stop"}
```

**推理等級成本對比**：

| 等級 | Provider | 模型 | 工具迭代 | 思考預算 |
|------|----------|------|---------|---------|
| minimal | Google | gemini-2.5-flash-lite | 1 | 0 |
| low | Google | gemini-2.5-flash-lite | 5 | 0 |
| medium | Anthropic | claude-haiku-4-5 | 2 | 1024 |
| high | Anthropic | claude-haiku-4-5 | 4 | 1024 |
| max | Anthropic | claude-haiku-4-5 | 10 | 2048 |

---

### 4.2 Peer Representation 端點

### `GET /v3/workspaces/{workspace_id}/peers/{peer_id}/representation`

取得觀察者對被觀察者的記憶表示（Representation）。

**Query Parameters**：
- `observer_id` (str, required): 觀察者 peer ID
- `session_id` (str, optional): 限定到特定 session
- `query` (str, optional): 語義搜尋查詢
- `top_k` (int, optional): 語義搜尋返回數量（預設 10）
- `max_distance` (float, optional): 最大語義距離
- `include_most_derived` (bool, optional): 包含最高引用次數的觀察記錄

```json
// Response
{
  "observations": [
    {
      "id": "doc-nanoid-21chars",
      "content": "Alice prefers oatmeal for breakfast",
      "level": "explicit",
      "created_at": "2025-01-01T00:00:00Z",
      "times_derived": 3
    }
  ]
}
```

---

### 4.3 Peer Card 端點

### `GET /v3/workspaces/{workspace_id}/peers/{peer_id}/card`

取得精簡的 peer 摘要（最多 40 條事實，低延遲）。

**Query Parameter**：`observer_id` (str, required)

```json
// Response
{
  "content": "- Alice is a software engineer\n- Prefers Python over JavaScript\n..."
}
```

### `PUT /v3/workspaces/{workspace_id}/peers/{peer_id}/card`

手動更新 peer card 內容。

---

### 4.4 Peer Context 端點

### `GET /v3/workspaces/{workspace_id}/peers/{peer_id}/context`

取得 peer 的完整上下文（訊息歷史 + 觀察記錄），適合注入 LLM 提示詞。

**Query Parameters**：
- `observer_id` (str)
- `session_id` (str, optional)
- `summary` (bool): 是否包含摘要

---

### `POST /v3/workspaces/{workspace_id}/peers/search` — 搜尋 Peers

```json
// Request Body
{
  "query": "alice",
  "observer_id": "agent-1",
  "top_k": 5
}
```

---

## 5. Session 端點

**路由前綴**：`/v3/workspaces/{workspace_id}/sessions`

### `POST /v3/workspaces/{workspace_id}/sessions` — 建立或取得 Session

```json
// Request Body
{
  "name": "conversation-001",
  "metadata": {},
  "configuration": {}
}

// Response 200/201
{
  "id": "session-nanoid-21chars",
  "name": "conversation-001",
  "workspace_id": "my-app",
  "is_active": true,
  "metadata": {},
  "configuration": {},
  "created_at": "2025-01-01T00:00:00Z"
}
```

### `POST /v3/workspaces/{workspace_id}/sessions/list` — 列出 Sessions

### `GET /v3/workspaces/{workspace_id}/sessions/{session_id}` — 取得 Session

### `PUT /v3/workspaces/{workspace_id}/sessions/{session_id}` — 更新 Session

### `DELETE /v3/workspaces/{workspace_id}/sessions/{session_id}` — 刪除 Session

（觸發背景異步刪除，包含 Messages、Documents 等）

---

### 5.1 Session Clone

### `POST /v3/workspaces/{workspace_id}/sessions/{session_id}/clone`

複製一個 session，產生新 session（相同訊息歷史，不同 ID）。

---

### 5.2 Session Context

### `GET /v3/workspaces/{workspace_id}/sessions/{session_id}/context`

取得 session 的對話上下文（包含訊息 + 摘要），適合注入 LLM 提示詞。

**Query Parameters**：
- `observer_id` (str, required)
- `summary` (bool): 包含摘要
- `recent` (bool): 只包含最近訊息
- `tokens` (int): 最大 token 數（預設 100000）

```json
// Response
{
  "messages": [
    {
      "peer_id": "alice",
      "content": "I like oatmeal",
      "created_at": "2025-01-01T00:00:00Z"
    }
  ],
  "summary": "Alice mentioned her breakfast preferences..."
}
```

---

### 5.3 Session Peers 管理

### `POST /v3/workspaces/{workspace_id}/sessions/{session_id}/peers/list`

列出 session 中的 peers。

### `POST /v3/workspaces/{workspace_id}/sessions/{session_id}/peers`

添加 peer 到 session。

```json
{
  "peer_id": "alice",
  "observe_me": true,
  "observe_others": false
}
```

### `DELETE /v3/workspaces/{workspace_id}/sessions/{session_id}/peers/{peer_id}`

從 session 移除 peer。

---

## 6. Message 端點

**路由前綴**：`/v3/workspaces/{workspace_id}/sessions/{session_id}/messages`

### `POST /v3/workspaces/{workspace_id}/sessions/{session_id}/messages` — 批次建立訊息

**最核心的端點**，觸發整個記憶處理管線。

```json
// Request Body
{
  "messages": [
    {
      "peer_id": "alice",
      "content": "I really love Python for data science work.",
      "metadata": {},
      "configuration": {}
    },
    {
      "peer_id": "agent-1",
      "content": "That's interesting! What libraries do you use?",
      "metadata": {}
    }
  ]
}

// Response 201
[
  {
    "id": "msg-nanoid-21chars",
    "peer_id": "alice",
    "session_id": "conversation-001",
    "workspace_id": "my-app",
    "content": "I really love Python for data science work.",
    "metadata": {},
    "token_count": 12,
    "seq_in_session": 1,
    "created_at": "2025-01-01T00:00:00Z"
  },
  { ... }
]
```

**批次限制**：最多 100 條訊息/請求（`src/routers/messages.py`）。

**背景觸發**：建立成功後，系統自動：
1. 為每條訊息生成嵌入向量
2. 將訊息加入記憶處理佇列（`QueueItem`）
3. 根據訊息數量判斷是否觸發摘要（每 20 條短摘，每 60 條長摘）

---

### `POST /v3/workspaces/{workspace_id}/sessions/{session_id}/messages/list`

列出 session 中的訊息（分頁）。

### `GET /v3/workspaces/{workspace_id}/sessions/{session_id}/messages/{message_id}`

取得單條訊息。

### `PUT /v3/workspaces/{workspace_id}/sessions/{session_id}/messages/{message_id}`

更新訊息元資料。

---

### 6.1 File Upload

### `POST /v3/workspaces/{workspace_id}/sessions/{session_id}/messages/upload`

上傳文件作為訊息（multipart/form-data）。

```
Form fields:
- peer_id (str, required): 上傳者 peer ID
- file (UploadFile, required): 文件（PDF 等），最大 5MB
- metadata (str, optional): JSON 字串
```

---

## 7. Conclusions（觀察記錄）端點

### `POST /v3/workspaces/{workspace_id}/conclusions/search`

搜尋 workspace 中的觀察記錄。

```json
// Request Body
{
  "query": "Python programming",
  "observer_id": "agent-1",
  "observed_id": "alice",
  "top_k": 10,
  "max_distance": 0.5
}

// Response
[
  {
    "id": "doc-nanoid",
    "content": "Alice is proficient in Python",
    "level": "explicit",
    "created_at": "...",
    "times_derived": 2
  }
]
```

---

## 8. Webhook 端點

### `POST /v3/workspaces/{workspace_id}/webhooks` — 建立 Webhook

```json
{
  "url": "https://your-server.com/webhook",
  "secret": "optional-hmac-secret"
}
```

**事件類型**：
- `queue.empty` — 佇列（representation 或 summary）處理完畢

**HMAC 簽名**：每個 webhook payload 包含 `X-Honcho-Signature` header。

### `POST /v3/workspaces/{workspace_id}/webhooks/list` — 列出 Webhooks

### `DELETE /v3/workspaces/{workspace_id}/webhooks/{webhook_id}` — 刪除 Webhook

---

## 9. 系統端點

### `GET /health`

健康檢查（Docker healthcheck 使用此端點）。

```json
{ "status": "ok" }
```

### `GET /metrics`

Prometheus 指標（若 `METRICS_ENABLED=true`）。

---

## 10. SDK 使用範例

### Python SDK

```python
from honcho import Honcho

client = Honcho(
    api_key="your-jwt-token",
    base_url="http://localhost:8000",  # 本地開發
)

# 建立 workspace 和 peer
workspace = client.workspaces.get_or_create("my-app")
peer = workspace.peers.get_or_create("alice")

# 建立 session 並加入訊息
session = workspace.sessions.get_or_create("conversation-001")
session.peers.add(peer)
session.messages.create([
    {"peer_id": "alice", "content": "I love Python!"}
])

# Dialectic 查詢（等待 Deriver 處理完後）
response = peer.chat(
    query="What programming languages does Alice like?",
    observer_id="agent-1",
    options={"reasoning": {"level": "low"}}
)
print(response.content)
```

### TypeScript SDK

```typescript
import { Honcho } from "@honcho-ai/sdk";

const client = new Honcho({
  apiKey: "your-jwt-token",
  baseUrl: "http://localhost:8000",
});

const workspace = await client.workspaces.getOrCreate({ name: "my-app" });
const peer = await workspace.peers.getOrCreate({ name: "alice" });
const session = await workspace.sessions.getOrCreate({ name: "chat-001" });

await session.messages.create({
  messages: [
    { peer_id: "alice", content: "I prefer functional programming" }
  ]
});

const response = await peer.chat({
  query: "What are Alice's programming preferences?",
  observer_id: "agent-1",
  options: { reasoning: { level: "medium" } }
});
console.log(response.content);
```

---

## 11. 完整端點清單

| Method | 路徑 | 說明 | 認證範圍 |
|--------|------|------|---------|
| POST | `/v3/workspaces` | 建立/取得 workspace | 任意 JWT |
| POST | `/v3/workspaces/list` | 列出所有 workspaces | admin |
| GET | `/v3/workspaces/{id}` | 取得 workspace | workspace |
| PUT | `/v3/workspaces/{id}` | 更新 workspace | workspace |
| DELETE | `/v3/workspaces/{id}` | 刪除 workspace | admin |
| POST | `/v3/workspaces/search` | 搜尋 workspaces | admin |
| POST | `/v3/workspaces/{id}/keys` | 建立 scoped JWT | admin |
| POST | `/v3/workspaces/{w}/peers` | 建立/取得 peer | workspace |
| POST | `/v3/workspaces/{w}/peers/list` | 列出 peers | workspace |
| GET | `/v3/workspaces/{w}/peers/{p}` | 取得 peer | workspace |
| PUT | `/v3/workspaces/{w}/peers/{p}` | 更新 peer | peer |
| DELETE | `/v3/workspaces/{w}/peers/{p}` | 刪除 peer | peer |
| POST | `/v3/workspaces/{w}/peers/{p}/chat` | Dialectic 查詢 | peer |
| GET | `/v3/workspaces/{w}/peers/{p}/representation` | 取得觀察記錄表示 | peer |
| GET | `/v3/workspaces/{w}/peers/{p}/card` | 取得 peer card | peer |
| PUT | `/v3/workspaces/{w}/peers/{p}/card` | 更新 peer card | peer |
| GET | `/v3/workspaces/{w}/peers/{p}/context` | 取得 peer 上下文 | peer |
| POST | `/v3/workspaces/{w}/peers/search` | 搜尋 peers | workspace |
| POST | `/v3/workspaces/{w}/sessions` | 建立/取得 session | workspace |
| POST | `/v3/workspaces/{w}/sessions/list` | 列出 sessions | workspace |
| GET | `/v3/workspaces/{w}/sessions/{s}` | 取得 session | session |
| PUT | `/v3/workspaces/{w}/sessions/{s}` | 更新 session | session |
| DELETE | `/v3/workspaces/{w}/sessions/{s}` | 刪除 session | session |
| POST | `/v3/workspaces/{w}/sessions/{s}/clone` | 複製 session | session |
| GET | `/v3/workspaces/{w}/sessions/{s}/context` | 取得 session 上下文 | session |
| POST | `/v3/workspaces/{w}/sessions/{s}/peers/list` | 列出 session peers | session |
| POST | `/v3/workspaces/{w}/sessions/{s}/peers` | 加入 peer 到 session | session |
| DELETE | `/v3/workspaces/{w}/sessions/{s}/peers/{p}` | 從 session 移除 peer | session |
| POST | `/v3/workspaces/{w}/sessions/{s}/messages` | 批次建立訊息 | session |
| POST | `/v3/workspaces/{w}/sessions/{s}/messages/list` | 列出訊息 | session |
| GET | `/v3/workspaces/{w}/sessions/{s}/messages/{m}` | 取得訊息 | session |
| PUT | `/v3/workspaces/{w}/sessions/{s}/messages/{m}` | 更新訊息 | session |
| POST | `/v3/workspaces/{w}/sessions/{s}/messages/upload` | 上傳文件 | session |
| POST | `/v3/workspaces/{w}/conclusions/search` | 搜尋觀察記錄 | workspace |
| POST | `/v3/workspaces/{w}/webhooks` | 建立 webhook | workspace |
| POST | `/v3/workspaces/{w}/webhooks/list` | 列出 webhooks | workspace |
| DELETE | `/v3/workspaces/{w}/webhooks/{wh}` | 刪除 webhook | workspace |
| GET | `/health` | 健康檢查 | 無 |
| GET | `/metrics` | Prometheus 指標 | 無 |

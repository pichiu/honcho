# Stage 2.5 - 外部整合

## 1. LLM Providers

### 1.1 Google Gemini（主要 provider）

**SDK**: `google-genai>=1.32.0`

**使用場景**：
- Deriver（預設）：`gemini-2.5-flash-lite`
- Dialectic minimal/low：`gemini-2.5-flash-lite`
- Summary：`gemini-2.5-flash`

**設定**：`LLM_GEMINI_API_KEY`

**客戶端初始化**（`src/utils/clients.py`）：
```python
from google import genai
client = genai.AsyncClient(api_key=settings.LLM.GEMINI_API_KEY)
```

**特殊處理**：
- `GEMINI_BLOCKED_FINISH_REASONS`：若 Gemini 因 SAFETY/RECITATION 等原因封鎖，觸發 backup provider 切換
- 工具格式：轉換為 Gemini FunctionDeclaration 格式

### 1.2 Anthropic Claude（推理 provider）

**SDK**: `sentry-sdk[anthropic]`（間接）+ `anthropic` 客戶端

**使用場景**：
- Dialectic medium/high/max：`claude-haiku-4-5`
- Dream orchestrator：`claude-sonnet-4-20250514`
- Dream specialists（deduction + induction）：`claude-haiku-4-5`

**設定**：`LLM_ANTHROPIC_API_KEY`

**特殊功能**：
- 擴展思維（Extended Thinking）：設定 `THINKING_BUDGET_TOKENS > 0` 時啟用
- `ThinkingBlock` 從回應中過濾，不傳遞到工具結果

### 1.3 OpenAI（嵌入 + 相容端點）

**SDK**: `openai>=1.99.7`

**使用場景**：
- 嵌入：`text-embedding-3-small`（預設，1536 維）
- 可透過 `custom` provider 路由到 OpenRouter/Together/Fireworks 等

**設定**：`LLM_OPENAI_API_KEY`

**嵌入客戶端** (`src/embedding_client.py`)：
- 批次處理，最大 `MAX_EMBEDDING_TOKENS_PER_REQUEST=300000` tokens/請求
- `MAX_EMBEDDING_TOKENS=8192` tokens/單條文字

### 1.4 Groq（可選）

**SDK**: `groq>=0.31.0`
**設定**：`LLM_GROQ_API_KEY`
**用途**：預設不使用，可配置為任何功能的 provider

---

## 2. 資料庫

### 2.1 PostgreSQL + pgvector

**驅動**: `psycopg[binary]>=3.1.19`（異步）

**設定**：
- `DB_CONNECTION_URI`：必須使用 `postgresql+psycopg://` 前綴
- `DB_POOL_SIZE=10`, `DB_MAX_OVERFLOW=20`
- `DB_POOL_RECYCLE=300` 秒

**pgvector 功能**：
- 向量欄位類型（Vector(1536)）
- HNSW 索引（m=16, ef_construction=64, cosine_ops）
- 自動安裝：`CREATE EXTENSION IF NOT EXISTS vector`

**失敗處理**：
- `pool_pre_ping=true`：連線前先 ping 確認存活
- `POOL_USE_LIFO=true`：優先使用最近使用的連線，減少 idle
- `prepare_threshold=None`：停用 prepared statements（pgBouncer 相容）

### 2.2 Redis（可選快取）

**SDK**: `redis>=7.0.0,<8.0.0`（+ `cashews[redis]==7.4.4`）

**使用場景**：
- 查詢結果快取（representation、peer card 等）
- 可透過 `CACHE_ENABLED=false` 完全停用

**失敗處理**：
- cashews Redis 錯誤日誌抑制為 CRITICAL（`src/main.py:73-74`）
- Redis 初始化失敗不中斷 API server

---

## 3. 觀測與監控

### 3.1 Sentry（錯誤追蹤）

**SDK**: `sentry-sdk[anthropic,fastapi,sqlalchemy]>=2.3.1`

**整合**：
- FastAPI/Starlette 中間件追蹤
- SQLAlchemy 查詢追蹤（`DB_TRACING=true` 時）
- Anthropic API 呼叫追蹤
- 自訂 `before_send`：過濾 HonchoException 和 ValidationError

**設定**：`SENTRY_ENABLED=true`, `SENTRY_DSN=...`

### 3.2 Langfuse（LLM 追蹤）

**SDK**: `langfuse>=3.3.2`

**使用場景**：LLM 呼叫的輸入/輸出/token 追蹤
**設定**：`LANGFUSE_HOST`, `LANGFUSE_PUBLIC_KEY`

**整合點** (`src/telemetry/logging.py`)：
```python
# conditional_observe 包裝 LLM 函式（若 Langfuse 未設定則 no-op）
```

### 3.3 Prometheus（指標）

**SDK**: `prometheus_client>=0.21.0`

**類型**：Pull-based（Prometheus 主動拉取）
**端點**：`GET /metrics`（API server）
**Deriver 端口**：9090（若啟用）
**設定**：`METRICS_ENABLED=true`

### 3.4 CloudEvents 遙測

**SDK**: `cloudevents>=1.12.0`

**類型**：Push-based（批次 HTTP POST 到遙測端點）
**設定**：`TELEMETRY_ENABLED=true`, `TELEMETRY_ENDPOINT=...`
**緩衝**：最多 `MAX_BUFFER_SIZE=10000` 個事件，每 `FLUSH_INTERVAL_SECONDS=1.0` 秒刷新

---

## 4. 部署平台

### 4.1 Fly.io

**設定檔**：`fly.toml`

**部署流程**：
```bash
flyctl launch --no-deploy
cat .env | flyctl secrets import
flyctl deploy
```

注意：`fly.toml` 不包含 PostgreSQL 設定，需獨立配置。

### 4.2 Docker

**基礎映像**：`python:3.13-slim-bookworm`
**uv 版本**：`ghcr.io/astral-sh/uv:0.9.24`

**Dockerfile 特性**：
- Multi-stage 依賴快取（分離 deps 和 code）
- 非 root 用戶（`app:app`）
- Healthcheck：`GET /health`
- 暴露端口：8000

**Docker compose 模板**：`docker-compose.yml.example`（含 PostgreSQL）

---

## 5. 嵌入服務

### 5.1 嵌入客戶端（src/embedding_client.py）

支援三種 embedding provider（`LLM_EMBEDDING_PROVIDER`）：
- `openai`：`text-embedding-3-small`（預設，1536 維）
- `gemini`：Google Gemini 嵌入
- `openrouter`：透過 OpenRouter 路由（OpenAI 相容格式）

**批次處理**：
- 最大 `MAX_EMBEDDING_TOKENS_PER_REQUEST=300000` tokens
- 超過自動分批

**失敗處理**：
- 嵌入失敗記錄錯誤，訊息仍儲存（embedding 為 nullable）
- reconciler 後續同步處理失敗的嵌入（sync_state="pending"）

---

## 6. 向量同步（Reconciler）

### 6.1 背景同步機制

當 vector store 是 Turbopuffer 或 LanceDB 時，需要從 pgvector（PostgreSQL）同步：

**ReconcilerScheduler** (`src/reconciler/scheduler.py`)：
- 每 `VECTOR_STORE_RECONCILIATION_INTERVAL_SECONDS=300` 秒觸發
- 佇列化 `reconciler` 任務

**sync_vectors** (`src/reconciler/sync_vectors.py`)：
- 查找 `sync_state="pending"` 的 documents 和 message_embeddings
- 批次 upsert 到外部向量儲存
- 更新 `sync_state="synced"`
- 清理軟刪除的記錄（`deleted_at IS NOT NULL`）

**失敗處理**：
- `sync_attempts` 計數器追蹤重試次數
- 使用 `upsert_with_retry`（tenacity 指數退避）

---

## 7. 第三方 SDK 整合（範例）

`examples/` 目錄包含以下框架整合範例：

| 整合 | 目錄 | 說明 |
|------|------|------|
| CrewAI | `examples/crewai/python/` | AI agent 框架整合 |
| LangGraph | `examples/langgraph/python/` + `.../typescript/` | LangGraph workflow 整合 |
| Gmail | `examples/gmail/` | Email 處理整合 |
| Granola | `examples/granola/` | 會議記錄整合 |
| N8N | `examples/n8n/` | Workflow 自動化整合 |

---

## 8. 失敗處理策略

| 整合 | 失敗處理方式 |
|------|------------|
| LLM API | tenacity 重試（指數退避）+ backup provider |
| 嵌入 API | 記錄錯誤 + 訊息仍儲存（embedding=NULL），reconciler 重試 |
| Redis | 降級為無快取模式（graceful degradation） |
| Sentry | 若未設定，靜默跳過 |
| Langfuse | 若未設定，靜默跳過（conditional_observe no-op） |
| Webhook 遞送 | 佇列重試（最多 3 次）|
| 向量同步 | sync_attempts 追蹤，reconciler 週期性重試 |
| 資料庫 | pool_pre_ping + 連線重試 |

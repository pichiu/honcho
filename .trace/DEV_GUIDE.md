# DEV_GUIDE.md — 開發者指南

> 基於 Honcho v3.0.6-rc | 最後更新：2026-05-08（增量 trace 5b6bd59→a4ae372）

---

## 1. 環境準備

### 1.1 系統需求

| 工具 | 最低版本 | 用途 |
|------|---------|------|
| Python | 3.10+ | 後端語言 |
| uv | 0.5.0+ | Python 套件管理器 |
| PostgreSQL | 14+ (含 pgvector 擴展) | 主要資料庫 |
| Redis | 7.0+ | 查詢快取（可選）|
| Docker | 任意版本 | 本地 DB 環境（選擇性）|
| Bun | 最新 | TypeScript SDK 測試（若需要）|

### 1.2 取得 pgvector

pgvector 需獨立安裝：

```bash
# macOS（Homebrew）
brew install pgvector

# Ubuntu/Debian
sudo apt-get install postgresql-14-pgvector

# Docker（推薦本地開發）
docker run -d \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=postgres \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```

---

## 2. 快速啟動

### 2.1 安裝依賴

```bash
# clone 後進入專案目錄
uv sync          # 安裝所有依賴（根據 pyproject.toml）
```

### 2.2 設定環境

```bash
cp .env.template .env          # 建立環境設定檔
cp config.toml.example config.toml  # 建立 TOML 設定檔

# 生成 JWT secret（若需要認證）
python scripts/generate_jwt_secret.py
```

**必填的 `.env` 設定**：

```env
# 資料庫（必須）
DB_CONNECTION_URI=postgresql+psycopg://postgres:postgres@localhost:5432/postgres

# LLM（至少選一個）
LLM_GEMINI_API_KEY=your-google-api-key        # Deriver + 低推理 Dialectic
LLM_ANTHROPIC_API_KEY=your-anthropic-key      # 中高推理 Dialectic + Dream

# 嵌入向量（必須）
LLM_OPENAI_API_KEY=your-openai-key            # 嵌入（預設）
# 或使用 OpenRouter（免費額度）
LLM_OPENAI_COMPATIBLE_BASE_URL=https://openrouter.ai/api/v1
LLM_OPENAI_COMPATIBLE_API_KEY=your-openrouter-key
LLM_EMBEDDING_PROVIDER=openrouter

# 認證（開發環境可關閉）
AUTH_USE_AUTH=false
```

### 2.3 資料庫初始化

```bash
uv run alembic upgrade head    # 執行所有 migrations，建立資料表
```

### 2.4 啟動服務

```bash
# API Server（開發模式，熱重載）
uv run fastapi dev src/main.py

# 在另一個終端啟動 Background Worker
uv run python -m src.deriver
```

---

## 3. Docker 啟動（推薦新手）

```bash
cp docker-compose.yml.example docker-compose.yml

# 只啟動資料庫和 Redis
docker compose up -d database redis

# 啟動完整服務（API + Deriver + DB）
docker compose up
```

**Docker compose 包含**：
- PostgreSQL（含 pgvector）
- Redis
- Honcho API server（port 8000）
- Deriver background worker

---

## 4. 設定詳解

### 4.1 設定優先順序（高→低）

```
1. 程式碼 init 參數（幾乎不用）
2. 環境變數（$ export DB_CONNECTION_URI=...）
3. .env 檔案
4. config.toml 檔案
5. 程式碼預設值
```

### 4.2 重要設定項速查

**開發環境推薦設定**：

```toml
# config.toml
[app]
LOG_LEVEL = "DEBUG"            # 詳細日誌

[auth]
USE_AUTH = false               # 停用認證（開發環境）

[deriver]
ENABLED = true
WORKERS = 1
POLLING_SLEEP_INTERVAL_SECONDS = 1.0

[dream]
ENABLED = false                # 開發時可停用（減少 LLM 費用）
DOCUMENT_THRESHOLD = 50        # 觸發 dream 的文件數量

[db]
SQL_DEBUG = false              # 改為 true 可看到所有 SQL 查詢
POOL_SIZE = 5                  # 本地開發不需要大型連線池
```

**LLM 最低成本設定（開發用）**：

```toml
[deriver]
PROVIDER = "google"
MODEL = "gemini-2.5-flash-lite"  # 最便宜

[dialectic.levels.minimal]
PROVIDER = "google"
MODEL = "gemini-2.5-flash-lite"

[dialectic.levels.low]
PROVIDER = "google"
MODEL = "gemini-2.5-flash-lite"
```

### 4.3 Feature Flags 快速開關

| Flag | 預設 | 建議開發設定 |
|------|------|------------|
| `AUTH_USE_AUTH` | false | false（不需 JWT）|
| `CACHE_ENABLED` | false | false（不需 Redis）|
| `DERIVER_ENABLED` | true | true（核心功能）|
| `DREAM_ENABLED` | true | false（節省費用）|
| `SUMMARY_ENABLED` | true | true |
| `DB_SQL_DEBUG` | false | true（除錯 SQL）|
| `METRICS_ENABLED` | false | false |
| `SENTRY_ENABLED` | false | false |
| `TELEMETRY_ENABLED` | false | false |

---

## 5. 開發工作流程

### 5.1 程式碼品質工具

```bash
# Linting（ruff）
uv run ruff check src/          # 檢查 lint 問題
uv run ruff check src/ --fix    # 自動修復

# 格式化
uv run ruff format src/         # 格式化程式碼

# 靜態型別檢查
uv run basedpyright             # 型別分析（strict mode）
```

**提交前建議執行**：

```bash
uv run ruff format src/ && uv run ruff check src/ && uv run basedpyright
```

### 5.2 測試

```bash
# 執行所有測試
uv run pytest tests/

# 執行特定測試檔案
uv run pytest tests/test_messages.py

# 執行特定測試函式
uv run pytest tests/test_messages.py::test_create_message

# TypeScript SDK 測試（不要直接執行 bun test！）
uv run pytest tests/ -k typescript
# 注意：TypeScript SDK 測試需要完整的 Honcho server 環境，
# 由 pytest fixture 自動管理（conftest.py）
```

### 5.3 資料庫 Migration 工作流程

```bash
# 新增 migration（修改 models.py 後）
uv run alembic revision --autogenerate -m "add column X to table Y"

# 查看 migration 歷史
uv run alembic history

# 升級到最新
uv run alembic upgrade head

# 降級一個版本
uv run alembic downgrade -1

# 降級到特定版本
uv run alembic downgrade <revision_id>
```

**Migration 注意事項**：
- 修改 `src/models.py` 後**必須**建立對應的 migration
- 建立 migration 前確認 `alembic.ini` 的 `script_location` 正確指向 `migrations/`
- Alembic 設定在 `migrations/env.py` 中引入 `Base.metadata`

### 5.4 Git 分支規範（CONTRIBUTING.md）

遵循 [Conventional Commits](https://www.conventionalcommits.org/)：

```bash
# 分支命名
feat/add-streaming-support     # 新功能
fix/memory-leak-in-deriver     # Bug 修復
chore/update-dependencies      # 維護工作
docs/update-api-reference      # 文件更新

# Commit 訊息格式
feat: add support for custom embedding providers
fix: resolve race condition in queue_manager
refactor: extract config helpers to separate module
```

---

## 6. honcho-cli 使用指南

> <!-- 新增於 2026-05-08, 5b6bd59→a4ae372 -->

`honcho-cli` 是 v3.0.6 新增的命令列工具，用於終端機操作 Honcho workspace 以及除錯記憶狀態。

### 6.1 安裝

```bash
uv tool install honcho-cli
```

### 6.2 初始設定

```bash
honcho init     # 設定 API key + Honcho URL
                # 寫入 ~/.honcho/config.json
honcho doctor   # 健康檢查（config、連線、workspace、peer、佇列）
```

設定檔路徑：`~/.honcho/config.json`
```json
{
  "apiKey": "your-jwt-token",
  "environmentUrl": "http://localhost:8000"
}
```

### 6.3 常用除錯指令

```bash
# 查看佇列狀態（確認 Deriver 是否正在處理）
honcho workspace queue-status -w <workspace_name>
honcho workspace queue-status -w <ws> --observer <agent>  # 過濾特定觀察者

# 查看 peer 的記憶表示
honcho peer representation -w <ws> -p <peer_id> --observer <obs>

# 查看 peer card
honcho peer card -w <ws> -p <peer_id>

# 即時 Dialectic 查詢
honcho peer chat -w <ws> -p <peer_id> --observer <obs> "<query>"

# 列出近期觀察記錄
honcho conclusion search -w <ws> "<search_query>"

# 查看 session 對話歷史
honcho session context -w <ws> -s <session_id>
```

### 6.4 範圍 flags

所有命令支援以下範圍 flags（或環境變數）：

| Flag | 環境變數 | 說明 |
|------|---------|------|
| `-w` / `--workspace` | `HONCHO_WORKSPACE` | Workspace 名稱 |
| `-p` / `--peer` | `HONCHO_PEER` | Peer 名稱 |
| `-s` / `--session` | `HONCHO_SESSION` | Session 名稱 |

<!-- 新增結束 -->

---

## 7. 新增功能指南

### 7.1 新增 API 端點

1. 在 `src/routers/` 選擇對應的 router 檔案（或建立新的）
2. 在 `src/crud/` 新增 DB 操作函式
3. 在 `src/schemas/api.py` 定義 Pydantic 輸入輸出模型
4. 在 `src/main.py` 掛載 router（若是新 router）
5. 為新端點添加測試

```python
# Router 模板
@router.post(
    "/{resource_id}/action",
    response_model=schemas.ActionResponse,
    dependencies=[Depends(require_auth(workspace_name="workspace_id"))],
)
async def do_action(
    workspace_id: str = Path(...),
    resource_id: str = Path(...),
    body: schemas.ActionRequest = Body(...),
    db: AsyncSession = db,
) -> schemas.ActionResponse:
    """端點說明。"""
    result = await crud.perform_action(db, workspace_id, resource_id, body)
    return result
```

### 6.2 新增 DB Model

1. 在 `src/models.py` 添加新 Model（繼承 `Base`）
2. 在 `src/crud/` 建立對應的 CRUD 函式
3. 執行 `uv run alembic revision --autogenerate -m "add ..."` 建立 migration

### 6.3 新增 Webhook 事件

1. 在 `src/webhooks/events.py` 定義新事件類別
2. 在業務邏輯中呼叫 `publish_webhook_event(event)`
3. Webhook delivery 自動處理 HTTP 遞送和 HMAC 簽名

### 6.4 新增向量儲存後端

1. 在 `src/vector_store/my_store.py` 繼承 `VectorStore` ABC
2. 實作所有抽象方法（`upsert_many`, `query`, `delete_many`, `delete_namespace`, `close`）
3. 在 `_create_store_by_type()` 新增分支（`src/vector_store/__init__.py:198-209`）
4. 在 `VectorStoreSettings.TYPE` 的 `Literal` 中新增類型名稱（`src/config.py:581`）

---

## 8. 除錯技巧

### 7.1 查看資料庫查詢

```env
DB_SQL_DEBUG=true
```

啟用後所有 SQL 查詢都會輸出到 log。

### 7.2 LLM 呼叫追蹤（Langfuse）

```env
LANGFUSE_HOST=http://localhost:3000
LANGFUSE_PUBLIC_KEY=your-key
LANGFUSE_SECRET_KEY=your-secret
```

Langfuse 提供 LLM 呼叫的輸入/輸出/token 使用量追蹤，適合除錯 prompt 問題。

### 7.3 調整日誌等級

```env
LOG_LEVEL=DEBUG   # 顯示所有日誌（包含 deriver 處理細節）
```

### 7.4 手動測試 Deriver

可以透過直接 insert QueueItem 觸發 Deriver 處理：

```python
# 或透過 API 建立訊息，觀察 Deriver log
import httpx

client = httpx.Client(base_url="http://localhost:8000")
response = client.post(
    "/v3/workspaces/my-app/sessions/test-session/messages",
    headers={"Authorization": "Bearer <jwt>"},
    json={
        "messages": [
            {"peer_id": "alice", "content": "Test message for deriver"}
        ]
    }
)
```

然後觀察 Deriver worker 的 log 輸出。

### 7.5 直接查看 Queue 狀態

```sql
-- 查看未處理的佇列項目
SELECT task_type, work_unit_key, processed, created_at 
FROM queue 
WHERE processed = false 
ORDER BY created_at DESC 
LIMIT 20;

-- 查看活躍的 worker session
SELECT * FROM active_queue_sessions;
```

### 7.6 重置特定 Peer 的 Dream

```sql
-- 強制觸發 Dream（插入 dream 任務）
INSERT INTO queue (task_type, work_unit_key, payload, workspace_name)
VALUES (
  'dream',
  'dream:{workspace}:{observer}:{observed}',
  '{"workspace_name": "...", "observer": "...", "observed": "...", "dream_type": "scheduled"}',
  'workspace_name'
);
```

### 7.7 Deriver 卡住處理

若 Deriver 看起來沒有在處理佇列，檢查：

```sql
-- 是否有 active_queue_sessions 卡住
SELECT * FROM active_queue_sessions 
WHERE last_updated < NOW() - INTERVAL '10 minutes';

-- 若有卡住的，可手動清除（謹慎操作）
DELETE FROM active_queue_sessions 
WHERE last_updated < NOW() - INTERVAL '10 minutes';
```

---

## 9. 部署指南

### 8.1 Fly.io 部署

```bash
# 初始化
flyctl launch --no-deploy

# 設定 secrets
cat .env | flyctl secrets import

# 部署
flyctl deploy
```

**注意**：`fly.toml` 不包含 PostgreSQL 設定，需要單獨配置 Fly Postgres 或外部 DB。

### 8.2 自架 Docker

```bash
# 建置映像
docker build -t honcho:latest .

# 啟動（含外部 DB）
docker run -d \
  -e DB_CONNECTION_URI=postgresql+psycopg://... \
  -e LLM_GEMINI_API_KEY=... \
  -p 8000:8000 \
  honcho:latest
```

**Dockerfile 特性**：
- 基礎映像：`python:3.13-slim-bookworm`
- 非 root 用戶運行（安全性）
- Healthcheck：`GET /health`
- 開放端口：8000

### 8.3 連線池調整（高負載）

```env
DB_POOL_SIZE=20           # 增加連線池大小
DB_MAX_OVERFLOW=40        # 允許更多溢出連線
DB_POOL_TIMEOUT=30        # 取得連線等待上限
DB_POOL_RECYCLE=300       # 定期回收連線（防止 DB server 斷線）
```

### 8.4 Redis 快取（提升效能）

```env
CACHE_ENABLED=true
CACHE_REDIS_URL=redis://localhost:6379
CACHE_DEFAULT_TTL_SECONDS=3600
```

啟用 Redis 快取後，`peer_card` 和 `representation` 等頻繁查詢會被快取。

---

## 10. 常見問題

### Q: 訊息建立了，但 Deriver 沒有處理？

**檢查**：
1. Deriver worker 是否在執行（`uv run python -m src.deriver`）
2. `DERIVER_ENABLED=true`
3. LLM API key 是否有效
4. 查看 Deriver worker 的 log 輸出是否有錯誤

### Q: `DB_CONNECTION_URI` 格式錯誤？

必須使用 `postgresql+psycopg://`（psycopg 3），而非 `postgresql://`（psycopg 2）。

```env
# 正確
DB_CONNECTION_URI=postgresql+psycopg://user:pass@localhost:5432/dbname

# 錯誤
DB_CONNECTION_URI=postgresql://user:pass@localhost:5432/dbname
```

### Q: TypeScript SDK 測試失敗？

不要直接執行 `bun test`。必須使用：

```bash
uv run pytest tests/ -k typescript
```

pytest fixture 會自動啟動所需的 Honcho server 環境。

### Q: Dreamer 不工作？

- 預設 `DREAM_ENABLED=true`，但 Dream 只在文件數量 ≥ `DOCUMENT_THRESHOLD(50)` 且空閒 60 分鐘後才觸發
- 開發時可調低閾值：`DREAM_DOCUMENT_THRESHOLD=5`、`DREAM_IDLE_TIMEOUT_MINUTES=1`

### Q: pgvector 擴展安裝問題？

資料庫必須先安裝 pgvector，Alembic migration 才能成功：

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

使用官方 `pgvector/pgvector` Docker 映像時已內置此擴展。

### Q: 如何使用 OpenRouter 代替 OpenAI 進行嵌入？

```env
LLM_OPENAI_COMPATIBLE_BASE_URL=https://openrouter.ai/api/v1
LLM_OPENAI_COMPATIBLE_API_KEY=your-openrouter-key
LLM_EMBEDDING_PROVIDER=openrouter
```

### Q: 如何使用自架 Ollama / vLLM？

<!-- 更新於 2026-05-08, 5b6bd59→a4ae372：ModelTransport 不再有 vllm/custom -->

v3.0.6+ 的 `ModelTransport` 只有三個值：`"anthropic"`, `"openai"`, `"gemini"`。
自架模型需透過 `ModelOverrideSettings`（`api_key` + `base_url`）覆蓋：

```toml
# config.toml（以 Deriver 為例）
[deriver.model_config.overrides]
base_url = "http://localhost:11434/v1"   # Ollama OpenAI-compatible endpoint
api_key = "placeholder"

[deriver.model_config]
transport = "openai"                     # 使用 OpenAI 相容格式
model = "llama3"
```

對應環境變數：
```env
DERIVER_MODEL_CONFIG__TRANSPORT=openai
DERIVER_MODEL_CONFIG__MODEL=llama3
DERIVER_MODEL_CONFIG__OVERRIDES__BASE_URL=http://localhost:11434/v1
DERIVER_MODEL_CONFIG__OVERRIDES__API_KEY=placeholder
```

---

## 11. 相關資源

| 資源 | 說明 |
|------|------|
| [docs.honcho.dev](https://docs.honcho.dev) | 官方開發者文件（v3）|
| [GitHub](https://github.com/plastic-labs/honcho) | 原始碼倉庫 |
| [Discord](https://discord.gg/honcho) | 社群支援 |
| [app.honcho.dev](https://app.honcho.dev) | 雲端 SaaS 版本 |
| [evals.honcho.dev](https://evals.honcho.dev) | 官方基準評估 |

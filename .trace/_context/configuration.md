# Stage 2.6 - 設定與環境

## 1. 設定載入機制

### 1.1 優先順序（高→低）

```
1. init 參數（程式碼直接傳入）
2. 環境變數（環境層或 Fly.io secrets）
3. .env 檔案（本地開發用）
4. config.toml 檔案（基礎設定）
5. 預設值（程式碼內建）
```

### 1.2 技術實現（src/config.py）

使用 `pydantic-settings` 的 `HonchoSettings` 基礎類別，自訂 `settings_customise_sources()` 定義優先順序：

```python
class HonchoSettings(BaseSettings):
    @classmethod
    def settings_customise_sources(cls, ...):
        return (
            init_settings,           # 最高優先
            env_settings,            # 環境變數
            dotenv_settings,         # .env 檔案
            TomlConfigSettingsSource(settings_cls),  # config.toml
            file_secret_settings,    # 最低優先
        )
```

### 1.3 TOML 設定到環境變數的映射

| TOML Section | 環境變數前綴 |
|-------------|------------|
| `[app]` | 無前綴（直接 `LOG_LEVEL=`）|
| `[db]` | `DB_` |
| `[auth]` | `AUTH_` |
| `[llm]` | `LLM_` |
| `[deriver]` | `DERIVER_` |
| `[dialectic]` | `DIALECTIC_` |
| `[dialectic.levels.minimal]` | `DIALECTIC_LEVELS__minimal__` |
| `[summary]` | `SUMMARY_` |
| `[dream]` | `DREAM_` |
| `[dream.surprisal]` | `DREAM_SURPRISAL__` |
| `[cache]` | `CACHE_` |
| `[vector_store]` | `VECTOR_STORE_` |
| `[metrics]` | `METRICS_` |
| `[telemetry]` | `TELEMETRY_` |
| `[sentry]` | `SENTRY_` |
| `[webhook]` | `WEBHOOK_` |
| `[peer_card]` | `PEER_CARD_` |

---

## 2. 必要設定

### 2.1 最低必要設定

```env
# 資料庫（必須）
DB_CONNECTION_URI=postgresql+psycopg://postgres:postgres@localhost:5432/postgres

# LLM（至少一個）
LLM_GEMINI_API_KEY=          # Deriver + 低推理 Dialectic + Summary
LLM_ANTHROPIC_API_KEY=       # 中高推理 Dialectic + Dream

# 或使用 OpenAI 相容端點（OpenRouter 等）
LLM_OPENAI_COMPATIBLE_BASE_URL=https://openrouter.ai/api/v1
LLM_OPENAI_COMPATIBLE_API_KEY=your-key

# 嵌入（預設使用 OpenAI）
LLM_OPENAI_API_KEY=          # 或改用 openrouter
LLM_EMBEDDING_PROVIDER=openrouter   # 若使用 openrouter
```

### 2.2 認證設定

```env
AUTH_USE_AUTH=false           # 預設關閉（適合開發）
AUTH_JWT_SECRET=              # 若 USE_AUTH=true 必須設定
                              # 生成：python scripts/generate_jwt_secret.py
```

---

## 3. 主要設定群組詳解

### 3.1 應用程式級設定（AppSettings）

| 設定 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `LOG_LEVEL` | str | "INFO" | DEBUG/INFO/WARNING/ERROR/CRITICAL |
| `SESSION_OBSERVERS_LIMIT` | int | 10 | 每個 session 最多觀察者數量 |
| `MAX_FILE_SIZE` | int | 5242880 | 上傳檔案大小上限（5MB）|
| `GET_CONTEXT_MAX_TOKENS` | int | 100000 | get_context 最大 token 數 |
| `MAX_MESSAGE_SIZE` | int | 25000 | 訊息內容最大字元數 |
| `EMBED_MESSAGES` | bool | true | 是否為訊息生成嵌入向量 |
| `NAMESPACE` | str | "honcho" | 向量命名空間前綴 |

### 3.2 資料庫設定（DBSettings）

| 設定 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `DB_CONNECTION_URI` | str | - | PostgreSQL 連接字串 |
| `DB_SCHEMA` | str | "public" | PostgreSQL schema |
| `DB_POOL_SIZE` | int | 10 | 連線池大小 |
| `DB_MAX_OVERFLOW` | int | 20 | 最大溢出連線數 |
| `DB_POOL_TIMEOUT` | int | 30 | 取得連線等待秒數 |
| `DB_POOL_RECYCLE` | int | 300 | 連線回收秒數 |
| `DB_POOL_USE_LIFO` | bool | true | 使用 LIFO 連線順序 |
| `DB_SQL_DEBUG` | bool | false | 輸出 SQL 日誌 |

### 3.3 LLM 設定（LLMSettings）

| 設定 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `LLM_GEMINI_API_KEY` | str | None | Google Gemini API 金鑰 |
| `LLM_ANTHROPIC_API_KEY` | str | None | Anthropic API 金鑰 |
| `LLM_OPENAI_API_KEY` | str | None | OpenAI API 金鑰（主要用於嵌入）|
| `LLM_OPENAI_COMPATIBLE_BASE_URL` | str | None | OpenAI 相容端點 URL |
| `LLM_OPENAI_COMPATIBLE_API_KEY` | str | None | 相容端點 API 金鑰 |
| `LLM_VLLM_BASE_URL` | str | None | vLLM 端點 URL |
| `LLM_EMBEDDING_PROVIDER` | str | "openai" | 嵌入 provider（openai/gemini/openrouter）|
| `LLM_DEFAULT_MAX_TOKENS` | int | 2500 | 預設最大輸出 token 數 |
| `LLM_MAX_TOOL_OUTPUT_CHARS` | int | 10000 | 工具輸出最大字元數（截斷用）|

### 3.4 Deriver 設定（DeriverSettings）

| 設定 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `DERIVER_ENABLED` | bool | true | 是否啟用 deriver |
| `DERIVER_WORKERS` | int | 1 | 並發 worker 數量 |
| `DERIVER_POLLING_SLEEP_INTERVAL_SECONDS` | float | 1.0 | 輪詢間隔（秒）|
| `DERIVER_STALE_SESSION_TIMEOUT_MINUTES` | int | 5 | 過期 session 超時（分鐘）|
| `DERIVER_PROVIDER` | str | "google" | LLM provider |
| `DERIVER_MODEL` | str | "gemini-2.5-flash-lite" | LLM 模型 |
| `DERIVER_DEDUPLICATE` | bool | true | 是否去重觀察記錄 |
| `DERIVER_MAX_OUTPUT_TOKENS` | int | 4096 | 最大輸出 token |
| `DERIVER_THINKING_BUDGET_TOKENS` | int | 1024 | 思考預算（> 0 啟用）|
| `DERIVER_MAX_INPUT_TOKENS` | int | 23000 | 最大輸入 token |
| `DERIVER_REPRESENTATION_BATCH_MAX_TOKENS` | int | 1024 | 批次處理 token 閾值 |
| `DERIVER_FLUSH_ENABLED` | bool | false | 是否繞過批次 token 閾值 |

### 3.5 Dialectic 設定（DialecticSettings）

| 設定 | 說明 |
|------|------|
| `DIALECTIC_MAX_OUTPUT_TOKENS` | 最大輸出 token（8192）|
| `DIALECTIC_MAX_INPUT_TOKENS` | 最大輸入 token（100000）|
| `DIALECTIC_HISTORY_TOKEN_LIMIT` | 歷史記錄 token 上限（8192）|
| `DIALECTIC_SESSION_HISTORY_MAX_TOKENS` | Session 歷史注入 token 上限（4096）|

每個推理等級獨立設定（以 `DIALECTIC_LEVELS__{level}__` 前綴）：
- `PROVIDER` / `MODEL` / `BACKUP_PROVIDER` / `BACKUP_MODEL`
- `THINKING_BUDGET_TOKENS`
- `MAX_TOOL_ITERATIONS`
- `MAX_OUTPUT_TOKENS`（可選，覆蓋全域）

### 3.6 Dream 設定（DreamSettings）

| 設定 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `DREAM_ENABLED` | bool | true | 是否啟用 dream |
| `DREAM_DOCUMENT_THRESHOLD` | int | 50 | 觸發 dream 的文件計數閾值 |
| `DREAM_IDLE_TIMEOUT_MINUTES` | int | 60 | 空閒等待時間（分鐘）|
| `DREAM_MIN_HOURS_BETWEEN_DREAMS` | int | 8 | 兩次 dream 最小間隔（小時）|
| `DREAM_PROVIDER` | str | "anthropic" | LLM provider |
| `DREAM_MODEL` | str | "claude-sonnet-4-20250514" | 主要 Dream 模型 |
| `DREAM_DEDUCTION_MODEL` | str | "claude-haiku-4-5" | 演繹 specialist 模型 |
| `DREAM_INDUCTION_MODEL` | str | "claude-haiku-4-5" | 歸納 specialist 模型 |
| `DREAM_THINKING_BUDGET_TOKENS` | int | 8192 | 思考預算 |
| `DREAM_MAX_TOOL_ITERATIONS` | int | 20 | 最大工具迭代次數 |

### 3.7 向量儲存設定（VectorStoreSettings）

| 設定 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `VECTOR_STORE_TYPE` | str | "pgvector" | pgvector/turbopuffer/lancedb |
| `VECTOR_STORE_MIGRATED` | bool | false | 是否已完成遷移 |
| `VECTOR_STORE_NAMESPACE` | str | "honcho" | 命名空間前綴 |
| `VECTOR_STORE_DIMENSIONS` | int | 1536 | 嵌入向量維度 |
| `VECTOR_STORE_TURBOPUFFER_API_KEY` | str | None | Turbopuffer API 金鑰 |
| `VECTOR_STORE_LANCEDB_PATH` | str | "./lancedb_data" | LanceDB 本地路徑 |
| `VECTOR_STORE_RECONCILIATION_INTERVAL_SECONDS` | int | 300 | 同步間隔（秒）|

---

## 4. Feature Flags

以下設定實際上是功能開關（Feature Flags）：

| Flag | 預設 | 效果 |
|------|------|------|
| `AUTH_USE_AUTH` | false | 停用則所有請求視為管理員（admin JWT）|
| `SENTRY_ENABLED` | false | 停用 Sentry 錯誤追蹤 |
| `METRICS_ENABLED` | false | 停用 Prometheus 指標 |
| `TELEMETRY_ENABLED` | false | 停用 CloudEvents 遙測 |
| `CACHE_ENABLED` | false | 停用 Redis 快取 |
| `EMBED_MESSAGES` | true | 停用則訊息不生成嵌入向量 |
| `DERIVER_ENABLED` | true | 停用則記憶擷取完全關閉 |
| `DREAM_ENABLED` | true | 停用則記憶整合關閉 |
| `SUMMARY_ENABLED` | true | 停用則對話摘要關閉 |
| `PEER_CARD_ENABLED` | true | 停用則不使用 peer card |
| `DERIVER_FLUSH_ENABLED` | false | 啟用則立即處理（不等待 token 批次）|
| `DB_SQL_DEBUG` | false | 啟用則輸出所有 SQL 查詢 |

---

## 5. Secrets 管理

**生產環境建議**：
- Fly.io：`flyctl secrets import`
- Docker：`--env-file` 或 secrets volume
- Kubernetes：Kubernetes Secrets

**本地開發**：
- 複製 `.env.template` 為 `.env`，填入所需值
- `.env` 已在 `.gitignore` 中

**JWT 金鑰生成**：
```bash
python scripts/generate_jwt_secret.py
```

---

## 6. 命名空間傳播

`AppSettings.propagate_namespace()` 在初始化後執行，將頂層 `NAMESPACE` 自動傳播到：
- `CACHE.NAMESPACE`（若未明確設定）
- `VECTOR_STORE.NAMESPACE`（若未明確設定）
- `TELEMETRY.NAMESPACE`（若未明確設定）
- `METRICS.NAMESPACE`（若未明確設定）

這允許用一個設定控制所有子系統的命名空間前綴。

---

## 7. 設定驗證

`pydantic-settings` 的 `model_validator` 確保：
- `AUTH_USE_AUTH=true` 時 `JWT_SECRET` 必須設定
- `VECTOR_STORE_TYPE=turbopuffer` 時 `TURBOPUFFER_API_KEY` 必須設定
- `BACKUP_PROVIDER` 和 `BACKUP_MODEL` 必須同時設定或同時為 None
- `DIALECTIC_MAX_OUTPUT_TOKENS` > 各等級的 `THINKING_BUDGET_TOKENS`
- `DREAM_MAX_OUTPUT_TOKENS` > `DREAM_THINKING_BUDGET_TOKENS`
- Anthropic provider 的 `THINKING_BUDGET_TOKENS` 若 > 0 則必須 ≥ 1024
- 所有五個 Dialectic 推理等級都必須有設定

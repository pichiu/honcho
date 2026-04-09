# Stage 2.1 - Entry Points

## 1. API Server Entry Point

### 1.1 主要啟動方式

**開發模式**：
```bash
uv run fastapi dev src/main.py  # 開發模式（熱重載）
```

**生產模式**（Dockerfile CMD）：
```bash
fastapi run --host 0.0.0.0 src/main.py
```

### 1.2 FastAPI 應用程式工廠 (`src/main.py`)

```python
# src/main.py:148-168
app = FastAPI(
    lifespan=lifespan,
    servers=[...],
    title="Honcho API",
    version="3.0.5",
)
```

**lifespan 流程** (`src/main.py:124-146`)：

```
啟動：
  1. initialize_telemetry_async()    ← CloudEvents 遙測系統初始化
  2. init_cache()                    ← Redis 快取初始化（失敗不中斷）
  3. yield（應用程式運行中）

關閉：
  4. close_external_vector_store()   ← 關閉外部向量儲存
  5. close_cache()                   ← 關閉 Redis 連線
  6. engine.dispose()                ← 關閉 DB 連線池
  7. shutdown_telemetry()            ← 刷新 CloudEvents 緩衝
```

### 1.3 Middleware 與插件初始化 (`src/main.py:170-196`)

```python
app.add_middleware(CORSMiddleware, ...)        # 允許的來源：localhost, api.honcho.dev
add_pagination(app)                            # fastapi-pagination 全域分頁
app.include_router(workspaces.router, prefix="/v3")
app.include_router(peers.router, prefix="/v3")
app.include_router(sessions.router, prefix="/v3")
app.include_router(messages.router, prefix="/v3")
app.include_router(conclusions.router, prefix="/v3")
app.include_router(keys.router, prefix="/v3")
app.include_router(webhooks.router, prefix="/v3")
app.add_route("/metrics", metrics_endpoint)    # Prometheus metrics
```

### 1.4 HTTP Middleware（請求追蹤）(`src/main.py:231-256`)

每個請求：
1. 產生 `request_id = "{METHOD}:{endpoint_pattern}:{uuid8}"`
2. 儲存在 `request.state.request_id` 和 `request_context` ContextVar
3. 請求結束後重設 ContextVar，確保非同步安全性
4. 如果 Prometheus 啟用，記錄請求指標

---

## 2. Deriver（背景 Worker）Entry Point

**啟動方式**：
```bash
uv run python -m src.deriver
```

**入口點** `src/deriver/__main__.py:67-84`：
```python
if __name__ == "__main__":
    setup_logging()
    asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())  # 高性能事件循環
    if settings.METRICS.ENABLED:
        start_metrics_server()       # Prometheus 指標 server（9090 端口）
    asyncio.run(run_deriver())
```

`run_deriver()` 流程：
```
1. initialize_telemetry_async()    ← CloudEvents 遙測
2. main()                          ← 進入 QueueManager 主循環
3. shutdown_telemetry()            ← 清理（finally）
```

**`main()` 函式** (`src/deriver/queue_manager.py:858-876`)：
```
1. init_cache()                    ← Redis 快取（可選）
2. QueueManager() 初始化
   - DreamScheduler 初始化（dream 任務排程器）
   - ReconcilerScheduler 初始化（向量同步排程器）
   - Sentry 整合（如啟用）
3. manager.initialize()
   - 設置 SIGTERM/SIGINT 信號處理器
   - 啟動 reconciler scheduler
   - 執行 polling_loop()
```

---

## 3. 資料庫初始化

**DB Engine** (`src/db.py:34-46`)：
- 使用 `create_async_engine`（asyncpg）
- 連線池設定：POOL_SIZE=10, MAX_OVERFLOW=20, POOL_TIMEOUT=30s
- POOL_USE_LIFO=True（減少 idle 連線）
- `prepare_threshold=None`（停用 server-side prepared statements，與 pgBouncer 相容）

**SessionLocal** = `async_sessionmaker`（`autocommit=False`, `autoflush=False`, `expire_on_commit=False`）

---

## 4. Sentry 初始化

在 module-level（`src/main.py:108-120`）：
- 若 `settings.SENTRY.ENABLED` 為 True，初始化 Sentry SDK
- 整合：`StarletteIntegration` + `FastApiIntegration`
- `before_send` 過濾器：HonchoException 和 ValidationError 不傳送到 Sentry

---

## 5. 設定載入流程 (`src/config.py:620-688`)

`AppSettings` 物件在 import 時立即初始化：
```python
settings: AppSettings = AppSettings()
```

優先順序（高→低）：
1. init 參數（通常無）
2. 環境變數（`DB_CONNECTION_URI`, `AUTH_JWT_SECRET` 等）
3. `.env` 檔案
4. `config.toml` 檔案（自訂 `TomlConfigSettingsSource`）
5. 預設值

`propagate_namespace` validator 在初始化後執行，將頂層 `NAMESPACE` 傳播到：
- `CACHE.NAMESPACE`
- `VECTOR_STORE.NAMESPACE`
- `TELEMETRY.NAMESPACE`
- `METRICS.NAMESPACE`

---

## 6. 全域例外處理器 (`src/main.py:206-227`)

```
HonchoException → 對應 HTTP status code + detail JSON
Exception       → 500 + Sentry capture
ValidationError → 422（由 FastAPI 自動處理，filtered from Sentry）
```

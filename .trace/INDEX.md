# Honcho 技術文件索引

> 文件版本：基於 Honcho v3.0.6-rc | 最後更新：2026-05-08（增量 trace 5b6bd59→a4ae372）

## 一句話總結

**Honcho** 是 Plastic Labs 開發的開源 AI agent 記憶基礎設施，透過三個協作的 LLM Agent（Deriver 記憶擷取、Dreamer 記憶整合、Dialectic 記憶查詢）讓任何 AI agent 能夠建立並長期維護對用戶或其他 agent 的個人化理解，支援用自然語言查詢取得個人化上下文。

---

## 技術棧總覽

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| 語言 | Python | ≥3.10 | 主要後端語言 |
| HTTP 框架 | FastAPI | ≥0.131.0 | REST API 框架 |
| ORM | SQLAlchemy | ≥2.0 | 資料庫抽象層 |
| 資料庫 | PostgreSQL + pgvector | - | 主要儲存 + 向量搜尋 |
| 向量 DB（替代）| Turbopuffer / LanceDB | - | 可替換向量後端 |
| 快取 | Redis（cashews）| ≥7.0.0 | 查詢快取（可選）|
| 套件管理 | uv | ≥0.5.0 | Python 依賴管理 |
| LLM（Deriver）| Google Gemini 2.5 Flash Lite | - | 記憶擷取（預設）|
| LLM（Dialectic low）| Google Gemini 2.5 Flash Lite | - | 低推理查詢 |
| LLM（Dialectic high）| Anthropic Claude Haiku 4.5 | - | 中高推理查詢 |
| LLM（Dream）| Anthropic Claude Sonnet 4 | - | 記憶整合（預設）|
| 嵌入 | OpenAI text-embedding-3-small | - | 向量嵌入（預設）|
| CLI 工具 | honcho-cli | - | 終端機 workspace 管理與除錯 |
| 遷移 | Alembic | ≥1.14.0 | 資料庫 schema 遷移 |
| 序列化 | Pydantic v2 | ≥2.11.7 | 輸入驗證與設定 |
| 部署 | Docker + Fly.io | - | 容器化和雲端部署 |
| 監控 | Sentry | - | 錯誤追蹤（可選）|
| 指標 | Prometheus | - | 指標收集（可選）|
| 遙測 | CloudEvents | ≥1.12.0 | 分析事件（可選）|
| 追蹤 | Langfuse | ≥3.3.2 | LLM 追蹤（可選）|

---

## 關鍵指令速查

```bash
# 環境設定
uv sync                                    # 安裝依賴
cp .env.template .env                      # 建立環境設定
cp config.toml.example config.toml         # 建立設定檔
python scripts/generate_jwt_secret.py      # 生成 JWT 金鑰

# 資料庫
uv run alembic upgrade head                # 執行 migrations

# 啟動服務
uv run fastapi dev src/main.py             # API server（開發模式）
uv run python -m src.deriver               # Background worker（Deriver）

# 測試
uv run pytest tests/                       # 執行所有測試
uv run pytest tests/ -k typescript         # 執行 TypeScript SDK 測試
uv run pytest tests/path/test_file.py::test_function  # 執行單個測試

# 程式碼品質
uv run ruff check src/                     # Lint
uv run ruff format src/                    # 格式化
uv run basedpyright                        # 類型檢查

# Docker
cp docker-compose.yml.example docker-compose.yml
docker compose up -d database              # 只啟動資料庫
docker compose up                          # 啟動完整服務

# CLI 工具（v3.0.6+ 新增）
uv tool install honcho-cli                 # 安裝 CLI
honcho init                                # 設定 API key + URL
honcho doctor                              # 健康檢查
honcho workspace queue-status -w <ws>      # 查看佇列狀態
honcho peer representation -w <ws> -p <peer> --observer <obs>  # 查看記憶
```

---

## 文件地圖

| 文件 | 說明 |
|------|------|
| [INDEX.md](./INDEX.md) | 本文件 — 總覽與速查 |
| [CODEBASE_MAP.md](./CODEBASE_MAP.md) | 程式碼地圖 — 目錄結構與「我要改 X 看哪裡？」|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 系統架構 — 元件、通訊模式、設計決策 |
| [DATA_MODEL.md](./DATA_MODEL.md) | 資料模型 — ORM 模型、ER 圖、資料生命週期 |
| [API_SURFACE.md](./API_SURFACE.md) | API 參考 — 所有端點、請求/回應範例 |
| [DEV_GUIDE.md](./DEV_GUIDE.md) | 開發者指南 — 環境設定、工作流程、除錯技巧 |
| [DISCOVERY_LOG.md](./DISCOVERY_LOG.md) | 探索紀錄 — 發現的問題、技術債、待解疑問 |

---

## 專案術語表

| 術語 | 說明 |
|------|------|
| **Workspace** | 最高層級組織單位，用於隔離不同應用的資料（前身：App）|
| **Peer** | 系統中任何參與者（人類用戶或 AI agent），統一抽象 |
| **Session** | 多個 Peer 之間的對話上下文（前身：Thread）|
| **Message** | 對話中的一條原子資料單元 |
| **Collection** | Peer 的文件集合（內部向量儲存，不透過 API 暴露）|
| **Document** / **Observation** | 向量嵌入的觀察記錄，儲存在 Collection 中 |
| **Deriver** | 背景記憶擷取 Agent — 處理訊息並生成明確觀察記錄 |
| **Dreamer** | 記憶整合 Agent — 深度分析，生成演繹和歸納觀察記錄 |
| **Dialectic** | 記憶查詢 API — 以自然語言回答關於 Peer 的查詢 |
| **Representation** | 對一個 Peer 的觀察記錄集合（記憶的完整表示）|
| **Peer Card** | 精簡的 Peer 摘要（最多 40 條事實，低延遲存取）|
| **Observer** | 觀察其他 Peer 的一方（Collection 中的 observer 欄位）|
| **Observed** | 被觀察的 Peer（Collection 中的 observed 欄位）|
| **Work Unit** | 一組需要按順序處理的佇列任務（以 work_unit_key 分組）|
| **explicit** | 觀察記錄層級：直接從訊息擷取的事實 |
| **deductive** | 觀察記錄層級：從明確事實邏輯推導的結論 |
| **inductive** | 觀察記錄層級：跨多個觀察發現的模式 |
| **Surprisal** | 幾何驚訝度 — 觀察記錄在嵌入空間中與近鄰的距離 |
| **Reasoning Level** | Dialectic 的推理深度（minimal/low/medium/high/max）|
| **nanoid** | 21 字元的隨機 ID 格式（`[A-Za-z0-9_-]{21}`），用於大多數主鍵 |
| **pgvector** | PostgreSQL 的向量搜尋擴展 |
| **HNSW** | Hierarchical Navigable Small World — 向量近似近鄰搜尋演算法 |
| **Reconciler** | 向量儲存同步背景任務（pgvector ↔ Turbopuffer/LanceDB）|
| **SSE** | Server-Sent Events — Dialectic streaming 回應格式 |
| **ModelTransport** | LLM 傳輸協定類型：`"anthropic"` / `"openai"` / `"gemini"`（取代舊的 SupportedProviders）|
| **ModelConfig** | 統一 LLM 模型配置物件（model + transport + 可選參數），取代分散的 PROVIDER/MODEL 字串對 |
| **ThinkingEffortLevel** | 推理思考強度：`none` / `minimal` / `low` / `medium` / `high` / `xhigh` / `max` |
| **ProviderBackend** | `src/llm/backend.py` 中的 ABC，各 LLM provider 的統一介面（AnthropicBackend / GeminiBackend / OpenAIBackend）|
| **honcho-cli** | 命令列工具，用於 workspace/peer/session 的終端機操作與除錯（`uv tool install honcho-cli`）|

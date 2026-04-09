# Stage 1 - Reconnaissance 偵察報告

## 1. 專案概述

**Honcho** 是 [Plastic Labs](https://plasticlabs.ai) 開發的開源 AI agent 記憶基礎設施平台（版本 3.0.5）。核心定位：

> 「為 agentic 世界提供身份層（Identity Layer）」

功能摘要：
- 讓 AI agent 對任何實體（用戶、agent、群組）建立並維護長期記憶
- 持續學習系統，能理解隨時間變化的實體
- 透過 Dialectic API 提供自然語言查詢個人化背景資訊
- 支援多 peer 會話（multi-peer sessions）

## 2. 技術棧

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| 語言 | Python | ≥3.10 | 主要後端語言 |
| 框架 | FastAPI | ≥0.131.0 | HTTP API 框架 |
| ORM | SQLAlchemy | ≥2.0.30 | 資料庫抽象層 |
| 資料庫 | PostgreSQL + pgvector | - | 主要儲存 + 向量搜尋 |
| 向量搜尋 | pgvector / Turbopuffer / LanceDB | - | 文件嵌入搜尋（可替換） |
| 快取 | Redis (cashews) | ≥7.0.0 | 查詢結果快取 |
| 嵌入 | OpenAI / Gemini / OpenRouter | - | 文字嵌入生成 |
| LLM（Deriver）| Google Gemini 2.5 Flash Lite | - | 預設記憶擷取模型 |
| LLM（Dialectic low）| Google Gemini 2.5 Flash Lite | - | 低推理層 |
| LLM（Dialectic med/high/max）| Anthropic Claude Haiku 4.5 | - | 中高推理層 |
| LLM（Dream）| Anthropic Claude Sonnet 4 | - | 記憶整合模型 |
| 遷移工具 | Alembic | ≥1.14.0 | 資料庫 schema 遷移 |
| 套件管理 | uv | ≥0.5.0 | Python 依賴管理 |
| 序列化 | Pydantic v2 + pydantic-settings | ≥2.11.7 | 輸入驗證和設定管理 |
| 分頁 | fastapi-pagination | ≥0.14.2 | API 分頁 |
| Token 計算 | tiktoken | ≥0.9.0 | LLM token 計算 |
| 觀測 | Sentry | - | 錯誤追蹤 |
| 指標 | Prometheus | - | Pull-based metrics |
| 遙測 | CloudEvents | ≥1.12.0 | 分析事件 |
| 追蹤 | Langfuse | ≥3.3.2 | LLM 呼叫追蹤 |
| JSON 修復 | json-repair | - | 修復格式錯誤的 LLM JSON 輸出 |
| 容器 | Docker | - | 容器化部署 |
| 部署 | Fly.io | - | 雲端部署平台 |
| 事件循環 | uvloop | - | 高性能 Python 事件循環 |
| 異步 HTTP | httpx | ≥0.27.0 | 非同步 HTTP 客戶端 |
| 類型檢查 | basedpyright | ≥1.29.4 | 靜態型別檢查 |
| Lint/格式 | ruff | ≥0.11.2 | Python linting 和格式化 |
| 測試 | pytest + pytest-asyncio | - | 測試框架 |
| TypeScript SDK | Bun | - | 執行環境 |

## 3. 架構模式

**Monorepo（單一倉庫）** 包含：
- 核心 API server（Python/FastAPI）
- Python SDK（`sdks/python/`）
- TypeScript SDK（`sdks/typescript/`）
- MCP server（`mcp/`）
- 範例（`examples/`）
- 文件（`docs/`）

整體架構是 **API Server + Background Worker** 雙行程模式：
- FastAPI server（API 服務）
- Deriver worker（背景處理）

## 4. 目錄結構（3層深）

```
honcho/
├── src/                          # 核心 API server 程式碼
│   ├── main.py                   # FastAPI 應用程式入口點
│   ├── models.py                 # SQLAlchemy ORM 模型
│   ├── config.py                 # 階層式設定管理
│   ├── db.py                     # 資料庫連線池管理
│   ├── dependencies.py           # FastAPI 依賴注入
│   ├── exceptions.py             # 自訂例外類型
│   ├── security.py               # JWT 認證
│   ├── embedding_client.py       # 嵌入服務客戶端
│   ├── schemas/                  # Pydantic 驗證 schema
│   │   ├── api.py               # API 輸入輸出模型
│   │   ├── configuration.py     # 設定 schema
│   │   └── internal.py          # 內部資料模型
│   ├── crud/                     # 資料庫 CRUD 操作
│   │   ├── workspace.py         # Workspace 操作
│   │   ├── peer.py              # Peer 操作
│   │   ├── session.py           # Session 操作
│   │   ├── message.py           # Message 操作
│   │   ├── collection.py        # Collection 操作
│   │   ├── document.py          # Document 操作
│   │   ├── deriver.py           # Deriver 相關 CRUD
│   │   ├── peer_card.py         # Peer Card 操作
│   │   ├── representation.py    # RepresentationManager
│   │   ├── session.py           # Session 操作
│   │   └── webhook.py           # Webhook 操作
│   ├── routers/                  # API 路由
│   │   ├── workspaces.py        # /workspaces 端點
│   │   ├── peers.py             # /peers 端點（含 Dialectic）
│   │   ├── sessions.py          # /sessions 端點
│   │   ├── messages.py          # /messages 端點
│   │   ├── conclusions.py       # /conclusions 端點
│   │   ├── keys.py              # /keys JWT 端點
│   │   └── webhooks.py          # /webhooks 端點
│   ├── deriver/                  # 背景記憶擷取系統
│   │   ├── __main__.py          # Deriver 進程入口點
│   │   ├── queue_manager.py     # 佇列管理主循環
│   │   ├── consumer.py          # 佇列項目處理器（分派各種任務）
│   │   ├── deriver.py           # 表示層批次處理
│   │   ├── enqueue.py           # 佇列化操作
│   │   └── prompts.py           # Deriver agent 提示詞
│   ├── dialectic/                # Dialectic API 實作
│   │   ├── chat.py              # agentic_chat() 入口函式
│   │   ├── core.py              # DialecticAgent 類別
│   │   └── prompts.py           # Dialectic agent 提示詞
│   ├── dreamer/                  # 記憶整合系統
│   │   ├── orchestrator.py      # Dream 週期協調器
│   │   ├── specialists.py       # 演繹/歸納 specialist agents
│   │   ├── dream_scheduler.py   # 排程觸發器
│   │   ├── surprisal.py         # 幾何驚訝度採樣
│   │   └── trees/               # 多種近鄰樹實作
│   ├── reconciler/               # 向量資料庫同步
│   │   ├── scheduler.py         # 同步排程
│   │   ├── sync_vectors.py      # 向量同步邏輯
│   │   └── queue_cleanup.py     # 佇列清理
│   ├── vector_store/             # 可插拔向量儲存後端
│   │   ├── lancedb.py           # LanceDB 後端
│   │   ├── turbopuffer.py       # Turbopuffer 後端
│   │   └── utils.py             # 向量工具函式
│   ├── cache/                    # Redis 快取
│   │   └── client.py            # cashews 快取客戶端
│   ├── telemetry/                # 可觀測性基礎設施
│   │   ├── logging.py           # 效能指標日誌
│   │   ├── emitter.py           # CloudEvents 發送器
│   │   ├── metrics_collector.py # 指標收集
│   │   ├── prometheus/          # Prometheus 指標
│   │   ├── events/              # CloudEvents 事件定義
│   │   ├── reasoning_traces.py  # LLM 推理追蹤
│   │   └── sentry.py            # Sentry 整合
│   ├── utils/                    # 共用工具
│   │   ├── agent_tools.py       # 三個 agent 的共用工具
│   │   ├── clients.py           # LLM 客戶端抽象
│   │   ├── search.py            # 混合搜尋（向量+全文）
│   │   ├── summarizer.py        # Session 摘要
│   │   ├── formatting.py        # 訊息格式化
│   │   ├── representation.py    # Representation 類別
│   │   ├── types.py             # 型別定義
│   │   └── tokens.py            # Token 計算
│   └── webhooks/                 # Webhook 系統
│       ├── events.py            # Webhook 事件定義
│       └── webhook_delivery.py  # Webhook 遞送邏輯
├── sdks/                         # 客戶端 SDK
│   ├── python/                  # Python SDK (honcho-ai)
│   └── typescript/              # TypeScript SDK (@honcho-ai/sdk)
├── migrations/                   # Alembic 資料庫遷移
│   └── versions/                # 遷移腳本
├── docs/                         # 官方文件（多版本）
│   ├── v1/, v2/, v3/            # 版本化文件
│   └── images/                  # 文件圖片
├── tests/                        # 測試套件
│   ├── alembic/                 # Migration 測試
│   └── bench/                   # 基準測試
├── examples/                     # 整合範例
│   ├── crewai/, langgraph/      # 框架整合
│   ├── zo/                      # 示範 agent
│   └── gmail/, granola/         # 應用整合
├── mcp/                          # MCP (Model Context Protocol) server
│   └── src/tools/               # MCP 工具
├── database/                     # 資料庫設定檔案
├── scripts/                      # 工具腳本
├── docker/                       # Docker 相關設定
├── Dockerfile                    # 容器建置
├── docker-compose.yml.example    # 本地開發 Docker compose 模板
├── pyproject.toml                # Python 專案設定（含依賴）
├── config.toml.example           # 完整設定範本（帶說明）
├── .env.template                 # 環境變數範本
├── alembic.ini                   # Alembic 遷移設定
├── fly.toml                      # Fly.io 部署設定
├── CLAUDE.md                     # AI assistant 指引文件
└── README.md                     # 專案主要說明文件
```

## 5. 既有文件掃描

`docs/` 目錄包含 v1、v2、v3 三個版本的文件：
- `docs/v3/` — 最新版本文件
  - `api-reference/` — API 參考
  - `documentation/` — 核心概念說明
  - `guides/` — 整合指南
  - `migrations/` — 版本遷移指南
  - `contributing/` — 貢獻指南

**CLAUDE.md** 包含開發者詳細指引，是本次 trace 的重要參考。

## 6. 文件與程式碼落差分析

| # | 文件說明 | 程式碼實際狀態 |
|---|---------|---------------|
| 1 | CLAUDE.md 提到 `src/deriver/agent/` 目錄（含 core.py、worker.py、prompts.py）| 實際上 deriver agent 已重構，目錄結構為 `src/deriver/deriver.py`（批次處理），沒有 agent 子目錄 |
| 2 | CLAUDE.md 提到 `src/dialectic/agent/` 目錄（含 core.py、prompts.py）| 實際上 dialectic agent 位於 `src/dialectic/core.py`（無 agent 子目錄） |
| 3 | CLAUDE.md 提到 `src/dreamer/agent.py`（DreamerAgent 類別）| 實際上 dreamer 已重構為 `orchestrator.py` + `specialists.py` 架構 |
| 4 | CLAUDE.md 提到 `src/utils/shared_models.py`、`src/utils/logging.py`、`src/utils/files.py` 等 | 部分存在（files.py 存在），但 logging 已移至 `src/telemetry/logging.py` |
| 5 | README 提到 Workspace 是「App」的舊名稱 | 現在完全使用 Workspace，且 API 路由也是 `/v3/workspaces/...` |
| 6 | CLAUDE.md 提到 `src/utils/summarizer.py` | 實際路徑正確 `src/utils/summarizer.py` ✓ |
| 7 | CLAUDE.md 描述 Dreamer 為「Random walk exploration」策略 | 現在已是「Surprisal-based sampling + specialist agents」架構 |

## 7. CI/CD 流程

- **GitHub Actions** 工作流程：
  - `unified-tests.yml` — 整合測試（在 Fly.io runner 上執行，需要 AWS 和 LLM API 金鑰）
  - `unittest.yml` — 單元測試
  - `staticanalysis.yml` — 靜態分析（ruff、basedpyright）
  - `docker-build.yml` — Docker 映像建置
  - `fly-deploy.yml` / `fly-deploy-prod.yml` — 部署到 Fly.io

## 8. 部署模式

- **自架（Self-hosted）**: Docker + PostgreSQL（含 pgvector）
- **雲端託管**: Fly.io（提供 fly.toml 設定）
- **Managed SaaS**: app.honcho.dev

## 9. 核心 API 路由前綴

所有 API 路由前綴為 `/v3/`：
```
/v3/workspaces/{workspace_id}/peers/{peer_id}/chat     ← Dialectic API
/v3/workspaces/{workspace_id}/peers/{peer_id}/representation
/v3/workspaces/{workspace_id}/sessions/{session_id}/context
/v3/workspaces/{workspace_id}/sessions/{session_id}/messages
```

## 10. 關鍵設計原則

1. **不在外部呼叫期間保持 DB session** — 所有 LLM/embedding 呼叫前必須先關閉 DB 連線
2. **Peer 統一模型** — 人類使用者和 AI agent 都用 `Peer` 表示
3. **多向量儲存後端** — pgvector（預設）、Turbopuffer、LanceDB 可切換
4. **背景處理佇列** — PostgreSQL 作為佇列（QueueItem 表），避免外部 message queue 依賴
5. **五層推理等級** — minimal / low / medium / high / max（不同成本與能力）

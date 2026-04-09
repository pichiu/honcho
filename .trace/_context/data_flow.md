# Stage 2.2 - Request / Data Flow

## 代表性 Use Case：建立訊息並觸發記憶處理

選擇 **POST /v3/workspaces/{workspace_id}/sessions/{session_id}/messages** 作為追蹤路徑，因為這是 Honcho 最核心的操作，會觸發後續整個記憶管線。

---

## 完整 Request 流程

```
客戶端
  │
  │ POST /v3/workspaces/{workspace_id}/sessions/{session_id}/messages
  │ Content-Type: application/json
  │ Authorization: Bearer <jwt_token>
  │ Body: {"messages": [{"peer_id": "alice", "content": "Hello"}]}
  ▼
FastAPI HTTP Stack
  │
  ├─ track_request middleware (src/main.py:231-256)
  │    ├─ 生成 request_id = "POST:_workspaces_sessions_messages:abc12345"
  │    └─ 設定 request_context ContextVar
  │
  ├─ CORSMiddleware
  │
  ├─ require_auth dependency (src/security.py)
  │    ├─ 若 AUTH_USE_AUTH=false → 允許所有請求
  │    └─ 若啟用 → 驗證 JWT，解析 workspace/peer/session 範圍
  │
  ├─ get_db dependency (src/dependencies.py:12-35)
  │    └─ 從 SessionLocal 取得 AsyncSession
  │
  ▼
create_messages_for_session (src/routers/messages.py:83-134)
  │
  ├─ crud.create_messages(db, messages, workspace_name, session_name)
  │    └─ (src/crud/message.py)
  │         ├─ 為每條訊息計算 token_count（tiktoken）
  │         ├─ 分配 seq_in_session（遞增序列號）
  │         ├─ 計算訊息嵌入（若 EMBED_MESSAGES=true）
  │         ├─ INSERT INTO messages
  │         ├─ INSERT INTO message_embeddings（sync_state="pending"）
  │         └─ commit()
  │
  ├─ prometheus_metrics.record_messages_created()（若啟用）
  │
  ├─ 準備 payloads 列表（message_id, content, peer_name, seq, ...）
  │
  ├─ background_tasks.add_task(enqueue, payloads)
  │    └─ [非同步，不阻塞 HTTP 回應]
  │
  └─ return created_messages → 201 Created
```

---

## 背景任務：訊息佇列化 (enqueue)

```
enqueue(payloads) (src/deriver/enqueue.py:26-79)
  │
  ├─ dream_scheduler.cancel_dreams_for_observed()
  │    └─ 取消與這批訊息觀察對象相關的待執行 dream 任務
  │
  ├─ tracked_db("message_enqueue")
  │
  ├─ handle_session(db, payload, workspace_name, session_name)
  │    │
  │    ├─ crud.get_or_create_session(...)
  │    │    └─ 確保 session 存在，返回設定
  │    │
  │    ├─ crud.get_workspace(...)
  │    │
  │    ├─ get_configuration(None, session, workspace)
  │    │    └─ 解析設定優先序（訊息 > session > workspace > 全域預設）
  │    │
  │    ├─ get_peers_with_configuration(db, workspace, session)
  │    │    └─ 查詢所有 session_peers 的設定（observe_me, observe_others）
  │    │
  │    └─ 對每條訊息呼叫 generate_queue_records()
  │         │
  │         ├─ 判斷是否觸發摘要（每 20 條一次短摘，60 條一次長摘）
  │         │    └─ 若是 → 建立 summary 佇列記錄
  │         │
  │         ├─ 判斷觀察者列表（observe_me, observe_others 設定）
  │         │    ├─ 如果 sender 啟用了 observe_me → 加入 self-observation
  │         │    └─ 其他 peers 若啟用 observe_others → 加入到觀察者列表
  │         │
  │         └─ 建立 representation 佇列記錄（含所有觀察者）
  │
  └─ INSERT INTO queue（批次）→ commit()
```

---

## 背景 Worker 循環

```
QueueManager.polling_loop() (src/deriver/queue_manager.py:361-406)
  │
  ├─ 每 POLLING_SLEEP_INTERVAL_SECONDS 秒輪詢
  │
  ├─ get_and_claim_work_units()
  │    ├─ 查詢 QueueItem 表（按 work_unit_key 分組）
  │    ├─ 過濾掉 ActiveQueueSession 中已處理的
  │    ├─ representation 任務：累積 token ≥ REPRESENTATION_BATCH_MAX_TOKENS 才處理
  │    └─ 插入 ActiveQueueSession（樂觀鎖定，ON CONFLICT DO NOTHING）
  │
  └─ 為每個 work_unit_key 建立 asyncio.Task
       └─ process_work_unit(work_unit_key, worker_id)
            │
            ├─ 若是 "representation" 任務：
            │    ├─ get_queue_item_batch()
            │    │    ├─ 查詢從最早未處理訊息開始的上下文視窗
            │    │    ├─ 包含前一條訊息（若來自不同 peer，提供對話上下文）
            │    │    └─ 按 token 上限截斷
            │    │
            │    └─ process_representation_batch()
            │         └─ → 呼叫 Deriver Agent（見下方）
            │
            └─ 其他任務（summary / dream / deletion / webhook）：
                 └─ process_item(queue_item)
                      └─ 路由到對應處理器
```

---

## Deriver Agent 處理（記憶擷取）

```
process_representation_tasks_batch() (src/deriver/deriver.py)
  │
  ├─ 取得 working representation（現有觀察記錄）
  │
  ├─ 格式化訊息為 LLM 輸入
  │
  ├─ 呼叫 LLM（Google Gemini 2.5 Flash Lite，預設）
  │    └─ 工具呼叫循環：
  │         ├─ create_observations()   → 建立明確觀察記錄
  │         ├─ update_peer_card()      → 更新 peer card
  │         ├─ search_memory()         → 查詢現有記憶（去重用）
  │         └─ [其他工具]
  │
  ├─ 儲存新觀察記錄到 documents 表
  │    ├─ 生成 embedding（OpenAI text-embedding-3-small）
  │    └─ 更新 collection 的文件計數
  │
  ├─ 若 DREAM.ENABLED 且文件計數 ≥ DOCUMENT_THRESHOLD：
  │    └─ dream_scheduler.schedule_dream()  ← 排程一個 dream 任務
  │
  └─ 標記 QueueItem 為 processed=True
```

---

## Dialectic Chat 流程（查詢記憶）

```
POST /v3/workspaces/{workspace_id}/peers/{peer_id}/chat
  │
  ├─ require_auth dependency
  │
  ├─ crud.get_or_create_peers()
  │
  ├─ 若 options.stream=true → StreamingResponse（SSE）
  │
  └─ agentic_chat(workspace_name, session_name, query, observer, observed, reasoning_level)
       │
       ├─ tracked_db("dialectic.preflight")
       │    ├─ crud.get_peer(observer)
       │    ├─ crud.get_peer(observed)
       │    ├─ crud.get_session()（若指定）
       │    ├─ get_configuration()
       │    └─ crud.get_peer_card()（若啟用）
       │         ← [DB session 關閉，Agent 在無 DB 連線情況下運行]
       │
       ├─ DialecticAgent(workspace, session, observer, observed, peer_cards, reasoning_level)
       │
       └─ agent.answer(query)
            │
            ├─ 依 reasoning_level 選擇 LLM（minimal=Gemini Lite, max=Claude Haiku）
            │
            ├─ 工具呼叫循環（最多 MAX_TOOL_ITERATIONS 次）：
            │    ├─ search_memory()            → 語義搜尋觀察記錄
            │    ├─ get_recent_history()       → 最近對話記錄
            │    ├─ get_observation_context()  → 觀察記錄詳情
            │    ├─ get_peer_card()            → peer card
            │    ├─ get_session_summary()      → 對話摘要
            │    └─ create_observations()      → 演繹性觀察（僅 dialectic）
            │
            └─ 最終合成回應 → 返回字串
```

---

## Dream 整合流程（記憶整合）

```
DreamScheduler.schedule_dream() (src/dreamer/dream_scheduler.py)
  │
  ├─ 若該 peer 已有待執行的 dream → 跳過（去重）
  │
  ├─ 等待 IDLE_TIMEOUT_MINUTES（預設 60 分鐘空閒）
  │
  └─ enqueue_dream(workspace, observer, observed, dream_type)
       └─ INSERT INTO queue（task_type="dream"）
            │
            └─ [Deriver worker 取出並處理]
                 └─ process_dream(payload, workspace_name)
                      └─ run_dream(workspace, observer, observed)
                           │
                           ├─ [可選] 幾何驚訝度採樣（surprisal sampling）
                           │    └─ 找出 embedding 空間中最「意外」的觀察記錄
                           │
                           ├─ 演繹 specialist（DEDUCTION_MODEL）
                           │    └─ 從明確事實推導邏輯結論
                           │
                           └─ 歸納 specialist（INDUCTION_MODEL）
                                └─ 跨觀察記錄找出模式
```

---

## 關鍵資料轉換對照

| 層次 | 輸入 | 輸出 |
|------|------|------|
| Router | HTTP Request + Path/Body params | validated schema objects |
| CRUD | Pydantic schema | SQLAlchemy models |
| Enqueue | SQLAlchemy messages | QueueItem records |
| QueueManager | work_unit_key | Semaphore-controlled tasks |
| Deriver | Message list + Representation | Document observations |
| Dialectic | query + context | Natural language response |
| Dream | Observation graph | Consolidated observations |

---

## 工作單元鍵（work_unit_key）格式

```
representation:{workspace_name}:{session_name}:{observed}
summary:{workspace_name}:{session_name}
dream:{workspace_name}:{observer}:{observed}
deletion:{workspace_name}:{resource_id}
reconciler:{type}
webhook:{workspace_name}:{url_hash}
```

用途：按 work_unit_key 分組確保同一 session 的同一 peer 的訊息依序處理。

# Stage 2.3 - 核心領域邏輯

## 1. 系統「心臟」：三個 LLM Agent 的協作架構

Honcho 的核心是三個專業化 LLM Agent，它們分工協作完成「記憶形成 → 記憶整合 → 記憶查詢」的完整閉環。

---

## 2. Deriver（記憶擷取 Agent）

### 2.1 功能定位

- **目標**：從訊息流中擷取明確事實（explicit observations）
- **觸發**：每條新訊息建立後，透過佇列異步觸發
- **策略**：窮盡式明確資訊擷取（exhaustive explicit capture），速度優先
- **主要入口**：`src/deriver/deriver.py` → `process_representation_tasks_batch()`

### 2.2 批次處理機制

```python
# src/deriver/queue_manager.py:258-333
# 批次條件：token 累積量 ≥ REPRESENTATION_BATCH_MAX_TOKENS（預設 1024 tokens）
# 每次處理包含：
#   - 從最早未處理訊息開始的上下文視窗
#   - 包含前一條訊息（若來自不同 peer，提供對話上下文）
#   - 按 token 上限截斷
```

### 2.3 觀察記錄層級

```python
# src/utils/agent_tools.py:177-242
# 四種觀察層級：
# - explicit      → 直接事實（"用戶提到喜歡 Python"）
# - deductive     → 邏輯必然（從明確事實推導）
# - inductive     → 模式（跨多個觀察發現的趨勢）
# - contradiction → 矛盾（觀察記錄間的衝突）
```

### 2.4 觀察記錄去重

- `DEDUPLICATE=true` 時，新觀察記錄建立前先語義搜尋現有記錄
- 使用 `asyncio.Lock`（按 workspace/observer/observed 分組）防止並發建立重複記錄
- Lock 使用 `WeakValueDictionary` 自動清理（`src/utils/agent_tools.py:56-91`）

---

## 3. Dreamer（記憶整合 Agent）

### 3.1 功能定位

- **目標**：深度整合和彙整記憶，生成高層次推論
- **觸發**：
  1. 文件計數 ≥ DOCUMENT_THRESHOLD（50）
  2. 空閒 IDLE_TIMEOUT_MINUTES（60 分鐘）後
  3. 從上次 dream 起至少 MIN_HOURS_BETWEEN_DREAMS（8 小時）
- **主要入口**：`src/dreamer/orchestrator.py` → `run_dream()`

### 3.2 Dream 週期（orchestrator.py:65-263）

```
Phase 0: [可選] 幾何驚訝度採樣（Surprisal Sampling）
  - 從觀察記錄中採樣
  - 用 kNN 樹計算嵌入空間中的驚訝度分數
  - 選取最「驚訝」的 10% 作為探索提示

Phase 1: 演繹 Specialist（DEDUCTION_MODEL = claude-haiku-4-5）
  - 從明確事實推導邏輯結論
  - 自主探索觀察空間
  - 產生 deductive 層級觀察記錄

Phase 2: 歸納 Specialist（INDUCTION_MODEL = claude-haiku-4-5）
  - 跨觀察記錄識別模式和趨勢
  - 建立 inductive 層級觀察記錄
  - 在 deduction 完成後執行（可看到 deductive 觀察記錄）
```

### 3.3 Surprisal 採樣子系統（src/dreamer/surprisal.py + trees/）

支援多種近鄰樹（都是 `VectorStore` 的「前驅」計算）：
- **kdtree** / **balltree** — scikit-learn 包裝
- **rptree** — 隨機投影樹
- **covertree** / **lsh** / **graph** / **prototype** — 各種 ANN 演算法

計算方式：測量觀察記錄嵌入向量與其 k 最近鄰的距離，高距離 = 高驚訝度 = 值得重新探索。

### 3.4 Dream 排程器（src/dreamer/dream_scheduler.py）

```python
# DreamScheduler 管理待執行的 dream 任務
# - 新訊息到來時取消待執行的 dream（用戶仍然活躍）
# - 空閒後重新排程
# - 使用 asyncio.Lock 確保同一 peer 只有一個排程的 dream
```

---

## 4. Dialectic Agent（記憶查詢 Agent）

### 4.1 功能定位

- **目標**：以自然語言回答關於 peer 的查詢
- **觸發**：API 呼叫 `/peers/{peer_id}/chat`
- **主要入口**：`src/dialectic/core.py` → `DialecticAgent.answer()`

### 4.2 五個推理等級

| 等級 | Provider | Model | 思考預算 | 最大工具迭代 | 用途 |
|------|----------|-------|---------|------------|------|
| minimal | google | gemini-2.5-flash-lite | 0 | 1 | 超快速，極低成本 |
| low | google | gemini-2.5-flash-lite | 0 | 5 | 快速，低成本 |
| medium | anthropic | claude-haiku-4-5 | 1024 | 2 | 平衡 |
| high | anthropic | claude-haiku-4-5 | 1024 | 4 | 深度推理 |
| max | anthropic | claude-haiku-4-5 | 2048 | 10 | 最全面 |

### 4.3 Agent 工具集（src/utils/agent_tools.py:177+）

所有三個 Agent 共用工具定義，透過 `create_tool_executor()` 工廠函式建立上下文感知的執行器：

| 工具 | Deriver | Dialectic | Dream Specialist | 說明 |
|------|---------|-----------|-----------------|------|
| `create_observations` | ✓（全部層級）| ✓（僅 deductive）| ✓ | 建立觀察記錄 |
| `search_memory` | ✓ | ✓ | ✓ | 語義搜尋觀察記錄 |
| `get_recent_history` | ✓ | ✓ | ✓ | 最近對話記錄 |
| `get_observation_context` | ✓ | ✓ | ✓ | 觀察記錄詳情 |
| `search_messages` | ✓ | ✓ | ✓ | 搜尋原始訊息 |
| `update_peer_card` | ✓ | - | ✓ | 更新 peer card |
| `get_peer_card` | - | ✓ | - | 取得 peer card |
| `get_session_summary` | - | ✓ | - | 取得對話摘要 |
| `delete_observations` | - | - | ✓ | 刪除過時觀察 |
| `get_recent_observations` | - | ✓ | ✓ | 最新觀察記錄 |
| `get_most_derived_observations` | - | ✓ | ✓ | 引用次數最多的觀察 |

### 4.4 Session History 注入

Dialectic 在有 `session_id` 時會自動注入最近的對話記錄（最多 `SESSION_HISTORY_MAX_TOKENS=4096` tokens），讓 Agent 能感知當前對話上下文。

---

## 5. LLM 客戶端抽象（src/utils/clients.py）

### 5.1 支援的 Provider

| Provider | 使用場景 | 特殊功能 |
|---------|---------|---------|
| `google` | Deriver、低推理 Dialectic、Summary | Google Gen AI SDK |
| `anthropic` | 中高推理 Dialectic、Dream | 擴展思維（Extended Thinking）|
| `openai` | 嵌入（text-embedding-3-small）| 相容 API |
| `custom` | 任何 OpenAI 相容端點（OpenRouter 等）| 可指向任意端點 |
| `vllm` | 自架模型 | OpenAI 相容格式 |
| `groq` | 可選（預設不使用）| Groq API |

### 5.2 備用 Provider（Backup Provider）

所有 LLM 設定支援 `BACKUP_PROVIDER` / `BACKUP_MODEL`：
- 主 provider 失敗時自動切換
- Gemini 的 SAFETY/RECITATION/PROHIBITED_CONTENT 回應也觸發備用切換
- 使用 `tenacity` 重試機制（指數退避）

### 5.3 工具呼叫循環

`honcho_llm_call()` 函式封裝了各 provider 的差異：
1. 格式化工具定義（適配各 provider schema）
2. 呼叫 LLM API
3. 解析工具呼叫
4. 執行工具
5. 將工具結果加入對話歷史
6. 重複直到達到 MAX_TOOL_ITERATIONS 或無工具呼叫

---

## 6. Representation（表示層）

### 6.1 RepresentationManager（src/crud/representation.py）

管理對一個 peer 的觀察記錄集合：
- **working representation**：供 Deriver 處理時使用的當前觀察記錄集
- **semantic search**：語義搜尋找相關觀察記錄
- **most derived**：`times_derived` 計數最高的觀察記錄（被引用次數最多）

### 6.2 Representation 格式（src/utils/representation.py）

```python
class ExplicitObservationBase(BaseModel):
    content: str                           # 觀察內容
    
class ObservationMetadata(BaseModel):
    id: str                                # Document ID
    created_at: datetime
    message_ids: list[int]                 # 來源訊息 ID
    session_name: str | None
```

格式化為 Markdown 後注入 LLM 提示詞。

### 6.3 Peer Card（src/crud/peer_card.py）

- 靜態、精簡的 peer 摘要（最多 40 條事實）
- 由 Deriver 和 Dream Specialist 的 `update_peer_card` 工具維護
- 適合低延遲注入（不需要 LLM 查詢）
- 儲存在 documents 表的 `internal_metadata` 欄位中

---

## 7. 佇列系統設計（src/models.py + queue_manager.py）

### 7.1 PostgreSQL 作為 Message Queue

**QueueItem** 表取代外部 message queue（Kafka / Redis Streams / SQS）：
- 使用 `ON CONFLICT DO NOTHING` 實現樂觀鎖定
- **ActiveQueueSession** 表追蹤哪個 worker 正在處理哪個 work unit
- Partial unique index 確保 dream 和 reconciler 任務不重複

### 7.2 work_unit_key 分組策略

```
representation:{workspace}:{session}:{observed}
```

同一個 work_unit_key 的 QueueItem 按 `id`（BigInteger Identity）順序處理，確保記憶處理遵循對話順序。

### 7.3 任務類型（TaskType）

```python
# src/utils/types.py
TaskType = Literal[
    "representation",  # 記憶擷取
    "summary",         # 對話摘要
    "dream",           # 記憶整合
    "deletion",        # 資源刪除
    "reconciler",      # 向量同步
    "webhook",         # Webhook 遞送
]
```

---

## 8. 混合搜尋（src/utils/search.py）

搜尋策略結合向量和全文：
1. **向量搜尋**：對查詢文字生成嵌入向量，使用 cosine similarity 搜尋
2. **全文搜尋**：PostgreSQL GIN 索引（`to_tsvector('english', content)`）
3. **結果合并**：去重後根據語義相似度排序

支援的搜尋層級：
- Workspace 層級（所有 sessions + peers）
- Session 層級（特定 session）
- Peer 層級（特定 peer 的所有 sessions）

---

## 9. 設定繼承體系（src/utils/config_helpers.py）

```
訊息級設定 > Session 設定 > Workspace 設定 > 全域預設
```

具體設定項目（階層化）：
- `reasoning.enabled` — 是否啟用記憶擷取
- `summary.enabled` — 是否啟用摘要
- `summary.messages_per_short_summary` — 短摘觸發間隔
- `dream.enabled` — 是否啟用 dream
- `peer_card.use` — 是否使用 peer card
- `observe_me` — 此 peer 是否應被觀察（SessionPeer 設定）
- `observe_others` — 此 peer 是否觀察其他人（SessionPeer 設定）

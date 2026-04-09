# DISCOVERY_LOG.md — 探索紀錄

> 本文件記錄在 Honcho v3.0.5 程式碼 trace 過程中發現的問題、技術債、設計疑問與潛在改進點。

---

## 1. 文件與程式碼落差（高優先）

### D-001：CLAUDE.md 描述的目錄結構已過時

**發現**：CLAUDE.md（AI assistant 指引）描述了三個子目錄結構，但實際程式碼已重構：

| CLAUDE.md 描述 | 實際程式碼 | 狀態 |
|---------------|-----------|------|
| `src/deriver/agent/` (core.py, worker.py, prompts.py) | `src/deriver/deriver.py` + `src/deriver/prompts.py` | ❌ 目錄不存在 |
| `src/dialectic/agent/` (core.py, prompts.py) | `src/dialectic/core.py` + `src/dialectic/prompts.py` | ❌ 目錄不存在 |
| `src/dreamer/agent.py` (DreamerAgent class) | `src/dreamer/orchestrator.py` + `src/dreamer/specialists.py` | ❌ 檔案不存在 |
| `src/utils/shared_models.py` | 不存在 | ❌ |
| `src/utils/logging.py` | 已移至 `src/telemetry/logging.py` | ❌ |

**影響**：新 AI assistant 讀 CLAUDE.md 後會找不到指定檔案，浪費偵錯時間。

**建議**：更新 CLAUDE.md 的 Project Structure 章節以反映當前實際目錄結構。

---

### D-002：CLAUDE.md 的 Dreamer 描述落後一個版本

**發現**：CLAUDE.md 描述 Dreamer 為「Random walk exploration - start from recent/high-value observations, search for related content, consolidate redundancies」，但實際已演進為：

1. **Surprisal-based sampling**（幾何驚訝度採樣）替代 random walk
2. **Specialist 架構**：Deduction Specialist + Induction Specialist
3. **三階段流程**：Surprisal 採樣 → Deduction → Induction

**影響**：Dreamer 行為理解不正確，調整 Dreamer 行為時會找錯地方。

**建議**：更新 CLAUDE.md 的 Dreamer Agent 章節。

---

### D-003：DreamScheduler 實際行為與描述有差異

**發現**：CLAUDE.md 說 Dreamer 的觸發條件包含「IDLE_TIMEOUT_MINUTES」，但 `DreamScheduler` 的實際邏輯更複雜：

- 新訊息到達時**取消**待執行的 dream
- 文件計數 ≥ DOCUMENT_THRESHOLD 才**排程** dream
- 空閒 IDLE_TIMEOUT_MINUTES 後才**執行**
- 必須距離上次 dream 至少 MIN_HOURS_BETWEEN_DREAMS

這些條件需要**同時滿足**，但文件給人印象是「定時觸發」。

---

## 2. 技術債與潛在問題

### D-004：PostgreSQL 作為 Message Queue 的擴展性限制

**發現**：`QueueItem` 表作為 message queue，輪詢間隔為 1 秒（`POLLING_SLEEP_INTERVAL_SECONDS`），每次輪詢執行 `SELECT ... FOR UPDATE SKIP LOCKED`。

**潛在問題**：
- 高並發場景下，`queue` 表可能成為瓶頸
- 目前沒有 dead letter queue（失敗任務只記錄 `error` 欄位，不重試）
- `ActiveQueueSession` 表不會自動清理（如果 worker 崩潰不優雅退出）

**現況**：對中等負載（< 1000 req/min）應該足夠，但需要監控。

**建議**：若未來遇到佇列積壓，考慮增加 WORKERS 設定或引入外部 queue（但這是大型重構）。

---

### D-005：ActiveQueueSession 的「孤兒鎖」問題

**發現**：若 Deriver worker 崩潰（kill -9），`active_queue_sessions` 中的記錄不會自動清理，導致對應的 work_unit_key 永遠不會被再次處理。

**現況**：`QueueManager` 在啟動時有清理邏輯（`src/deriver/queue_manager.py` 的 `initialize()`），但如果是**部分崩潰**（worker thread 失敗但主 process 仍在），清理不會觸發。

**緩解方式**：`last_updated` 欄位支持基於超時的清理，但現有程式碼是否有此清理邏輯需要進一步確認。

**建議**：在 `polling_loop()` 中定期清理超時的 ActiveQueueSession（如超過 1 小時未更新的記錄）。

---

### D-006：缺乏嵌入失敗的自動重試機制

**發現**：`message_embeddings` 的 `sync_state` 設計為追蹤向量同步狀態，但**嵌入本身生成失敗**時的重試邏輯不清楚。

在 `src/crud/message.py` 中，嵌入生成失敗記錄錯誤但訊息仍儲存（`embedding=NULL`）。`sync_state` 用於追蹤向量儲存同步，而非嵌入生成。

**潛在問題**：若嵌入 API 暫時不可用，批次訊息的嵌入會持續缺失，影響向量搜尋品質。

**建議**：考慮在 Reconciler 中也掃描 `embedding IS NULL` 的記錄並重新生成嵌入。

---

### D-007：Dream Specialist 之間沒有 DB 隔離

**發現**：Dream Deduction 和 Induction Specialist 在 `run_dream()` 中**串行執行**（先 Deduction，完成後再 Induction）。Induction Specialist 可以看到 Deduction 產生的觀察記錄（這是設計上的意圖），但如果 Deduction Specialist 產生了大量觀察記錄，Induction 的上下文視窗可能過大。

**現況**：輸入 token 限制（`DERIVER_MAX_INPUT_TOKENS`）應該有所保護，但未看到針對 Dream 的特定限制。

---

### D-008：collection 刪除可能遺留向量

**發現**：`Document` 支持軟刪除（`deleted_at`），Reconciler 負責從外部向量 DB 清理已刪除的文件。但如果整個 `Collection` 被刪除（CASCADE DELETE），相關的 Document 記錄也會被硬刪除（不是軟刪除），可能在外部向量 DB 留下孤兒向量。

**影響**：僅在使用外部向量 DB（Turbopuffer/LanceDB）時才有問題，預設 pgvector 不受影響。

---

## 3. 設計觀察與疑問

### D-009：Peer Card 的更新競爭

**發現**：Peer Card 可以由三個路徑更新：
1. Deriver Agent 的 `update_peer_card` 工具
2. Dream Specialist 的 `update_peer_card` 工具
3. 直接 PUT API 端點

當 Deriver 和 Dream 同時執行（理論上不應發生，因為有 work_unit 鎖定，但如果是不同的 observer/observed 組合），可能產生寫入競爭。

**現況**：Peer Card 儲存在 `documents` 表的 `internal_metadata` 欄位，更新為全量覆寫（非 merge）。

---

### D-010：Session History 注入的 Token 預算衝突

**發現**：在 Dialectic 中，Session History 注入（最多 `SESSION_HISTORY_MAX_TOKENS=4096 tokens`）和工具呼叫結果都會佔用 `MAX_INPUT_TOKENS` 預算。若 session 歷史很長，工具呼叫（search_memory 等）可用的上下文空間就縮小了。

**現況**：並未看到對 session history 長度的動態調整邏輯（如果加上工具結果超過上限）。

---

### D-011：消息序列號（seq_in_session）的並發安全

**發現**：`seq_in_session` 是在應用層計算的（`src/crud/message.py` 中用 `SELECT MAX(seq_in_session)` 計算），而非使用資料庫序列。

**潛在問題**：高並發建立訊息時可能有 race condition，兩個請求同時 SELECT MAX 後可能插入相同的 `seq_in_session`，觸發 `UniqueConstraint` 錯誤。

**現況**：有 `(workspace_name, session_name, seq_in_session)` 的唯一約束，DB 層面會阻止重複，但 application 層應該有重試邏輯。

---

## 4. 潛在改進建議

### D-012：Deriver 批次處理 Token 閾值的可調性

**發現**：`REPRESENTATION_BATCH_MAX_TOKENS=1024`（預設），表示需要累積足夠的新訊息 tokens 才觸發一次 Deriver 處理。在低流量環境，訊息可能長時間不被處理。

**現有解決方案**：`DERIVER_FLUSH_ENABLED=true` 可繞過此閾值，立即處理每條訊息。

**建議**：考慮添加「最大等待時間」設定（如「超過 5 分鐘未處理的訊息，即使不足 token 閾值也強制處理」），避免低流量場景下的處理延遲。

---

### D-013：Surprisal 採樣的樹型選擇缺乏文件

**發現**：`src/dreamer/trees/` 包含 7 種樹型（kdtree, balltree, rptree, covertree, lsh, graph, prototype），但沒有文件說明各種樹型的適用場景和效能差異。

`DREAM_SURPRISAL__TREE_TYPE` 的預設值是什麼也不明確（需要查看 config.py）。

**建議**：在設定文件中說明各樹型的特性和適用場景。

---

### D-014：Reconciler 的 pgvector 空行為

**發現**：當 `VECTOR_STORE_TYPE=pgvector`（預設），`ReconcilerScheduler` 仍會定期觸發，但 `sync_vectors` 幾乎是 no-op（pgvector 是直接寫入，不需要同步）。

可能造成不必要的 DB 查詢（掃描 `sync_state=pending` 的記錄）。

**建議**：若 `VECTOR_STORE_TYPE=pgvector`，可跳過 Reconciler 排程以節省資源。

---

## 5. 安全性觀察

### D-015：JWT 沒有過期機制（Expiration）

**發現**：`src/security.py` 的 `create_jwt()` 不設定 `exp` claim，表示產生的 JWT 永不過期。

**影響**：若 JWT 洩漏，無法強制失效（除非更換 `JWT_SECRET`）。

**現況**：對於 scoped JWT（peer/session level），這是有意的設計決定（方便 SDK 使用），但管理員 JWT 應考慮加入過期時間。

---

### D-016：Webhook HMAC 簽名的選擇性

**發現**：建立 Webhook 時 `secret` 是可選的（若不提供，則不進行 HMAC 簽名驗證）。這意味著接收方無法確認 webhook 是否來自真正的 Honcho。

**建議**：考慮強制要求 webhook secret，或至少在文件中明確說明不使用 secret 的安全風險。

---

## 6. 值得深入研究的領域

### D-017：Vector 命名空間的碰撞理論

向量儲存命名空間使用 SHA256 前 43 字元：
```
{NAMESPACE}.doc.{SHA256(workspace, observer, observed)[:43]}
{NAMESPACE}.msg.{SHA256(workspace)[:43]}
```

理論上有碰撞風險（43 個 hex 字元 = 172 bits），實際上不太可能，但值得注意。

### D-018：Dreamer 的 source_ids 追蹤

`Document.source_ids` 追蹤哪些觀察記錄啟發了這個觀察記錄（GIN 索引），形成記憶的「來源樹」。這個欄位的完整使用方式（如刪除傳播）值得進一步研究。

### D-019：times_derived 的計算方式

`Document.times_derived` 追蹤一個觀察記錄「被引用次數」，`get_most_derived_observations()` 返回最高引用次數的記錄。這些記錄被認為是最重要的記憶。但 `times_derived` 的遞增觸發條件不完全清楚（可能是當記錄被作為 source 產生新記錄時）。

---

## 7. 正面發現（值得保留的好設計）

### D-020：WeakValueDictionary 的 Lock 管理

`src/utils/agent_tools.py:56-91` 中，使用 `WeakValueDictionary` 管理 `asyncio.Lock`，確保沒有活躍用戶時，鎖會被垃圾回收。

這個設計優雅地解決了「記憶體不足但功能正確」的問題，避免了鎖的累積。

### D-021：tracked_db 的 request_context 追蹤

`src/dependencies.py` 的 `tracked_db` 自動設定 `request_context` ContextVar，讓 DB 查詢可以關聯到特定請求（用於追蹤和除錯）。同時確保非同步安全的事務清理。

### D-022：Provider 備用切換的優雅降級

`honcho_llm_call()` 中，Gemini 的 SAFETY/RECITATION/PROHIBITED_CONTENT 回應會自動觸發 backup provider 切換。這確保了即使主要 LLM 拒絕回應，系統仍能嘗試備用方案，提升可用性。

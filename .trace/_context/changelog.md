# Changelog — Base Commit 範圍 5b6bd59..a4ae372

> 產出日期：2026-05-08 | 本次增量 trace

---

## 統計

- **Commit 數**：49 個（不含 merge commit）
- **變更檔案總數**：152 個
- **新增行數**：17,211 | **刪除行數**：4,786
- **核心 src/ 變更**：~41 個檔案（25 修改 + 16 新增 `src/llm/`）

---

## 重大變更（Breaking / Structural）

### 1. `src/utils/clients.py` 刪除 → 新 `src/llm/` 模組取代

**最重大的架構變更**。原本的 2575 行 `clients.py` 被完全移除，改由全新的 `src/llm/` 套件（16 個檔案）取代。

**新架構**：
```
src/llm/
├── __init__.py        # 公開 API（honcho_llm_call, 型別等）
├── api.py             # 頂層入口點（含 retry, telemetry）
├── backend.py         # ProviderBackend ABC
├── backends/
│   ├── anthropic.py   # AnthropicBackend
│   ├── gemini.py      # GeminiBackend
│   └── openai.py      # OpenAIBackend
├── caching.py         # Prompt cache 策略
├── conversation.py    # 對話歷史管理
├── credentials.py     # API key 解析
├── executor.py        # honcho_llm_call_inner（單次呼叫）
├── history_adapters.py # 跨 provider 的歷史格式轉換
├── registry.py        # LRU-cached 客戶端 singleton + backend 選擇
├── request_builder.py # 請求建構
├── runtime.py         # AttemptPlan, plan_attempt, 運行時 config 解析
├── structured_output.py # JSON/Pydantic 輸出支援
├── tool_loop.py       # 工具呼叫迭代循環（extract 自原 clients.py）
└── types.py           # HonchoLLMCallResponse, IterationData 等型別
```

**關鍵變化**：
- `ModelTransport = Literal["anthropic", "openai", "gemini"]` 取代舊的 `SupportedProviders = Literal["anthropic", "google", "openai", "custom", "vllm", "groq"]`
- 新增 `ProviderBackend` ABC，各 provider 有獨立實作
- LRU-cached client singleton（`get_anthropic_client()` 等）集中在 `registry.py`
- Prompt caching 現為一等公民（`PromptCachePolicy`）
- Tool loop 獨立成 `tool_loop.py`，不再嵌入主函式
- 引入 `ModelConfig` 統一模型配置物件（見下）

---

### 2. `src/config.py` 大幅擴充（+867 行）— 新 ModelConfig 系統

原本 provider 以字串（`"anthropic"`, `"google"` 等）+ model 字串分散設定，現在引入統一的型別系統：

```python
ModelTransport = Literal["anthropic", "openai", "gemini"]

ThinkingEffortLevel = Literal["none", "minimal", "low", "medium", "high", "xhigh", "max"]

class ModelConfig(BaseModel):
    model: str
    transport: ModelTransport
    fallback: ResolvedFallbackConfig | None = None
    api_key: str | None = None
    base_url: str | None = None
    thinking_effort: ThinkingEffortLevel | None = None
    thinking_budget_tokens: int | None = None
    cache_policy: PromptCachePolicy | None = None
    ...

class ConfiguredModelSettings(BaseModel):
    """Operator-configurable (persisted in config.toml/env) model settings."""
    model: str
    transport: ModelTransport
    ...
```

**影響**：所有 `DERIVER_PROVIDER` / `DERIVER_MODEL` 等設定現在解析成 `ModelConfig` 物件，而非直接使用字串。

---

### 3. 新增 `honcho-cli/` 頂層套件

全新的命令列工具，用於 Honcho workspace 的終端機操作與除錯。

**安裝**：`uv tool install honcho-cli`

**主要命令**：
```
honcho init / doctor            # 設定與健康檢查
honcho workspace list/create/inspect/search/queue-status/delete
honcho peer list/create/inspect/representation/card/chat
honcho session list/create/inspect/context/messages
honcho message list/create
honcho conclusion search
```

**設定檔**：`~/.honcho/config.json`（apiKey + environmentUrl）

---

## 重要修正（Bug Fixes）

### 4. Dream Scheduler 語義修正（`fix(dreamer): threshold and time-guard semantics`）

**修正前問題（D-003）**：
- `DOCUMENT_THRESHOLD` 計算包含了所有層級（explicit + deductive + inductive），導致 Dreamer 本身產生的觀察記錄也計入閾值，造成 feedback loop
- `last_dream_document_count` 即使 dream 失敗也會更新，導致下次觸發條件錯誤

**修正後行為**：
- 閾值只計算 `level = "explicit"` 的文件（Dreamer 輸出不計入）
- `last_dream_at` 和 `last_dream_document_count` 只在成功整合後才更新
- 引入 `documents_since_last_dream` 計算，對比上次 dream 後新增的 explicit doc 數量

### 5. Dialectic N+1 查詢修正（`fix: internal N+1 query in dialectic agent calls`）

Dialectic agent 呼叫中存在內部 N+1 查詢問題，現已修正。

### 6. Vector Sync Retry 預算大幅增加（`fix: give vector sync a substantial retry budget`）

`src/reconciler/sync_vectors.py` 中向量同步的 tenacity 重試次數大幅增加，解決 D-006 觀察到的嵌入同步失敗問題。

### 7. Surprisal 過濾層級修正（`fix(surprisal): use correct filter format for level observations`）

`src/dreamer/surprisal.py` 中過濾 explicit 層級觀察記錄時使用了錯誤的格式，現已修正。

### 8. Deriver Stop Sequences 移除（`fix: remove hardcoded stop_sequences override from Deriver model config`）

原本 Deriver 有硬編碼的 stop sequences，現已移除，避免提前截斷 LLM 輸出。

### 9. 嵌入 embed() 輸入格式修正（`fix: embed() sends string input instead of array`）

修正 OpenAI 相容 providers 的嵌入函式傳入 array 而非 string 的問題。

---

## 新整合文件（docs/）

新增 4 個整合指南：
- **SillyTavern**（AI roleplay 平台）
- **Paperclip**（macOS AI assistant）
- **OpenCode**（terminal AI coding assistant）
- **Vercel AI SDK**（Web app 整合）

---

## MCP 更新

- 新增 `HONCHO_API_URL` 環境變數支援（用於 self-hosted Honcho）
- `mcp/src/config.ts` 更新配置邏輯

---

## 其他小變更

- Dockerfile 移除 API-specific healthcheck
- `pyproject.toml` 依賴更新（移除 groq SDK、更新 openai 版本等）
- `src/vector_store/utils.py` 刪除（功能已整合或移除）
- Turbopuffer server error 處理改進
- Langfuse metadata 新增 namespace/model/provider 欄位（方便過濾）

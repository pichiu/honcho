# 更新計畫

## 變更摘要
- **Base Commit 範圍**：5b6bd59..a4ae372
- **變更檔案數**：152（含 docs、tests、lock files）
- **核心 src/ 變更**：~41 個（25 修改 + 16 新增 `src/llm/`）
- **變更幅度**：~41%（中度更新，繼續增量）

---

## 受影響文件與更新策略

### CODEBASE_MAP.md — 需要更新（Main Agent）
- **原因**：
  - `src/llm/` 全新模組（16 個檔案）取代 `src/utils/clients.py`
  - `honcho-cli/` 新增頂層套件
  - `src/vector_store/utils.py` 刪除
- **影響段落**：目錄結構樹、快速查詢表
- **更新策略**：Main Agent 局部修改（新增目錄條目、更新快速查詢表）

### ARCHITECTURE.md — 需要大幅更新（Sub-Agent）
- **原因**：
  - LLM Client 層完全重設計（ProviderBackend ABC + 三個後端實作）
  - `ModelTransport` 取代 `SupportedProviders`（不再有 custom/vllm/groq）
  - Tool loop 移至 `src/llm/tool_loop.py`
  - Prompt caching 成為新功能
  - Dream scheduler 語義修正影響「核心設計決策」段落
  - `honcho-cli/` 新增為客戶端層元件
- **影響段落**：高層次架構圖、元件清單、LLM Client 段落、核心設計決策
- **更新策略**：Sub-Agent（多個段落 + 重繪 Mermaid 圖部分）

### INDEX.md — 需要更新（Main Agent）
- **原因**：
  - 技術棧：honcho-cli 新增；groq 依賴移除
  - 術語：`ModelTransport` 是新概念；`ThinkingEffortLevel` 是新型別
  - 關鍵指令：新增 honcho-cli 安裝和基本使用
- **影響段落**：技術棧表格、關鍵指令速查、術語表
- **更新策略**：Main Agent（追加/修改幾行）

### DEV_GUIDE.md — 需要更新（Main Agent）
- **原因**：
  - honcho-cli 是重要的開發/除錯工具，需要安裝和使用說明
  - ModelTransport 簡化（`custom`/`vllm`/`groq` provider 字串不再有效）
  - 「Groq 相容端點」FAQ 需更新
- **影響段落**：工具指南（新增 CLI 章節）、FAQ（更新 provider 相關問題）
- **更新策略**：Main Agent（新增 CLI 章節 + 修正 FAQ）

### DISCOVERY_LOG.md — 需要更新（Main Agent）
- **原因**：
  - D-003（DreamScheduler 語義）已在本次更新中修正
  - D-006（嵌入失敗重試）的向量同步部分已改進
  - 新增整合（SillyTavern、Paperclip、OpenCode、Vercel AI SDK）
  - 新發現：`src/llm/` 重構帶來更清晰的後端邊界
  - 新發現：Dialectic N+1 查詢修正
- **更新策略**：Main Agent（追加新段落，標記已修正項目）

### DATA_MODEL.md — 不需更新
- **原因**：`src/models.py` 在本次 diff 中未出現

### API_SURFACE.md — 不需更新
- **原因**：API 路由端點無結構性變更（僅 `src/schemas/api.py` 微小修改）

---

## 執行順序
1. `_context/` 更新（已完成：changelog.md）
2. CODEBASE_MAP.md
3. INDEX.md
4. ARCHITECTURE.md（Sub-Agent）
5. DEV_GUIDE.md
6. DISCOVERY_LOG.md
7. TRACE_META.md（最後）

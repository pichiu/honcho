# Stage 1 - 線上資源搜尋結果

## 搜尋摘要

共執行 4 次 web search + 1 次 web fetch，涵蓋專案文件、部落格文章和架構說明。

---

## 1. 官方資源

| 資源 | URL | 說明 |
|------|-----|------|
| GitHub | https://github.com/plastic-labs/honcho | 主要程式碼倉庫（本次 trace 對象） |
| 文件網站 | https://docs.honcho.dev | 官方開發者文件（v3） |
| 替代文件網站 | https://docs.honcho.to | 另一個文件入口 |
| 官方網站 | https://honcho.dev | 行銷網站 |
| 管理服務 | https://app.honcho.dev | 雲端 SaaS 版本 |
| Discord | https://discord.gg/honcho | 社群頻道 |

---

## 2. 部落格文章與關鍵 Takeaway

### 2.1 Honcho 3 發布公告
- **URL**: https://blog.plasticlabs.ai/blog/Honcho-3
- **關鍵 Takeaway**:
  - Honcho 3 是完整的系統重構，從單一代理轉為多代理架構
  - **Deriver**（記憶擷取）：已精簡為只處理「窮盡式明確資訊擷取」（exhaustive explicit information capture），速度更快、成本更低、完全平行化
  - **Dreamer**（記憶整合）：由新的「Dreaming Agent」系統接手原本 Deriver 的彙整工作，以異步方式在低計算成本時段執行
  - **Dialectic**（查詢）：不再走固定程式路徑，改為 agentic loop，將各種取回方法視為「工具」
  - 推理等級（reasoning levels）：minimal → low → medium → high → max，各有不同 provider 和成本
  - 定價：$2/百萬 token 輸入

### 2.2 Peer 典範介紹
- **URL**: https://blog.plasticlabs.ai/blog/Beyond-the-User-Assistant-Paradigm;-Introducing-Peers
- **關鍵 Takeaway**:
  - 將 User-Assistant 二元模型打破，任何實體（人類、AI、NPC、API）都是「Peer」，在系統中具有平等地位
  - 此設計使複雜的多 agent 互動成為可能（multi-player AI experiences）

### 2.3 Honcho 發布公告（初版）
- **URL**: https://blog.plasticlabs.ai/blog/Launching-Honcho;-The-Personal-Identity-Platform-for-AI
- **關鍵 Takeaway**:
  - Plastic Labs 獲得 $5.4M pre-seed 融資
  - 定位為「個人身份平台（Personal Identity Platform）」

### 2.4 基準測試
- **URL**: https://blog.plasticlabs.ai/research/Benchmarking-Honcho
- **關鍵 Takeaway**:
  - Honcho 聲稱定義了「Agent 記憶的 Pareto 前沿」
  - 官方評估頁面：https://evals.honcho.dev/

---

## 3. 架構文件（官方 v2 版本）

- **URL**: https://docs.honcho.dev/v2/documentation/core-concepts/architecture
- **關鍵 Takeaway**: v2 文件描述了 Peer 中心架構
  - Peers 是最重要的實體，代表工作空間中的個別用戶、代理或實體
  - 作為記憶和上下文管理的主要主題

---

## 4. 社群整合

- **Nous Research Hermes Agent 整合**: https://hermes-agent.nousresearch.com/docs/user-guide/features/honcho/
  - Hermes Agent 原生支援 Honcho 記憶

- **DeepWiki 記憶架構分析**: https://deepwiki.com/plastic-labs/nanobot-honcho/6.1-memory-architecture
  - 第三方對 Honcho 記憶架構的分析

- **Self-hosted 教學**: https://github.com/elkimek/honcho-self-hosted
  - 使用 OpenRouter + Venice 的自架版本

---

## 5. 發布記錄

- **Release Notes 04.17.25**: https://blog.plasticlabs.ai/releases/Release-Notes-04.17.25
  - 最近的版本更新

---

## 6. 主要技術發現

1. **三個 LLM Agent 分工**:
   - Deriver：快速擷取明確事實，完全平行化
   - Dreamer：深度彙整，演繹+歸納，在閒置時段執行
   - Dialectic：查詢時 agentic 推理，5 個推理等級

2. **向量儲存抽象化**:
   - 預設 pgvector（PostgreSQL 內建）
   - 可遷移至 Turbopuffer（雲端向量 DB）或 LanceDB（本地嵌入）
   - 背景 reconciler 負責同步

3. **Session-scoped 佇列**:
   - PostgreSQL QueueItem 表取代外部 message queue
   - 按 work_unit_key 確保同一 session 內的順序處理

4. **幾何驚訝度採樣（Surprisal Sampling）**:
   - 使用 kNN 樹（kdtree/balltree/rptree 等）計算觀察值的驚訝度
   - 高驚訝度的觀察值被優先用作 Dream specialist 的探索提示

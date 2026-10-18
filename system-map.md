<!-- system-map.md -->
# System Map — LLM Chess Agents (Groq ↔ Cloudflare)

## 0. Assumptions（明示假設）
- 先做 **CLI/純後端** 對局與回放（PGN + logs），不做網頁回放、不直接接 Lichess。
- 目標強度先不追求接近 Stockfish；第一優先是 **可重現、可監控、可驗收**，且 **走子 100% 合法**（由 Referee 強制保證）。
- 兩個外部模型供應商：**GroqCloud（免費模型）** 與 **Cloudflare Workers AI（免費模型）**，可用設定檔切換。
- Repo 以 GitHub 作為控制台：Actions 跑批次、Artifacts 存證據包、Issues/Projects 追蹤任務與異常。

> 若未來要改成 Web 回放或要 ELO/強度目標，會新增「UI service」與「評測基準」層；但不影響既有信任邊界。

---

## 1. System Context（系統在做什麼）
**Single job**：在可控與可驗證的條件下，讓兩個 LLM agent 進行 N 盤棋局，並輸出可重現的對局證據包（PGN + 全步 log + 指標報表），支援供應商/模型 A/B 測試。

**Primary outputs**
- `game.pgn`（可回放）
- `events.jsonl`（逐步事件與 I/O）
- `summary.json`（對局結果、錯誤分類、延遲、非法率等）
- （可選）`stockfish_eval.json`（事後品質評估）

---

## 2. Components（元件地圖）
### 2.1 Orchestrator（控制面 / 主編）
- 讀取 `match.yaml`（或等價 config）
- 建立對局（game_id、seed、初始 FEN）
- 每個 ply：
  - 由 Referee 取得 `legal_moves_uci`
  - 形成 `MoveRequest`
  - 呼叫當前 side 的 Agent（其背後會走對應 provider adapter）
  - 接收 `MoveResponse`，交給 Referee 審核
- 控制重試/超時/上限（max_ply、max_retries）
- 將所有事件寫入可追溯 log（evidence capsules）

### 2.2 Agents（執行面 / 作者）
- 僅做一件事：在收到 `MoveRequest` 後輸出 `MoveResponse.move_uci`
- 不允許自寫檔案/改設定（能力邊界由程式本身保證）
- Prompt/Policy 可演進，但不得突破 schema

### 2.3 Provider Adapters（供應商轉接層）
- Groq adapter：把 `MoveRequest` + prompt 送到 Groq Chat API（或等價），回收文字輸出
- Cloudflare adapter：把同樣的 request 送到 Workers AI
- 供應商差異（endpoint、auth、model naming、response shape）被封裝在此層
- 對上層暴露統一介面：`generate_move(request) -> raw_text + provider_meta`

### 2.4 Referee（審計面 / 零信任）
- 唯一真相來源（source of truth）
- 根據當前棋盤狀態生成 `legal_moves_uci`
- 驗證：
  - 是否為合法 UCI move
  - 是否在 legal list
  - 套用後 FEN/PGN 可重建
  - 終局判定（checkmate/stalemate/threefold/50-move/insufficient/material 等）
- 產出 `RefereeDecision`（accept/reject + reason）

### 2.5 Analyst（事後分析，可選）
- 以 Stockfish 對每步做評分、centipawn loss、分類 blunder/mistake 等
- 只讀 artifacts，不影響對局進行（避免把「品質」混進「合法性」）

### 2.6 Storage / Artifacts（證據包）
- 每盤棋至少輸出：
  - `request/PLY.json`
  - `response/PLY.json`
  - `referee/PLY.json`
  - `game.pgn`
  - `events.jsonl`
  - `summary.json`

---

## 3. Trust Boundaries（信任邊界）
- **不信任**：LLM 的任何自然語言敘述、理由、聲稱的棋盤狀態
- **信任**：
  - Referee 使用的棋規引擎（例如 python-chess / 等價）
  - Orchestrator 寫入的事件序列（可重播重建）
- **硬護欄**：
  - `max_ply` 防無限局
  - `max_retries_per_ply` 防 API 爆單
  - `time_budget_ms` 防卡死
  - `illegal_move_policy` 明確化（retry → forfeit / 或 fallback）

---

## 4. Data Flow（資料流）
### 4.1 Sequence（每個 ply）
1. Orchestrator → Referee：給當前 board state
2. Referee → Orchestrator：回 `legal_moves_uci` + game status
3. Orchestrator → Agent(side)：送 `MoveRequest`（含 fen + legal_moves_uci）
4. Agent → Provider Adapter → External Model：取回 raw output
5. Parser：抽出 `move_uci`（只接受 schema）
6. Orchestrator → Referee：提交 move
7. Referee：accept → 更新棋盤 / reject → 回 reason
8. Orchestrator：依 policy 重試/判負/或 fallback，並寫 event log
9. Loop 直到終局或觸發上限

### 4.2 External Dependencies
- GroqCloud API
- Cloudflare Workers AI API
- （可選）Stockfish binary（本地/Actions runner）

---

## 5. Observability（可觀測性 / 指標）
最低必備 metrics（每盤與全批次彙總）：
- `illegal_move_rate`（非法步 / 總步數）
- `parse_error_rate`
- `timeout_rate`
- `avg_latency_ms`、p95 latency
- `forfeit_count`（因非法/超時判負）
- `termination_reason` 分布（checkmate/draw/max_ply/forfeit）

Artifacts 可追溯：
- 任一盤任一 ply 可由 request+response+referee decision 重播出同一 PGN（可驗收條件）

---

## 6. GitHub as Console（GitHub 當控制台）
- Projects：Match jobs 狀態（Queued/Running/Needs human/Done）
- Issues：異常自動回報（含 artifact 連結、錯誤摘要）
- PR：prompt/config 變更必附回歸報告（同 seeds 的對局比較）
- Actions：
  - `workflow_dispatch`：手動跑一批 matches
  - `schedule`：nightly 小樣本 + leaderboard/report artifact
  - `pull_request`：回歸 suite（避免 prompt 改壞格式）

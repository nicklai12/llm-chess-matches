<!-- spec.md -->
# Spec — LLM Chess Match System (CLI-first)

## 1. Scope / Non-goals
### In scope
- 以設定檔驅動：跑 1 盤或 N 盤 LLM 對局
- 兩供應商可切換（Groq / Cloudflare Workers AI）
- Referee 保證 100% 合法走子（非法永不落盤）
- 產出可重現 artifacts（PGN + 逐步 I/O + summary 指標）
- 可在 GitHub Actions 無人值守跑批次並產出 artifacts

### Non-goals（現階段不做）
- Web UI / 動態回放頁
- 線上對戰平台整合（Lichess）
- 追求棋力到特定 ELO（只做事後分析指標）

---

## 2. Glossary
- **ply**：半回合（一方走一步 = 1 ply）
- **UCI move**：例如 `e2e4`、升變 `e7e8q`；王車易位用座標表示（如 `e1g1`）
- **FEN**：局面字串
- **PGN**：棋譜格式
- **Evidence capsule**：每一步的 request/response/referee decision 固化檔

---

## 3. Interfaces（核心介面）
### 3.1 Provider Interface（統一供應商介面）
**Input**：`MoveRequest`（見 4.1）與 prompt policy（由 agent 層決定）  
**Output**：
- `raw_text`（模型原始輸出）
- `provider_meta`：
  - `provider`（`groq` | `cloudflare`）
  - `model_id`
  - `latency_ms`
  - （若拿得到）`request_id` / `usage_tokens`

> Provider adapter 不做合法性判斷，不碰棋規。

### 3.2 Agent Interface
`decide_move(request: MoveRequest) -> MoveResponse`

Agent 必須：
- 僅輸出 schema 指定欄位
- `move_uci` 必須是單一決策（不得多步、不得 SAN）

### 3.3 Referee Interface
- `get_legal_moves(fen) -> legal_moves_uci[]`
- `apply_move(fen, move_uci) -> (new_fen, pgn_fragment, status)`
- `validate_move(fen, move_uci, legal_moves_uci) -> RefereeDecision`

---

## 4. Data Contracts（資料契約）
### 4.1 MoveRequest (JSON)
最小必備欄位（MUST）：
- `game_id: string`
- `ply: integer`（從 1 開始）
- `side: "white" | "black"`
- `fen: string`
- `legal_moves_uci: string[]`（當前局面所有合法 UCI）
- `time_budget_ms: integer`
- `policy_version: string`（prompt/policy 版本，用於回歸比較）

選填（MAY）：
- `seed: integer`
- `style: "solid" | "sharp" | "randomized"`（只做提示，不保證達成）

### 4.2 MoveResponse (JSON)
MUST：
- `move_uci: string`（必須為單一 UCI move）
- `provider: "groq" | "cloudflare" | "local"`
- `model_id: string`
- `latency_ms: integer`

MAY：
- `alt_moves: string[]`（最多 3 個）
- `rationale: string`（不作為真相來源）
- `raw_text: string`（建議保留，利於 debug）
- `parse_warnings: string[]`

### 4.3 RefereeDecision (JSON)
- `accepted: boolean`
- `reason: "ok" | "illegal_move" | "parse_error" | "timeout" | "game_over" | "internal_error"`
- `fen_before: string`
- `move_uci: string | null`
- `fen_after: string | null`
- `pgn_after: string | null`
- `status`：
  - `state: "ongoing" | "checkmate" | "stalemate" | "draw" | "forfeit" | "max_ply_reached"`
  - `winner: "white" | "black" | "draw" | null`
  - `detail: string`（例如 threefold/50-move）

---

## 5. Parsing Rules（輸出解析規格）
- 系統只接受 UCI move（`^[a-h][1-8][a-h][1-8][qrbn]?$`）
- 若模型輸出含多行文字，Parser 只取第一個符合 UCI regex 的 token
- 若找不到 token → `parse_error`
- `move_uci` 不在 `legal_moves_uci` → `illegal_move`

---

## 6. Policies（策略）
### 6.1 Retry Policy
`max_retries_per_ply`（例如 2）  
每次 retry MUST：
- 重送同一 fen 與同一 legal_moves_uci
- 明確提示「必須從 legal_moves_uci 選一個，僅回 UCI」

### 6.2 Illegal Move Policy（必須明確配置）
可選枚舉（先實作其中一種也可，但要寫死或配置擇一）：
- `retry_then_forfeit`（預設建議）
- `retry_then_fallback_random_legal`（保證對局可繼續，但要在 summary 記錄 fallback 次數）

### 6.3 Termination Policy
- `max_ply` 上限（例如 300）
- 到達上限 → `max_ply_reached`，判定為 draw（或配置）

---

## 7. Artifact Spec（證據包格式）
每局（game_id）輸出目錄（示意）：
- `games/<game_id>/`
  - `match_config.yaml`（實際生效 config 的快照）
  - `game.pgn`
  - `events.jsonl`（append-only）
  - `summary.json`
  - `ply/0001/`
    - `request.json`
    - `response.json`
    - `referee.json`
  - `ply/0002/` ...

`events.jsonl` 每行至少含：
- `ts`
- `game_id`
- `ply`
- `event_type`（request_sent/response_received/referee_accepted/referee_rejected/retry/terminated）
- `payload_ref`（指向檔案路徑或內嵌摘要）

---

## 8. Acceptance Criteria（可驗收標準）
### Milestone 1（Local random vs random）
- 跑 100 局能結束（checkmate/draw/max_ply）
- `illegal_move_rate = 0`
- 每局產出 `game.pgn` + `summary.json` + `events.jsonl`

### Milestone 2（接上 Groq & Cloudflare）
- 兩 provider 各自能完成至少 20 局（不要求棋力）
- `parse_error_rate < 5%`（初期）→ 迭代到 `< 1%`
- `timeout_rate < 2%`（或符合你的 runner 限制）
- 任一拒絕/判負必須在 summary 有可追溯原因（可從 artifacts 重建）

### Milestone 3（Config-driven switching）
- 只改 `match.yaml`（不改程式）即可切白方/黑方 provider/model
- 同 seed 的本地隨機策略可重現（至少在 local policy 下）

### Milestone 4（GitHub Actions nightly）
- nightly workflow 產出 artifacts（zip 或 Actions artifact）
- 異常（非法率/timeout 超標或 workflow fail）自動開 issue，附 artifacts 連結與摘要

---

## 9. Open Questions（待你決策）
- 對局呈現：只要 PGN/CLI，還是要加「最小回放頁」？
- `illegal_move_policy`：偏向直接 forfeit（嚴格），還是 fallback random legal（保證跑完、利於批次統計）？
- Stockfish：要不要納入 nightly（會增加 runner 負擔，但品質指標更清楚）？

<!-- README.md -->
# LLM Chess Match Runner (Groq ↔ Cloudflare) — CLI-first

用兩個可切換的 LLM agent 下西洋棋，**Referee 零信任審核**保證走子合法，輸出可回放 PGN 與完整證據包（逐步 request/response/referee decision），可在 GitHub Actions 夜間批次跑對局並產報表。

## What this repo is
- 一個「可監控、可驗收、可迭代」的對局系統骨架
- 專注在 **工程閉環**：重現性、可觀測性、明確錯誤分類、Artifacts 可追溯

## What this repo is not (yet)
- 不提供 Web UI 回放（先 CLI）
- 不追求棋力到特定 ELO（先把合法性與流程做硬）

---

## Quickstart (planned)
> 本段以「預期的使用方式」描述，等程式落地後會把 `TBD` 補成實際指令。

### 1) Prepare config
建立 `configs/match.yaml`（範例）：

    ```yaml
    match:
      game_count: 20
      seed: 42
      white:
        provider: groq
        model: llama-3.1-8b-instant
        temperature: 0.2
      black:
        provider: cloudflare
        model: "@cf/meta/llama-3-8b-instruct"
        temperature: 0.2

    rules:
      max_ply: 300
      max_retries_per_ply: 2
      illegal_move_policy: "retry_then_forfeit" # or retry_then_fallback_random_legal
      time_budget_ms: 8000
    ```

### 2) Set secrets (local / Codespaces / Actions)
- `GROQ_API_KEY=...`
- `CLOUDFLARE_ACCOUNT_ID=...`
- `CLOUDFLARE_API_TOKEN=...`

### 3) Run (TBD)
- `TBD: run one match batch from yaml`
- outputs:
  - `games/<game_id>/game.pgn`
  - `games/<game_id>/events.jsonl`
  - `games/<game_id>/summary.json`

---

## Repository layout (target)
    ```
    core/            # Referee, board state, rules, logging primitives
    agents/          # prompt policy + response parsing (schema-locked)
    providers/       # groq + cloudflare adapters (API differences encapsulated)
    configs/         # match.yaml examples
    evals/           # fixed seeds / regression match sets
    scripts/         # CLI entrypoints (thin)
    games/           # local artifacts (gitignored) OR generated in temp then archived
    .github/
      workflows/     # CI + nightly matches + report publishing
    docs/
      system-map.md
      spec.md
    ```

---

## Key design principles
### Zero-trust legality
- LLM 只「提議」move
- Referee 以客觀 legal move list 驗證
- 非法 move 永不落盤；處理由 `illegal_move_policy` 決定（retry/forfeit/fallback）

### Structured I/O (no freeform chat)
- 對 agent 的輸入是 `MoveRequest`（含 FEN + legal_moves_uci）
- agent 的輸出是 `MoveResponse`（核心欄位只有 `move_uci`）
- 所有 I/O 都落地成 artifacts，支援回放與 debug

### Evidence-first iteration (RSI via protocol)
- 迭代的主要槓桿：prompt/policy、parser/validator、重試策略、觀測指標
- 每次改動要用固定 seeds 的回歸對局驗證（Artifacts 作為證據包）

---

## Observability & reports
最低指標（每批次 summary 需含）：
- illegal_move_rate
- parse_error_rate
- timeout_rate
- avg/p95 latency
- termination_reason 分布（checkmate/draw/max_ply/forfeit）

（可選）Stockfish 事後分析：
- centipawn loss
- blunder/mistake/inaccuracy 次數

---

## GitHub Actions (planned)
- `pull_request`: regression matches（小樣本 + 固定 seeds）
- `workflow_dispatch`: 手動跑 N 盤
- `schedule`: nightly 跑小樣本、上傳 artifacts；若錯誤率超標自動開 issue

Artifacts 建議以 zip 打包上傳：
- `games/<game_id>/...` 全目錄
- `batch_summary.json`（本次批次彙總）

---

## Contribution workflow (intended)
- Prompt / policy / config 的變更一律走 PR
- PR 必須附：
  - 變更前後同一組 eval seeds 的 `batch_summary.json`
  - 至少 1 個可重播 game 的 artifacts（供人工 spot check）

---

## Docs
- `docs/system-map.md`：系統地圖、信任邊界、資料流
- `docs/spec.md`：資料契約、策略、artifact 規格、驗收標準

---

## Next decisions (you choose)
1) 你要採用哪個 `illegal_move_policy` 作為預設？
- 嚴格：`retry_then_forfeit`（資料乾淨，但對局可能較常中止）
- 連續性：`retry_then_fallback_random_legal`（保證跑完，便於大量統計）

2) nightly 要不要跑 Stockfish（多成本，但品質指標更清楚）？

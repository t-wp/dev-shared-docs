# 現在フェーズ

> **このファイルが日々の会話の主役。**
> ここだけ見れば現在地・論点・残タスクが分かるようにする。

---

## フェーズ: Step 6 — dry_run 安定化（安定稼働確認フェーズ）✅ **完了**
## 次フェーズ: Step 7 — 本番移行判断

> Step 6 完了: **2026-04-17**
> P1 異常なし・JPY 待機/再配分が設計意図どおりであることを確認。observation-log に根拠エントリ記録済み。

> Step 5（dry_run 観測・意図確認）も **2026-04-17 完了**。
> 主要確認項目（exit sell / rebalance sell / risk_budget 動的変化 / all-negative → JPY 全待機）は全て実観測済み。

---

## 現在の実装状態（前提）

| 機能 | 実装状態 |
|---|---|
| exit sell（MA/RSI シグナルによる退出） | 実装済み・稼働中 |
| rebalance sell（目標配分超過の是正） | 実装済み・稼働中（完全撤退も含む） |
| 強い銘柄への加重配分（MA 乖離率スコア） | 実装済み・稼働中 |
| risk budget（市場強度による総投入率の動的制御） | 実装済み・稼働中（breadth / top1 / total の3要素） |
| market_regime の記録（thinking.json） | 実装済み・記録中 |
| all-negative 時の JPY 全待機 | 実装済み・**観測済み（Step 5/6 で確認）** |

---

## このフェーズの論点

「直近に重大異常が出ていないか」の確認として、以下を押さえる。

1. P1 異常（peak_jpy 膨張 / virtual_jpy 負値 / alloc=0% 銘柄への BUY 誤発火 / ゼロ保有 SELL）が出ていないか
2. JPY 待機と再配分が設計意図どおりに動いているか（弱い時は JPY に寄る・強い行き先がある時は再配分）
3. risk_budget が相場状況に連動して自然に推移しているか（急変・固定化がないか）
4. Step 6 完了条件が揃ったと判断できるか

---

## 今の判断

| 確認項目 | 状態 |
|---|---|
| exit sell の発火 | ✅ 確認済み（2026-04-16 18:00 ADA デッドクロス） |
| rebalance sell の発火 | ✅ 確認済み（2026-04-16 17:40/17:50 ADA 段階的縮小） |
| risk budget の動的変化 | ✅ 確認済み（91.9% → 0% まで変化） |
| market_regime の記録 | ✅ thinking.json / JSON に保存済みを確認 |
| JPY が自然な待機資金として動くか | ✅ 確認済み |
| all-negative（全 JPY 待機）の実観測 | ✅ 確認済み（2026-04-16 20:30〜21:10 / 23:40〜00:20） |
| ADA 単独 72% 配分の異常 | 直近では再発なし（修正後） |

---

## Step 6 完了済みタスク

- [x] Step 5 の主要確認項目（exit sell / rebalance sell / risk_budget 動的変化 / all-negative → JPY 全待機）が全て実観測済み
- [x] 直近観測で P1 異常（peak_jpy 膨張・BUY 誤発火・ゼロ保有 SELL・virtual_jpy 異常）が出ていないことを確認
- [x] JPY 待機と再配分の挙動が設計意図どおりであることを確認
- [x] observation-log に根拠エントリを記録（2026-04-17）

**→ Step 6 全条件達成。Step 7 へ移行。**

---

## 完了済み

- ma_rsi 戦略 v2.0（short=5, long=25, RSI=14）dry_run 稼働開始
- 6銘柄対応（BTC/ETH/XRP/SOL/DOGE/ADA）、portfolio_history 記録
- exit_mode=full の config フラグ実装
- SELL 時リスクチェック免除（利確・損切りをブロックしない）
- market_prices 事前取得バグ修正（peak_jpy 5.4億膨張の根因解消、¥100K リセット済み）
- rebalance sell 実装（完全撤退 alloc=0% 含む、deadband / 最小売却額チェック付き）
- risk budget 層実装（breadth_weight / top1_weight / total_weight による動的総投入率）
- market_regime を thinking.json に記録

---

## 次フェーズへ進む条件

### 方針転換（2026-04-17）

Step 7（本番移行判断）を直接進める前に、**並行Dry-Run比較運用の体制構築**（Phase 5）を先行する。
理由: 本番移行前にパラメータの最適化基盤を整え、根拠を持って本番設定を決定するため。

**方針書:** https://github.com/t-wp/dev-shared-docs/blob/main/docs/crypto-bot/multi-run-policy.md

### 当面の進め方（2026-04-17 設計レビュー後に整理）

**Phase 5 本体の前に、BUY 抑制比較用 replay baseline を先行する（Phase 5 前段）。**

> この前段は Phase 5-1（bot.py 分解）の前置きであり、Phase 5 本体（slot×5 + docker + shared feeder）ではない。
> interval / cooldown / deadband の効果を portfolio-level replay で定量比較し、
> 1hour 足への移行判断と本番 param set の初期値を確定することが目的。

#### Phase 5 前段：BUY 抑制比較用 replay baseline の整備

| タスク | 内容 | 優先度 | 状態 |
|---|---|---|---|
| **①識別子付与** | replay 結果に `param_set_id` / `run_id` を持たせる。Param Set は不変スナップショット扱い | 高 | ✅ 完了（2026-04-17） |
| **②trace 拡充** | cooldown block 数・risk NG 数・min lot skip 数・alloc zero 数を記録。「なぜ BUY が減ったか」を分離できるようにする | 高 | ✅ 完了（2026-04-17） |
| **③best 選定の修正** | `final_portfolio` 単独選抜 → Gate 条件先行 + 複合スコアに変更。`select_best()` 関数を実装 | 高 | ✅ 完了（2026-04-17） |
| **④config スナップショット参照** | `ReplayConfig` のデフォルト値を `config.yaml` から読み込むか、param_set として明示的に渡す構造にする | 中 | 未着手 |

> ①〜③ 完了。追跡可能な比較実験として成立する状態になった。

#### Phase 5 本体への移行条件

以下が全て揃ったら Phase 5-1（bot.py 分解）へ進む。

- [x] 1hour 足への移行判断が完了（replay 結果で根拠あり）→ **2026-04-17 config.yaml を 1hour に変更・デプロイ済み**
- [ ] 本番初期 param_set の候補が確定（Gate 条件先行で選定済み）— 暫定: 1hour/cd0/db0.15 (ps-012, Gate 3/4)
- [x] replay が param_set_id / run_id で追跡可能になっている

Step 7（本番移行判断）は Phase 5 の運用が軌道に乗った段階で改めて判断する。

---

## Step 6 の観測設計

**位置づけ:** 長期観測フェーズではなく、**安定稼働確認の最終整理フェーズ**。
日数ではなく「必要なケースが踏めていて、直近に重大異常が出ていない」ことをもって完了とする。

### 観測の合否基準

| 区分 | 内容 |
|---|---|
| **即確認 P1（1件でも出たら要対処）** | peak_jpy 膨張 / virtual_jpy 負値 / alloc=0% 銘柄への BUY 誤発火 / ゼロ保有 SELL |
| **週次追跡 P2（傾向を見る）** | 連敗停止の特定銘柄集中 / 日次損失停止の頻発 / rebalance sell 過剰連発（収束しない） / alloc_pct 合計が risk_budget 超過 |
| **都度観察 P3（方向性確認）** | risk_budget の連続サイクル間急変（50%pt 超）/ JPY が相場強弱と逆相関しているか / all-negative 復帰後の BUY 再開 |

### Step 6 完了条件（条件ベース）

日数基準は設けない。以下の条件が揃った時点で完了とみなす。

- [x] Step 5 の主要確認項目（exit sell / rebalance sell / risk_budget 動的変化 / all-negative → JPY 全待機）が全て実観測済み
- [ ] 直近観測で P1 異常（peak_jpy 膨張・BUY 誤発火・ゼロ保有 SELL・virtual_jpy 異常）が出ていない
- [ ] JPY 待機と再配分の挙動が設計意図どおり（弱い時は JPY に寄る・強い行き先がある時は再配分される）
- [ ] observation-log に上記の根拠エントリが記録されている

### Step 6 中はやらないこと

| やらないこと | 判断タイミング |
|---|---|
| パラメータ最適化（short / long / rsi 期間等） | Step 9 |
| `top1_ref` / `total_ref` の調整 | Step 6 完了後、根拠を持って行う |
| `breadth_weight` / `top1_weight` / `total_weight` のチューニング | 同上 |
| negative penalty / smoothing の導入 | Step 6 では設計変更を持ち込まない |
| 1hour 足への変更 | Step 6 完了後に判断 |
| 本番（live）移行の検討 | Step 7 のチェックリストが揃うまで行わない |

---

## 今はやらないこと

→ Step 6 観測設計の「Step 6 中はやらないこと」に統合済み。

---

## 設計レビュー所見（2026-04-17）

`portfolio_replay.py` と `bot.py` を正本 docs と突き合わせたレビューの結果。

### 危険な点（severity 順）

| # | 分類 | 内容 | 根拠 |
|---|---|---|---|
| 1 | **評価軸ズレ** | `best = max(results, key=lambda r: r.final_portfolio)`（replay.py:809）— Gate 判定は表示のみで選抜に不使用。短期観測では aggressive な設定を拾うリスクがある | portfolio_replay.py:809 |
| 2 | **追跡性なし** | `run_id` / `experiment_id` / `param_set_id` が replay にも bot.py にも未実装。成果物単体で「どの param set のものか」が分からない。multi-run-policy の設計思想と矛盾 | bot.py・portfolio_replay.py 全体 |
| 3 | **原因分離不可** | trace が executed trade の集計のみ。cooldown block 数・risk NG 数・min lot skip 数・alloc zero 数が残らないため「なぜ BUY が減ったか」を分離できない | portfolio_replay.py:RunResult |
| 4 | **config 非参照** | `ReplayConfig` にデフォルト値直書き。`config.yaml` 変更が replay に自動反映されない。param set 不変思想とも相性が悪い | portfolio_replay.py:40-95 |
| 5 | **スコープ二重化** | docs は「Phase 5 先行」のまま、実装は「BUY 抑制比較 replay」に寄っている。意図的か事故かが読み取れない状態だった → 上の「当面の進め方」で明文化して解消 | ドキュメント vs コード |

### 確認された事実（replay 実測、2026-04-17）

- 10min 足は全パターン損失。1hour 足のみ全パターン利益（+¥2,772〜+¥3,381）
- deadband 拡大だけでは不十分。BUY がサイクルの 81% で発火しており、これが高 turnover の主因
- 1hour/cd0/db0.15 が Gate 先行 + 複合スコアによるベスト（ps-012, Gate 3/4）
- 残る Gate NG は「週片側 turnover ≤ 500%」のみ。さらなる対策が必要
- **2026-04-17 config.yaml を 1hour に変更・NAS にデプロイ済み。データ蓄積開始**

### trace 内訳（2026-04-17 replay 追加実装による新知見）

- 10min 足で `risk_ng` が 764 回（ボラティリティチェック主因）。1hour 足では 0〜13 回に激減
- 1hour + deadband 拡大で主なスキップ原因は `min_lot_skip`（gap が lot 刻み未満）。正常動作
- `alloc_zero` はほぼ 0。配分ゼロによるスキップは稀

---

## 未決事項

| # | 論点 | 状態 |
|---|---|---|
| 1 | ~~10min 足はローテーション bot として適切か~~ | **解決済み: 2026-04-17 に 1hour へ移行・デプロイ完了** |
| 2 | risk_budget 参照値（top1_ref=0.010 / total_ref=0.030）は実態に合っているか | 保留中（観測後に更新予定） |
| 3 | rebalance_min_jpy=500 は適切か（小額すぎる、または大きすぎないか） | 保留中 |
| 4 | Gate 残 NG（週片側 TO > 500%）をどう解消するか | 保留中（BUY 条件追加 or 銘柄数削減 or TO 閾値見直し） |

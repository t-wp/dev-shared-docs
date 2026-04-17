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

### 当面の進め方

1. **Phase 5-0（設計）**: 現行値確定表の作成、評価指標の具体値確定
2. **Phase 5-1（分解）**: bot.py から Signal Model / Portfolio Policy / Param Set を構造的に分離
3. Phase 5-2 以降は 5-1 完了後に着手

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

## 未決事項

| # | 論点 | 状態 |
|---|---|---|
| 1 | 10min 足はローテーション bot として適切か（短すぎないか） | 保留中 |
| 2 | risk_budget 参照値（top1_ref=0.010 / total_ref=0.030）は実態に合っているか | 保留中（観測後に更新予定） |
| 3 | rebalance_min_jpy=500 は適切か（小額すぎる、または大きすぎないか） | 保留中 |

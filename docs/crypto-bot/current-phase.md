# 現在フェーズ

---

## フェーズ: Step 5 — dry_run 観測（意図確認）

**目的:**
「設計どおりに動いているか」を確認する。儲かるかどうかはまだ問わない。

---

## このフェーズの論点

確認したいのは以下の順序:

1. exit sell が MA/RSI シグナルで正しく発火しているか
2. rebalance sell が目標配分超過に対して発火しているか（現在は実装中か未実装か要確認）
3. BUY 時に「強い銘柄へ厚く配分」の意図が allocation に反映されているか
4. JPY 残高が「待機資金」として自然に増減しているか（主役になっていないか）
5. risk budget 層（Phase 4）は未実装。現状は固定投資率で動いている

---

## 今の判断

| 項目 | 状態 |
|---|---|
| exit sell の発火確認 | 観測中 |
| rebalance sell の分離 | 未実装（Phase 2 の対象） |
| allocation の強弱反映 | 部分的に実装済み（ma_rsi スコアベース） |
| risk budget 制御 | 未実装（Phase 4 の対象） |
| dry_run 安定稼働 | 継続中（10min 足） |

---

## 残タスク（このフェーズで詰めること）

- [ ] exit sell がログ上で確認できる形になっているか確認
- [ ] rebalance sell の発火ロジックが存在するか確認
- [ ] SELL 時にログに種別（exit/rebalance）が付いているか確認
- [ ] 「設計どおりに動く」と判定できる観測エントリを observation-log.md に積む

---

## 完了済み

- ma_rsi 戦略 v2.0 の実装と dry_run 稼働開始
- 6銘柄対応・portfolio_history 記録
- exit_mode=full の config フラグ実装
- risk check の SELL 免除（利確・損切りをブロックしない）
- market_prices 事前取得バグ修正（peak_jpy 膨張の根因解消）
- risk_budget セクションを config.yaml に追記（全て仮値）

---

## 次フェーズへ進む条件

以下が揃えば Step 6（dry_run 安定化）または Phase 2 開発へ進む:

- exit sell の発火を観測ログで確認済み
- rebalance sell の実装方針が決まっている
- 異常な state.json 値（peak_jpy 膨張など）が再発していない

---

## 今はやらないこと

- パラメータのチューニング → Step 9 で行う
- 本番移行の検討 → Step 7 の判断条件が揃うまで行わない
- 1hour 足への変更 → フェーズ外（将来拡張候補）
- risk budget の実装 → Phase 4 の対象

---

## 未決事項

| # | 論点 | 状態 |
|---|---|---|
| 1 | rebalance sell のロジックは現状実装されているか | 保留中 |
| 2 | 10min 足はローテーション bot として適切か（短すぎないか） | 保留中 |
| 3 | SELL ログに種別を付けるのは Phase 2 の作業か、今すぐ確認できるか | 保留中 |

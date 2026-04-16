# 現在フェーズ

> **このファイルが日々の会話の主役。**
> ここだけ見れば現在地・論点・残タスクが分かるようにする。

---

## フェーズ: Step 5 — dry_run 観測（意図確認）✅ **完了**
## 次フェーズ: Step 6 — dry_run 安定化（移行判断中）

**Step 5 目的（達成）:**
「設計どおりに動いているか」を確認する。**儲かるかどうかはまだ問わない。**

**Step 6 目的:**
「異常なく継続して動き続けるか」を確認する。設計意図の確認（Step 5）から、**安定稼働の確認**へ移る。

---

## 現在の実装状態（前提）

| 機能 | 実装状態 |
|---|---|
| exit sell（MA/RSI シグナルによる退出） | 実装済み・稼働中 |
| rebalance sell（目標配分超過の是正） | 実装済み・稼働中（完全撤退も含む） |
| 強い銘柄への加重配分（MA 乖離率スコア） | 実装済み・稼働中 |
| risk budget（市場強度による総投入率の動的制御） | 実装済み・稼働中（breadth / top1 / total の3要素） |
| market_regime の記録（thinking.json） | 実装済み・記録中 |
| all-negative 時の JPY 全待機 | 実装済み・**観測済み** |

---

## このフェーズの論点

「設計どおりに動いているか」の確認として、以下を順に押さえる。

1. exit sell が MA/RSI シグナルと連動して発火しているか
2. rebalance sell が配分超過銘柄に対して発火・または完全撤退しているか
3. risk budget が市場強度に応じて実際に変化しているか（固定率ではないか）
4. 強い銘柄に厚く配分 → 弱い銘柄は alloc 0% → rebalance sell / BUY スキップ、の流れが機能しているか
5. JPY 残高が「行き先がない時に増える」という動きをしているか
6. all-negative シナリオ（全銘柄下降 → JPY 全待機）がいつ初観測できるか

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

## 残タスク（このフェーズで詰めること）

- [x] exit sell と rebalance sell が observation-log にエントリとして記録されているか確認
- [x] risk budget が実際に変化しているサイクルを observation-log に記録する
- [x] all-negative が実際に発生したときに JPY 待機に切り替わることを確認する
- [x] 「設計どおりに動く: Yes」と判定できるエントリを observation-log に複数積む

**→ 全タスク完了。Step 6（dry_run 安定化）への移行を検討する段階。**

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

Step 6（dry_run 安定化）へ進む前提:

- [x] exit sell・rebalance sell それぞれの発火を観測ログで確認済み
- [x] risk budget が市場強度に応じて変化していることを観測ログで確認済み
- [x] all-negative → JPY 待機の切り替えを少なくとも 1 回観測済み
- [x] state.json の異常値（peak_jpy 膨張など）が再発していない

**→ 全条件達成。Step 6 へ移行可能。**

## Step 6 の目的と最初の確認項目

**目的:** 設計どおりに動くことが確認された状態で、**一定期間異常なく継続稼働できるか**を確認する。

最初に確認すること:
- 異常値（peak_jpy 膨張・virtual_jpy 急落・連敗停止など）が一定期間（目安: 1週間）発生しないか
- state.json / thinking.json が正常な値の範囲内で推移し続けるか
- サイクルごとのシグナル・配分・売買が引き続き設計意図に沿っているか

---

## 今はやらないこと

| やらないこと | 理由 |
|---|---|
| パラメータのチューニング | Step 9 の作業。今は「動くか」を確認する段階 |
| 1hour 足への変更 | フェーズ外（将来拡張候補）。今は 10min 足で観測継続 |
| risk_budget 参照値（top1_ref / total_ref）の最適化 | 現在は仮値での挙動確認が目的 |
| 本番移行の検討 | Step 7 のチェックリストが揃うまで行わない |
| open-questions.md の作成 | 未決事項が 5 件以上になってから分離する |

---

## 未決事項

| # | 論点 | 状態 |
|---|---|---|
| 1 | 10min 足はローテーション bot として適切か（短すぎないか） | 保留中 |
| 2 | risk_budget 参照値（top1_ref=0.010 / total_ref=0.030）は実態に合っているか | 保留中（観測後に更新予定） |
| 3 | rebalance_min_jpy=500 は適切か（小額すぎる、または大きすぎないか） | 保留中 |

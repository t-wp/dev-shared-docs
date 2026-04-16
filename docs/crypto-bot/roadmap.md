# crypto-bot ロードマップ

> **全体地図。今どこにいるかは** https://github.com/t-wp/dev-shared-docs/blob/main/docs/crypto-bot/current-phase.md **を参照。**

---

## 設計思想（サマリ）

詳細は https://github.com/t-wp/dev-shared-docs/blob/main/.github/copilot-instructions.md を参照。

このbotは **銘柄ローテーションbot**。強い銘柄へ資金を移し、弱い銘柄から撤退する。
「誰に配るか（allocation）」と「市場に何%出すか（risk budget）」を分けて制御することが最終形。

---

## 開発〜運用の全体フロー

| Step | 名前 | 完了の目安 |
|---|---|---|
| 1 | 設計思想の固定 | copilot-instructions.md に記載済み ✅ |
| 2 | ロジック設計 | 仕様ドキュメントがある ✅ |
| 3 | ローカル実装 | コードが動く ✅ |
| 4 | ローカル検証 | バックテスト・単体テスト通過 ✅ |
| **5** | **dry_run 観測（意図確認）** | **現在ここ ← 「設計どおりに動く」を文書で判定できる** |
| 6 | dry_run 安定化 | 異常なし期間が一定以上続く |
| 7 | 本番移行判断 | チェックリスト全通過 |
| 8 | 本番初期運用 | 小ロット・監視強化期間 |
| 9 | チューニング | パラメータ最適化（設計は変えない） |
| 10 | 継続改善 | 定期レビューサイクル |

> **Step 5 の目的は「儲かるか」ではなく「設計意図どおりか」の確認。**
> チューニング（Step 9）はその後に行う。この順序は崩さない。

---

## 機能開発フェーズ

全体フローの中で「ロジックをどう育てていくか」の開発段階。
**Phase 2・4 は既に実装・dry_run 観測中。** 以下はその定義と完了条件の記録。

---

### Phase 1 — 現状ロジックの可視化 ✅ 完了

**目的:** 今何が動いていて何が動いていないかを明確にする。

完了内容:
- exit sell（MA/RSI シグナル）が稼働していることを確認
- rebalance sell が実装・稼働していることを確認
- 各銘柄のシグナル・配分・sell_size がログで追える状態になった

---

### Phase 2 — exit sell と rebalance sell の分離整理 ✅ 実装済み・観測中

**目的:** SELL を 2 種に正式分離し、それぞれが独立して発火・ログに残ること。

完了条件（実装は済、観測継続中）:
- [x] exit sell がトレンド転換時に発火する
- [x] rebalance sell が目標配分超過または alloc=0% の銘柄に発火する
- [x] SELL ログに種別（exit / rebalance）が区別して記録される
- [ ] dry_run 上で双方の発火を observation-log に記録する（観測中）

---

### Phase 3 — 配分ロジックの再設計 ✅ 実装済み・観測中

**目的:** allocation（誰に配るか）を銘柄の強さに応じて動的に変更する構造。

完了条件（実装は済、観測継続中）:
- [x] 銘柄ごとの MA 乖離率スコアが計算される
- [x] スコアに基づいた目標配分がサイクルごとに算出される
- [x] 下降銘柄は alloc=0%（完全撤退 rebalance の対象）
- [ ] 強い銘柄へ厚く配分される流れを observation-log に記録する（観測中）

---

### Phase 4 — market regime / risk budget の導入 ✅ 実装済み・観測中

**目的:** 「市場全体に何%出すか」をシグナルで動的に変える。

完了条件（実装は済、観測継続中）:
- [x] breadth（positive 銘柄数）・top1 強度・positive 強度の総量を計算
- [x] 上記 3 要素から risk budget（0.0〜max_budget）を算出
- [x] risk budget が allocation と掛け合わさって実際の発注量に反映される
- [x] market_regime が thinking.json に記録される
- [ ] all-negative → JPY 全待機のシナリオを実観測する（未達）

参考: config.yaml の `risk_budget` セクション（breadth_weight / top1_weight / total_weight / top1_ref / total_ref）は全て**仮値**。実測後に更新予定。

---

### Phase 5 — 検証指標と評価方法の固定化

**目的:** チューニング前に「何をもって良しとするか」の基準を決める。

完了条件:
- 評価指標（手数料込み損益・最大 DD・連敗数・取引回数など）が定義される
- バックテスト・dry_run それぞれの評価方法が決まっている
- 「設計どおり動いているか」と「効率がよいか」を分けて評価できる

---

## 将来拡張候補（保留中）

- 1hour 足への変更（現在 10min はテスト用。ローテーション bot としては足が短すぎる可能性あり）
- risk_budget 参照値（top1_ref / total_ref）の調整（観測後に根拠を持って行う）
- rebalance_min_jpy の調整（小額閾値の妥当性確認後）

# crypto-bot ロードマップ

> **全体地図。今どこにいるかは** `current-phase.md` **を参照。**
> **具体的な実務タスクは** GitHub Issues **で管理する。**

---

## 設計思想（サマリ）

詳細は `copilot-instructions.md` を参照。

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
| 5 | dry_run 観測（意図確認） | ✅ **完了**（2026-04-17）— exit sell / rebalance sell / risk_budget 動的変化 / all-negative → JPY 全待機、全項目観測済み |
| 6 | dry_run 安定化 | ✅ **完了**（2026-04-17）— P1 異常なし・設計意図どおりの挙動を確認 |
| **7** | **本番移行判断** | **Phase 5（並行Dry-Run比較運用）を先行中 ← 現在ここ** |
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

### Phase 2 — exit sell と rebalance sell の分離整理 ✅ 完了

**目的:** SELL を 2 種に正式分離し、それぞれが独立して発火・ログに残ること。

完了条件（実装は済、観測終了）:
- [x] exit sell がトレンド転換時に発火する
- [x] rebalance sell が目標配分超過または alloc=0% の銘柄に発火する
- [x] SELL ログに種別（exit / rebalance）が区別して記録される
- [x] dry_run 上で双方の発火を observation-log に記録する（2026-04-16 ADA にて観測済み）

---

### Phase 3 — 配分ロジックの再設計 ✅ 完了

**目的:** allocation（誰に配るか）を銘柄の強さに応じて動的に変更する構造。

完了条件（実装は済、観測終了）:
- [x] 銘柄ごとの MA 乖離率スコアが計算される
- [x] スコアに基づいた目標配分がサイクルごとに算出される
- [x] 下降銘柄は alloc=0%（完全撤退 rebalance の対象）
- [x] 強い銘柄へ厚く配分される流れを observation-log に記録する（2026-04-16 18サイクル観測済み）

---

### Phase 4 — market regime / risk budget の導入 ✅ 完了

**目的:** 「市場全体に何%出すか」をシグナルで動的に変える。

完了条件（実装は済、観測終了）:
- [x] breadth（positive 銘柄数）・top1 強度・positive 強度の総量を計算
- [x] 上記 3 要素から risk budget（0.0～max_budget）を算出
- [x] risk budget が allocation と掛け合わさって実際の発注量に反映される
- [x] market_regime が thinking.json に記録される
- [x] all-negative → JPY 全待機のシナリオを実観測済み（2026-04-16 20:30～21:10 / 23:40～00:20）

参考: config.yaml の `risk_budget` セクション（breadth_weight / top1_weight / total_weight / top1_ref / total_ref）は全て**仮値**。実測後に更新予定。

---

### Phase 5 — 並行Dry-Run比較運用の構築

**目的:** 複数の設定値セットを並行して比較し、最適な組み合わせを継続的に探索する体制を作る。

**方針書:** `multi-run-policy.md`

**運用上の注意:**
- 現在は **Phase 5 前段** を先行中
- 具体作業は Issue で追う
- 親 Issue: https://github.com/t-wp/crypt-bot/issues/1

サブフェーズ:

| Sub-Phase | 内容 |
|---|---|
| **5-0. 設計** | 識別子体系確定、評価指標定義、探索パラメータ軸確定、必須観測項目固定、昇格ゲート条件明文化、現行値確定表の作成 |
| **5-1. 分解** | bot.py から Signal Model / Portfolio Policy / Param Set を構造的に分離。差し替え可能なリファクタ |
| **5-2. データ共有** | Market Data Feeder 分離。共有ストア設計。Prod フォールバック実装 |
| **5-3. 並行Dry-Run** | docker-compose で slot×5 + prod。独立ログ/state。experiment_id/run_id 管理 |
| **5-4. 比較ダッシュボード** | 全 slot の成績並列表示。複合スコアランキング。ゲート条件合否表示 |
| **5-5. 運用開始** | 探索サイクル開始。週次入れ替え。手動昇格判断 |
| **5-6. モデル多様化** | trend/range/defensive 等、Signal Model や Policy 自体を複数用意（十分な運用知見を得てから） |

完了条件:
- [ ] 5台の Dry-Run slot が同一初期条件・同一市場データで並行稼働している
- [ ] 週次で成績比較し、slot 入れ替えが実行できる
- [ ] 昇格ゲート条件を満たした param_set を本番に適用するフローが確立している

---

## Phase 5 前の最適化タスク

- `candle_days` 3→1: `get_klines` の冗長取得削減（Issue #8）
- Dashboard API呼び出し削減: 並行化前の負荷低減（Issue #9）

---

## Issue 運用

- 現在地と移行条件: `current-phase.md`
- 実務タスク一覧: https://github.com/t-wp/crypt-bot/issues
- Phase 5 前段の親 Issue: https://github.com/t-wp/crypt-bot/issues/1

---

## 将来拡張候補（保留中）

- ~~1hour 足への変更~~ → **実施済み（2026-04-17）**: replay で 1hour が唯一の利益足であることを確認し、config.yaml を変更・デプロイ
- risk_budget 参照値（top1_ref / total_ref）の調整（観測後に根拠を持って行う）
- rebalance_min_jpy の調整（小額閾値の妥当性確認後）

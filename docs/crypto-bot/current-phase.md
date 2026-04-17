# 現在フェーズ

> **このファイルが日々の会話の主役。**
> ここでは **現在地・今フェーズの目的・最優先タスク・移行条件** だけを扱う。
> 詳細な観測結果は `observation-log.md`、実務タスクは GitHub Issues を参照する。

---

## 現在地

- **Step 6 — dry_run 安定化**: ✅ 完了（2026-04-17）
- **次判断**: Step 7 — 本番移行判断
- **ただし現実の進行**: Step 7 を直接進めず、**Phase 5 前段** を先行中

> Step 6 完了: P1 異常なし・JPY 待機/再配分が設計意図どおりであることを確認済み。
> 根拠は `observation-log.md` に記録済み。

---

## 今フェーズの目的

**Phase 5 本体の前に、BUY 抑制比較用 replay baseline を整備する。**

- 1hour 足への移行判断を replay と実観測の両方で固める
- replay を「追跡可能な比較実験」として成立させる
- 本番初期 param_set 候補を選べる状態にする
- Phase 5-1（bot.py 分解）へ進む条件を明文化する

> この前段は Phase 5 本体（slot×5 + docker + shared feeder）ではない。
> まずは replay 比較の品質と 1hour 観測の継続を優先する。

---

## 現在の判断

- **1hour 足への移行**: 実施済み（2026-04-17 `config.yaml` 更新・デプロイ済み）
- **replay 主要改修 ①〜③**: 完了
- **残る高優先**: ④config スナップショット参照 / 本番初期 param_set 候補の確定 / Gate 残 NG の整理
- **最大のボトルネック**: データ不足。1hour 観測の継続が必要

---

## Phase 5 前段の状態

| 項目 | 状態 | 関連 Issue |
|---|---|---|
| ① 識別子付与 | ✅ 完了（2026-04-17） | 実装済み |
| ② trace 拡充 | ✅ 完了（2026-04-17） | 実装済み |
| ③ best 選定修正 | ✅ 完了（2026-04-17） | 実装済み |
| ④ config スナップショット参照 | 未着手 | #2 |
| 1hour dry_run 観測継続 | 進行中 | #3 |
| 本番初期 param_set 候補の確定 | 未完 | #4 |
| Gate 残 NG の解消案整理 | 未着手 | #5 |
| docs 整理 | 進行中 | #6 |
| Phase 5-1 着手条件の明文化 | 未着手 | #7 |

---

## 最優先 Issue

- Epic: https://github.com/t-wp/crypt-bot/issues/1
- config スナップショット参照: https://github.com/t-wp/crypt-bot/issues/2
- 1hour dry_run 観測継続: https://github.com/t-wp/crypt-bot/issues/3
- 本番初期 param_set 候補の確定: https://github.com/t-wp/crypt-bot/issues/4
- Gate 残 NG の解消案整理: https://github.com/t-wp/crypt-bot/issues/5
- docs 整理: https://github.com/t-wp/crypt-bot/issues/6
- Phase 5-1 着手条件の明文化: https://github.com/t-wp/crypt-bot/issues/7

---

## Phase 5 本体への移行条件

以下が揃ったら Phase 5-1（bot.py 分解）へ進む。

- [x] 1hour 足への移行判断が完了
- [x] replay が `param_set_id` / `run_id` で追跡可能
- [ ] 本番初期 `param_set` 候補が確定
- [ ] Phase 5-1 着手条件が docs / issue で一致

---

## 詳細の置き場

- 観測結果の詳細: `observation-log.md`
- 並行 Dry-Run 比較運用の原則: `multi-run-policy.md`
- 全体地図とフェーズ定義: `roadmap.md`
- 実務タスク一覧: GitHub Issues https://github.com/t-wp/crypt-bot/issues

---

## 未決事項

| # | 論点 | 状態 |
|---|---|---|
| 1 | risk_budget 参照値（`top1_ref=0.010` / `total_ref=0.030`）は実態に合っているか | 保留中（観測後に更新予定） |
| 2 | `rebalance_min_jpy=500` は適切か | 保留中 |
| 3 | Gate 残 NG（週片側 TO > 500%）をどう解消するか | 保留中 |
| 4 | 2-slot 比較運用をどの構成で先行導入するか | 保留中 |

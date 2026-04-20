# 現在フェーズ

> **このファイルは司令塔。**
> ここでは **現在地・目的・最優先課題・移行条件** だけを扱う。
> 詳細な観測結果は `observation-log.md`、実務タスクは GitHub Issues を参照する。

---

## 現在地

- **Step 6 — dry_run 安定化**: ✅ 完了
- **現在の実務フェーズ**: Step 7 本番移行判断のための 1hour dry_run 観測フェーズ
- **完了済みの土台整備**: cadence 修正、Dashboard Logs 整理、候補 interval の動作確認、2-slot 比較基盤の設計・実装・起動確認
- **2-slot 比較基盤**は champion / challenger の最小比較構成として成立確認済み
  - manifest 差分確認済み
  - /api/slots 確認済み
  - slot-2 cold-start の read-only 失敗モード確認済み
  - shared DB 作成後の自動復帰確認済み

---

## 今フェーズの目的

- 1hour 運用の初期観測データを整理する
- replay の見立てと実観測のズレを確認する
- 本番初期 `param_set` 判断に必要な論点を絞る

---

## 最優先課題

1. **#3 1hour dry_run 観測を継続する**
  - turnover
  - BUY / SELL 件数
  - skip 理由
  - replay との差分

2. **本番移行判断に必要な差分を整理する**
  - replay では良いが実運用では弱い点があるか
  - 逆に実運用でのみ見えるボトルネックがあるか

3. **観測上の新しい論点を評価する**
  - ~~min_lot スキップが多いことをどう評価するか~~ → **#21 で評価完了。資金スケール問題ではない（rebalance 離散性 + alloc=0% + 配分充足の 3 分類）。ログ分類バグも修正済み（90d6a99）**
  - all-negative 比率が高いことをどう評価するか → #22 で評価中

---

## 現在の判断

- **1hour 足への移行**: 実施済み
- **replay 側の主要改修**: 実施済み
- **本番初期 `param_set` 候補**: replay では primary 候補あり
- **2-slot 比較基盤**: 最小構成で成立確認済み
- **ローソク足データ欠損 WARN**:
  - 4/17〜4/19 の定常 1hour 運用では再発 0 件
  - interval 切替時の過渡現象だった可能性が高い
  - 定常運用の判断材料としては大きな阻害要因ではなくなった
- **1hour 実観測**:
  - 初期傾向を整理できるだけの材料は揃った
  - min_lot スキップは #21 で再分類済み（資金スケール問題ではない）
  - 本番昇格の最終判断に残る論点は **all-negative 比率の評価（#22）のみ**

---

## Phase 5-1 へ進む条件

- [x] cadence / restart / interval 動作が安定している
- [x] replay が追跡可能な比較実験として成立している
- [x] 本番初期 `param_set` 候補が replay で選定できている
- [x] 1hour 実観測の初期傾向が整理されている
- [x] ローソク足データ欠損 WARN が定常運用の阻害要因ではないと確認できている
- [x] 本番判断に向けて残論点が all-negative 評価（#22）のみに絞り込まれている（#21 完了済み）

---

## Phase 5-1 実装開始順

Phase 5-1 は大規模 modularization ではなく、**正本固定と境界定義** から最小コストで着手する。
詳細な定義は `multi-run-policy.md` Section III-A〜III-D を参照。

**前提: LogLayout 移行完了（2026-04-20）**
- bot.py / dashboard.py ともに LogLayout 経由で records/ を正本として読み書き
- 旧フラットファイル（state.json, thinking.json, events.json, bot_*.log 等）は全削除済み
- log_layout.py のフォールバックコード（`_resolve_read()`, `migrate_from_flat()`）も削除済み

| 順 | 内容 | Issue | 対応セクション |
|---|---|---|---|
| 1 | **param_set 正本固定** — `param_sets/ps-001.yaml` / `ps-002.yaml` を正本フォーマットで作成。config.yaml から可変値を分離する | #25 | III-A 正本フォーマット / III-B 責務分離 |
| 2 | **param_set 読み込み + resolved manifest 保存** — bot.py 起動時に `--param-set` で読み込み、config とマージし、resolved manifest を slot ディレクトリに出力する | #25, #23 | III-C 起動時の解決順 |
| 3 | **Portfolio Policy interface 定義** — `core/policy/base.py` に抽象クラスを定義。risk budget 算出 / allocation 算出 / rebalance sell size 算出 / exit sell size 算出 の 4 責務 | #24 | III-D Policy と RM の責務境界 |
| 4 | **bot.py から policy 切り出し** — 現行の rotation + risk budget + full exit を `core/policy/rotation_riskbudget.py` へ抽出 | #24 | III-D |

**制約:**
- Risk Manager (`core/risk_manager.py`) は Policy に吸収しない。安全停止の別責務として維持（III-D）
- overlay は移行期の互換用途として残す。param_set 読み込みが安定したら廃止（III-B）
- 各ステップで既存の dry-run / 2-slot 比較挙動を壊さないことを確認してから次へ進む

---

## 詳細の置き場

- 観測結果の詳細: `observation-log.md`
- 全体地図と将来計画: `roadmap.md`
- 比較運用の原則: `multi-run-policy.md`
- 実務タスク・判断履歴: GitHub Issues

---

## 参照する Issue

- Epic: https://github.com/t-wp/crypt-bot/issues/1
- 1hour dry_run 観測継続: https://github.com/t-wp/crypt-bot/issues/3
- ローソク足データ欠損 WARN 調査: https://github.com/t-wp/crypt-bot/issues/20
- min_lot スキップ再分類: https://github.com/t-wp/crypt-bot/issues/21 （評価完了）
- all-negative 比率評価: https://github.com/t-wp/crypt-bot/issues/22
- Dashboard UI 刷新: https://github.com/t-wp/crypt-bot/issues/11

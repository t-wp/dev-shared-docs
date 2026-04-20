# Dry-run 観測ログ

---

## 観測の目的と判断軸

**目的:** 設計意図どおりに動いているかの確認。損益の最大化はまだ問わない。

**判断軸:**

| 確認項目 | 設計意図 |
|---|---|
| SELL の発火 | exit sell / rebalance sell が分かれて動いているか |
| BUY の配分 | 強い銘柄へ厚く配分されているか |
| JPY 残高 | 待機資金として自然に増減しているか（主役になっていないか） |
| state.json | peak_jpy・virtual_jpy に異常値がないか |
| スキップ理由 | 「配分枠充足」と「JPY 残高不足」が正しく区別されているか |

---

## 記録フォーマット

```
### YYYY-MM-DD HH:MM（JST）

- 見たポイント:
- 設計意図どおりだったか: Yes / No / 一部No
- 気になった点:
- 次に見ること:
```

生ログは貼らない。結論・解釈・次の観点だけを残す。

---

## 観測エントリ

### 2026-04-16（dry_run 開始期）

- 見たポイント: market_prices 事前取得バグ修正後の peak_jpy 挙動
- 設計意図どおりだったか: **修正後は正常**（修正前は BTC 価格で全銘柄評価 → peak_jpy 5.4億に膨張）
- 気になった点: 根因は market_prices 未渡しによる参照汚染。state.json の peak_jpy を ¥100,000 に手動リセットで対処済み
- 次に見ること: 複数サイクル後も peak_jpy が正常範囲（portfolio 総額付近）に収まっているか

### 2026-04-16（risk_budget・配分観測）

- 見たポイント: risk_budget の挙動と配分の自然さ
- 設計意図どおりだったか: Yes
- 気になった点:
    - risk_budget は固定72%ではなく、60〜70%台で自然に推移している
    - market_regime が thinking.json / JSON に保存されている
    - alloc 0% の銘柄は target_qty<=0 で BUY が止まる
    - ADA=72% 単独配分のような不自然な挙動は直近観測では再発していない
- 次に見ること:
    - all-negative（全銘柄下降）時に JPY 全待機へ切り替わるか
    - 各SELL種別（exit/rebalance）が observation-log で明確に記録できるか

### 2026-04-16（18サイクル連続観測 — risk_budget 動態・ローテーション動作確認）

- 見たポイント:
    - risk_budget が市場の弱化に応じてどう変化するか（18サイクル、14:10〜17:20）
    - exit sell / rebalance sell の役割分担が維持されているか
    - JPY が待機資金として自然に増えるか
    - alloc=0% 銘柄での BUY 誤発火がないか
- 設計意図どおりだったか: **Yes**
- 気になった点:
    - risk_budget は固定値ではなく、相場の弱化に応じて 91.9% → 36〜41% まで滑らかに低下した（暴れなし）
      - 起動直後: 91.9%（多数銘柄 positive、強い相場）
      - 中盤の安定帯: 74〜78%（全銘柄参加でなだらか）
      - 弱化後: 36〜41%（BTC/ETH/DOGE が順次脱落）
    - ローテーション動作は教科書的に観測できた。DOGE が 42.8% → 脱落、BTC が段階的に 0% へ縮小し、ADA が最強ポジションを維持
    - Branch A（exit sell）: SOL でデッドクロス → `SELL exit(full)` が正常発火。Branch B（rebalance sell）と混線なし
    - Branch B（rebalance sell）: alloc=0% になった銘柄に対して完全撤退方向の rebalance sell が順次動作
    - alloc=0% 銘柄への BUY 誤発火: なし（target_qty<=0 ガード正常）
    - JPY 残高は「強い行き先がない時に増える」挙動を示した（待機資金として機能）
    - ADA=72% 単独配分の異常: 直近では再発なし
    - alloc_sum=0 / min_alloc 残存などの異常: なし
    - DOGE が複数サイクル連続で rebalance sell → 現時点では過剰売買とは断定せず、段階的縮小として観測継続
    - top1_ref=0.010 / total_ref=0.030 は現市場レベル（top1 ≈ 0.7%、total ≈ 1.3%）に対して余裕があり、100% には届かない設定。明らかに不自然とは言えず、当面は観測継続
- 次に見ること:
    - **all-negative（全銘柄下降）時に JPY 全待機へ切り替わるか**（未観測。Step 5 完了条件の最終項目）
    - smoothing 未実装でも問題ない相場かどうか、より強い相場変動での risk_budget 挙動

### 2026-04-16〜17（exit sell / rebalance sell / all-negative の初観測）

- 見たポイント:
    - ADA のデッドクロスによる exit sell(full) の発火（18:00）
    - ADA の rebalance sell 連続動作（17:40 / 17:50）
    - all-negative（全銘柄下降）時の JPY 全待機への切り替わり
- 設計意図どおりだったか: **Yes**
- 気になった点:
    - exit sell(full): ADA がデッドクロス → 281 ADA を全量売却。`SELL exit(full)` が正常発火。rebalance sell と混線なし
    - rebalance sell: 17:40 に 483→447 ADA（超過率+8.1%）、17:50 に 447→281 ADA（超過率+59.1%）と段階的縮小。設計どおり
    - all-negative: 2026-04-16 20:30〜21:10（5サイクル）と 23:40〜00:20（5サイクル）の 2 回、計 10 サイクル以上発生
        - `全銘柄下降または市場低温 → JPY待機。...総投入率0.0%` が出力され、全銘柄が「判断: 見送り」に
        - BUY / SELL ともに一切実行されず、JPY に 100% 退避して静止。設計どおり
    - all-negative 明けに相場が反転上昇 → risk_budget が 0% から 94.7% まで自然回復
- 次に見ること:
    - Step 5 全完了条件が揃ったため、Step 6（dry_run 安定化）への移行判断

---

### 2026-04-17（Step 6 完了確認 — P1 異常チェック・JPY 挙動確認）

- 見たポイント:
    - 直近ログ（bot_20260417.log）での P1 異常不在の確認
    - thinking.json・state.json による JPY 待機と再配分の設計適合確認
- 設計意図どおりだったか: **Yes**
- 気になった点:
    - **P1 異常チェック（04-17 04:30 時点）**
        - peak_jpy: ¥100,697（初期 ¥100K 比 +0.7%、正常範囲）
        - virtual_jpy: ¥8.46（正値・全額投資中）
        - alloc=0% 銘柄への BUY 誤発火: なし（全銘柄 alloc_pct > 0）
        - ゼロ保有 SELL: なし（全銘柄 position_qty > 0）
        - 04-17 ログ ERROR/Traceback: **0 件**
        - ※ 04-16 03:06 の _run_once_symbol エラーは既知 deploy バグで修正済み・以降再発なし
    - **JPY 待機と再配分（04-17 04:30 時点）**
        - market_regime: breadth=6（全銘柄ポジティブ）、risk_budget=1.0
        - balance_jpy=¥8.46 → 行き先（全銘柄ポジティブ）があるため JPY を使い切っている
        - BUY シグナルあり・order_executed=false → JPY 残高不足のみ（設計どおり）
        - 「強い行き先がある → JPY を最大限使う」動作を確認
- 次に見ること: Step 7（本番移行判断）の検討

---

### 2026-04-17 17:00（1hour 移行初日 — ベースライン記録）

- 見たポイント: 1hour 切替後の初回サイクルと、比較用 10min ベースラインの記録
- 設計意図どおりだったか: **Yes**（副作用なし）
- 1hour bot 起動: 16:59:57、初回サイクル完了 17:00:13、次回 18:00:10
- 初回サイクル実績: BUY 3件（XRP, SOL, ADA）、SELL 3件（BTC rebalance, ETH exit, DOGE rebalance）
- 10min ベースライン（同日 03:39〜16:50、80サイクル）:

| 指標 | 10min 実測 |
|---|---|
| サイクル数 | 80（13.2h） |
| BUY | 99件（7.5/h） |
| SELL | 79件（6.0/h）— exit 11, rebalance 68 |
| TOTAL/h | 13.5 |
| risk_ng スキップ | 0 |
| 配分枠充足スキップ | 19 |
| 最小ロット未満スキップ | 58 |
| 週推算取引数 | ~2,268 |

- ポートフォリオ: ¥98,365（初期 ¥100K 比 -1.6%）、JPY ¥12,201（88% 投資中）
- 1hour 期待値（構造的推算）: サイクル数が 1/6 → BUY/h ~1.3, SELL/h ~1.0, TOTAL/h ~2.3
- 次に見ること（Issue #3 観点）:
    1. turnover が replay 想定に近いか（24h+ のデータ蓄積後）
    2. BUY 件数が 10min 比で構造的に落ちているか
    3. risk_ng vs min_lot_skip のどちらが主因か
    4. 1hour 化での副作用（シグナル遅延・配分偏り等）

> **判断**: 1hour 初回サイクルは正常。有意な比較分析には最低 24h（~24 サイクル）必要。次回観測は 4/18 以降。

---

### 2026-04-19（1hour 定常運用 2 日分の初期整理）

- 見たポイント:
  - 4/18〜4/19 の 1hour dry_run 観測集計
  - ローソク足データ欠損 WARN の再発有無
  - min_lot スキップと all-negative 比率
  - 2-slot 比較基盤が定常観測の土台として成立しているか
- 設計意図どおりだったか: **一部 Yes**
- 気になった点:
  - 4/18 は 25h で 57 cycles、BUY 8、SELL exit 0、SELL rebalance 7、欠損 WARN 0、all-negative 10
  - 4/19 は 10h で 55 cycles、BUY 38、SELL exit 10、SELL rebalance 31、欠損 WARN 0、all-negative 21
  - ポートフォリオは ¥98,821、累計 87 trades、peak は ¥100,192
  - ローソク足データ欠損 WARN は 4/17〜4/19 の定常運用で 0 件。interval 切替時の過渡現象だった可能性が高い
  - min_lot スキップが多く、BUY 試行の 6 割超を占める
  - all-negative は全サイクルの約 28% で、市場低温時に待機比率が高まる構造がはっきり出ている
  - 2-slot 比較基盤は起動確認済み。manifest 差分、/api/slots、slot-2 cold-start、shared DB 作成後の read-only 復帰まで確認できた
- 次に見ること:
  - min_lot スキップの多さが本番移行上の問題か、単なる 1hour 粒度の自然な結果かを切り分ける
  - all-negative 比率の高さが過度保守なのか、相場条件に対して妥当なのかを切り分ける
  - replay の見立てと実観測のズレを、件数だけでなく資金回転と配分挙動でも確認する

### 2026-04-19（#21 min_lot スキップ再分類 — 資金スケール仮説の棄却）

- 見たポイント:
  - min_lot スキップ 31 件（4/18: 14, 4/19: 17）の詳細内訳
  - 初期仮説「dry-run ¥100K で BTC/ETH の初回 BUY が届かない」の検証
  - `target_qty <= 0` 分岐のログ表示の正確性
- 設計意図どおりだったか: **一部 No（ログの可観測性バグ）**
- 評価結果:
  - min_lot 31 件を 3 カテゴリに再分類:
    - **rebalance 微小超過 < lot_step**: 19 件（61%）— lot_step 離散性の構造的制約。資金額に非依存
    - **BUY target=0 (alloc_pct=0%)**: 8 件（26%）— all-negative で配分率ゼロ。#22 と同一現象
    - **BUY 配分枠充足**: 4 件（13%）— 保有が目標近傍。正常動作
  - 6 銘柄すべてで ¥12K 配分予算は最小購入額の 30〜300 倍あり、初回 BUY が届かないケースは不成立
  - **ログバグ**: `target_qty <= 0` で `alloc_pct=0` の場合も「残高不足」と表示 → 「配分率ゼロ」に修正（90d6a99）
  - **初期仮説（資金スケール問題）は棄却**。min_lot スキップは本番でも同様に発生する構造的現象
- 次に見ること:
  - 修正後のログで 3 カテゴリが正しく分離表示されるか確認
  - #22 all-negative 比率 28% の評価（2-slot 差分を含む）

---

## フェーズ判定サマリ

| 日付 | 判定 | 根拠 |
|---|---|---|
| 2026-04-17 | **Step 5 完了条件 全達成** | exit sell・rebalance sell・risk_budget 動的変化・all-negative → JPY 全待機、すべて observation-log で確認済み |
| 2026-04-17 | **Step 6 完了条件 全達成** | P1 異常なし・JPY 待機/再配分が設計意図どおりであることを確認・observation-log 記録完了 |

> Step 7（本番移行判断）へ進む判断材料が揃った。

---

### 2026-04-20（LogLayout 移行 — bot.py records/ 正本化 + フラットファイル完全廃止）

- 見たポイント:
  - Phase 1: bot.py を LogLayout 経由に改修し、records/ を正本として書き込むようにする
  - Phase 2: デプロイ後に records/ への書き込み・フラット側の更新停止・停止ボタン連携を検証
  - Phase 3: prod / slot-1 / slot-2 の旧フラットファイル（state.json, thinking.json, events.json, bot_*.log, portfolio_history.json）を全削除
  - Phase 4: log_layout.py からフォールバックコードを全削除（`_resolve_read()`, `migrate_from_flat()`, 旧flat結合ロジック）、dashboard.py から `migrate_from_flat()` 呼び出し削除
- 設計意図どおりだったか: **Yes**
- 検証結果:
  - **bot.py 変更点（8箇所）**: LogLayout import 追加、パス初期化を LogLayout 経由に変更、thinking.json パス（3箇所）、events→summaries パス、portfolio_history を JSONL append 方式に変更
  - **Phase 2 検証（全7チェック PASS）**:
    - records/state.json: 18:50:11 にサイクルで更新 ✅
    - records/thinking.json: 18:50:11 に更新 ✅
    - summaries/summaries.json: 18:50:11 に更新 ✅
    - cache/portfolio_history JSONL: 6行→8行に増加 ✅
    - debug/bot_*.log: 起動ログ + サイクルログ出力 ✅
    - フラットファイル更新停止: 18:20 で凍結 ✅
    - 停止ボタン連携: records/state.json の running フラグを bot が正しく読み取り「[停止中]」スキップを確認 ✅
  - **slot-1/slot-2 も同様に全チェック PASS**（records/ に書き込み、フラット側は凍結）
  - **Phase 3**: prod/slot-1/slot-2 のフラットファイル計15ファイルを削除。削除後も bot/dashboard 正常動作
  - **Phase 4**: log_layout.py から `_resolve_read()`, `migrate_from_flat()`, 旧flat結合ロジックを全削除。dashboard.py から `migrate_from_flat()` 呼び出し削除。構文検証 OK、デプロイ後正常動作
- 最終状態:
  - records/ が唯一の正本。フラットファイルは存在しない
  - フォールバックコードなし。読み込みメソッドは直接新配置パスを返す
  - 全5コンテナ正常稼働（prod + slot-1 + slot-2 + dashboard + tailscale）
- 次に見ること:
  - 次回サイクル以降のエラーなし稼働の継続確認
  - Phase 5-1 の実装着手（param_set 正本固定 → Policy interface 定義）


### 2026-04-20（設計決定: interval 方針 + AI half-day summary 設計）

- 見たポイント:
  - 「1hour 固定」前提の docs 記述を実態に合わせて見直す
  - AI half-day summary の責務分離と設計方針の確定
- 設計意図どおりだったか: **設計確定（実装前）**
- 決定内容:
  - **interval の扱い**: bar-close 型は可変（10min / 30min / 1hour）。1hour は暫定 champion だが固定ではない。同一 experiment 内では interval を混在させない。interval 比較は experiment 間で行う
  - **docs 更新**: current-phase.md / roadmap.md から「1hour 固定」前提の表現を除去。「bar-close dry_run 観測フェーズ」に統一
  - **新規 docs 追加**:
    - un-log-architecture.md: LogLayout 完了後の保存構造と責務分離の正本
    - observation-method.md: ログ確認手順・コマンド早見表・interval 変更時の注意
    - i-summary-pipeline.md: AI half-day summary の設計決定版
  - **AI half-day summary 設計**:
    - cycle_payload（records/thinking.json）が正本。読み取り専用
    - window は 08:30 / 20:30 の固定時刻で切る（interval に依存しない）
    - window_summary は派生物。ai_summaries/ に保存
    - AI 失敗時は fallback（数値テキスト化）。bot 本体を止めない
- 次に見ること:
  - Issue #26（AI half-day summary 実装）の着手判断
  - #22（all-negative 評価）の継続
  - Phase 5-1 の実装着手

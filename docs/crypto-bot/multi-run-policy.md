# 並行Dry-Run比較運用 方針書 v1.0

> **策定日:** 2026-04-17
> **ステータス:** 承認済み・Phase 0 着手可

---

## 原則

- **Param Set は不変のスナップショットとして扱い、既存セットの上書き更新は行わない。**
- **Dry-Run 比較は同一市場データ・同一初期条件・同一コスト前提のもとで行う。**
- **experiment 期間中は model / policy / param_set / コスト前提を凍結する。途中変更した場合は別 experiment_id に切る。**

---

## I. 管理単位の3階層

| 階層 | 役割 | 具体例（現行） |
|---|---|---|
| **Signal Model** | 銘柄ごとのBUY/SELL/HOLDシグナル生成 | `ma_rsi`（MA cross + RSI filter） |
| **Portfolio Policy** | 配分・exit・rebalance・risk budgetの方式 | rotation + momentum alloc + risk budget + full exit + excess trimming |
| **Param Set** | 上記2つに渡す具体的な数値の組 | short=5, long=25, rsi=14, OB=70, OS=30, breadth重み, deadband, ... |

**「モデル」= Signal Model + Portfolio Policy** の組み合わせ全体。
Param Setはモデルに対する設定値の**不変スナップショット**。

---

## II. 探索対象パラメータ

### Signal Model側

| パラメータ | 範囲案 | 現在値 |
|---|---|---|
| short_window | 3〜20 | 5 |
| long_window | 15〜60 | 25 |
| rsi_period | 7〜21 | 14 |
| overbought | 65〜80 | 70 |
| oversold | 20〜35 | 30 |
| interval | 10min / 30min / 1h | 10min |

### Portfolio Policy側（本丸）

| パラメータ | 説明 |
|---|---|
| breadth_weight | risk budgetへのbreadth寄与 |
| top1_weight | risk budgetへのtop1寄与 |
| total_weight | risk budgetへのtotal寄与 |
| exit_mode | full / half_repeat / half_once |
| rebalance_deadband | 超過何%でtrimするか |
| min_alloc_pct | 配分下限（丸め閾値） |
| JPY待機比率 | 最低/最大現金保持 |

> **注:** 各パラメータの現行値と探索範囲の具体値は Phase 0 成果物（現行値確定表）で埋める。

---

## III. 識別子体系

```
model_id        : "ma-rsi-rotation-v1"
model_version   : "1.0.0"
signal_model    : "ma_rsi"
policy_id       : "rotation-riskbudget-fullexit"
param_set_id    : "ps-001"          ← 不変。修正時は ps-002 を新規発行
experiment_id   : "exp-20260417-w01" ← 週次ラウンド単位
run_id          : "dryrun-slot-1-20260417"
mode            : "dryrun" | "prod"
```

### 追跡ルール

- `param_set_id` は一度作成したら中身固定。変更は新IDで発行
- `experiment_id` で「この週の5台」をグルーピング
- どの `run_id` がどの `param_set_id` / `experiment_id` を使ったか完全追跡可能にする
- **param 変更のみの週** と **model_version が変わった週** はダッシュボード・ログ上で別扱いにする

---

## IV. Dry-Run比較の前提条件（統一必須）

| 項目 | 統一内容 |
|---|---|
| 市場データ | Market Data Feeder経由の同一OHLC |
| 開始時刻 | 全slot同時スタート |
| 初期JPY | 同額（例: ¥100,000） |
| 初期保有 | 全slot同一（通常はゼロスタート） |
| 手数料前提 | taker_fee 統一 |
| スリッページ前提 | 同一モデル（当面0、将来的にシミュレーション追加可） |
| 最小発注量・丸め | 同一symbols定義 |

---

## V. experiment 凍結ルール

- `experiment_id` 開始後は、その週の **model / policy / param_set / コスト前提** を凍結
- 途中でいずれかを変更した場合は **別 experiment_id** に切り替え、前experimentは中断扱いで記録
- コードの変更（model_version更新）を伴う週と、param_set のみの変更週は明確に区別する

---

## VI. アーキテクチャ

```
┌──────────────────────────────────────────────────┐
│  Market Data Feeder（1プロセス）                    │
│  OHLC / ticker を定期取得 → 共有ストア（SQLite等）  │
└──────────┬───────────────────────────────────────┘
           │ 同一データを参照
    ┌──────┼──────┬──────┬──────┬──────┐
    ▼      ▼      ▼      ▼      ▼      ▼
  Slot1  Slot2  Slot3  Slot4  Slot5  Prod
  ps-001 ps-002 ps-003 ps-004 ps-005 ps-XXX
  (dry)  (dry)  (dry)  (dry)  (dry)  (prod)
```

### Feeder障害対策

- **Dry-Run**: Feeder必須（Feeder停止中はslot全停止 → 比較公平性を維持）
- **Prod**: Feeder優先、Feeder障害時は直接API取得にフォールバック

---

## VII. 探索スロット配分ルール

| スロット | 役割 | 説明 |
|---|---|---|
| Slot 1 | **チャンピオン維持** | 前週1位のParam Setをそのまま継続 |
| Slot 2 | 近傍探索A | チャンピオンから1〜2軸を微調整 |
| Slot 3 | 近傍探索B | チャンピオンから別の1〜2軸を微調整 |
| Slot 4 | **遠方探索A** | パラメータ空間の離れた領域 |
| Slot 5 | **遠方探索B** | 同上（別の領域） |

→ 局所最適回避のため、常に40%を離れた領域に割く

---

## VIII. 評価体系（二段構え）

### A. ランキング用複合スコア

| 指標 | 重み案 |
|---|---|
| 総損益（手数料込み） | ★★★ |
| Sharpe比 | ★★ |
| 最大DD（小さいほど良い） | ★★★ |
| 連敗耐性 | ★ |
| Turnover（低いほど良い） | ★ |

### B. 昇格ゲート条件（一つでも未達なら昇格不可）

| 条件 | 閾値案 |
|---|---|
| 最大DD | ≤ 15% |
| 最大連敗数 | ≤ 7 |
| 週間売買回数 | ≤ 上限値（Phase 0 で確定） |
| 観測期間 | ≥ 2週間 |
| 手数料比率 | ≤ 利益の30% |

**「総合スコアで順位をつけるが、ゲート条件を満たさないものは本番候補にしない」**

---

## IX. 必須観測項目（全run共通でログに残す）

| カテゴリ | 項目 |
|---|---|
| Signal | 判定根拠（MA値、RSI値、クロス方向、判定理由文） |
| Alloc | 算出結果（銘柄別target_alloc、momentum score） |
| Risk Budget | 算出内訳（breadth/top1/total各スコア、最終budget%） |
| Exit/Rebalance | 発火理由（exit条件、超過量、trim量） |
| Execution | target_qty vs actual_qty の差分、丸め影響 |
| Portfolio | cycleごとのsnapshot（JPY残高、銘柄別評価額、合計） |

→ 後から「なぜこのrunが勝った/負けたか」を分析可能にする

---

## X. 本番運用ルール

| 項目 | ルール |
|---|---|
| 昇格 | 手動判断。ゲート条件クリア + 2週間以上の安定上位 |
| 昇格後監視 | 昇格直後は監視強化期間（1週間）。想定外のturnover/DD/API異常で即ロールバック |
| ロールバック | DD閾値超過 or 連敗閾値到達で前のParam Setに即時復帰 |
| 記録 | 昇格/ロールバックの日時・理由・前後のparam_set_idを記録 |

---

## XI. 探索サイクルと昇格判断の分離

| | 周期 | 目的 |
|---|---|---|
| **探索更新** | 1週間 | 5台のうち下位を入れ替え、新Param Setを投入 |
| **本番昇格判断** | 2〜4週間以上 | 安定して上位にいる組み合わせのみ昇格候補 |
| **本番ロールバック** | 即時（条件ベース） | DD閾値超過 or 連敗閾値で前の設定に戻す |

---

## XII. 実装ロードマップ

| Phase | 内容 | 備考 |
|---|---|---|
| **0. 設計** | 識別子体系確定、評価指標定義、探索パラメータ軸確定、必須観測項目の固定、昇格ゲート条件の明文化、**現行値確定表の作成** | ドキュメント作業。抽象値を全て具体値に落とす |
| **1. 分解** | 現行bot.pyからSignal Model / Portfolio Policy / Param Setを構造的に分離。差し替え可能な構造へのリファクタ | **重い。** 判定責務切り出し、I/O分離、state更新整理、ログ責務分離を含む |
| **2. データ共有** | Market Data Feederの分離。共有ストア設計。Prodフォールバック実装 | |
| **3. 並行Dry-Run** | docker-compose で slot×5 + prod。独立ログ/state。experiment_id/run_id管理 | |
| **4. 比較ダッシュボード** | 全slotの成績並列表示。複合スコアランキング。ゲート条件合否表示 | |
| **5. 運用開始** | 探索サイクル開始。週次入れ替え。手動昇格判断 | |
| **6. モデル多様化** | trend/range/defensive等、Signal ModelやPolicy自体を複数用意 | 十分な運用知見を得てから |

---

## Phase 0 成果物チェックリスト

Phase 0 完了時に以下が全て埋まっていること:

- [ ] breadth_weight 現行値と探索範囲
- [ ] top1_weight 現行値と探索範囲
- [ ] total_weight 現行値と探索範囲
- [ ] rebalance_deadband 現行値と探索範囲
- [ ] weekly trade cap 閾値
- [ ] turnover 計算式
- [ ] Sharpe 計算粒度（日次 or cycle単位）
- [ ] 複合スコアの重み付け数値（★を具体的係数に）
- [ ] 昇格ゲート条件の全閾値確定

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

### モデルの管理方法

モデルは **コード置き場** と **識別子** の2軸で管理する。

| 軸 | 管理対象 | 保存先 |
|---|---|---|
| コード置き場 | Signal Model のロジック | `core/strategy/*.py` |
| コード置き場 | Portfolio Policy のロジック | `bot.py`（今後 Phase 1 で policy module として分離予定） |
| 識別子 | 組み合わせの特定 | `model_id` + `model_version` + `policy_id` |

運用上の「モデル」はコードのファイルパスではなく、**識別子（`model_id` / `policy_id` / `model_version`）で管理する**。
コードが同じでも `model_version` が異なれば別モデルとして扱う。

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

| 指標 | 重み | 備考 |
|---|---|---|
| 総損益（手数料込み） | 0.30 | |
| Sharpe比（日次） | 0.25 | 日次リターンから算出。年率換算は ×√365 |
| 最大DD（小さいほど良い） | 0.25 | 低DDを高く評価 |
| 連敗耐性 | 0.10 | 最大連敗数の逆数的評価 |
| Turnover（低いほど良い） | 0.10 | 週次片側turnover。低回転を高く評価 |

> スコア = Σ（正規化指標 × 重み）。正規化は slot 間 min-max。初期重みは上記で運用開始し、2〜3週間後に感度を見て調整する。

### B. 昇格ゲート条件（一つでも未達なら昇格不可）

| 条件 | 閾値 | 根拠 |
|---|---|---|
| 最大DD | ≤ 15% | |
| 最大連敗数 | ≤ 7 | |
| 週次売買回数 | ≤ 500 回/週 | 現行実測 ~350件/日=~2,500/週（10min足, rebalance過多含む）。改善後の上限として設定 |
| 週次片側turnover | ≤ 500% | 現行実測 ~6,600%/週は異常。deadband拡大後の目標上限 |
| 手数料率 | ≤ 総損益の 20% | 手数料負けを昇格前に排除 |
| 観測期間 | ≥ 2週間 | |
| 手数料比率 | ≤ 利益の30% |

**「総合スコアで順位をつけるが、ゲート条件を満たさないものは本番候補にしない」**

---

## IX. 保存構造

### ディレクトリ構成

```
crypto-bot/
├── config.yaml                       ← 共通設定（symbols, 手数料等）
├── param_sets/                       ← 不変 Param Set の保管庫
│   ├── ps-001.yaml
│   ├── ps-002.yaml
│   └── ...
├── logs/
│   ├── shared/                       ← Market Data Feeder の共有データ
│   │   └── market_data.db
│   ├── experiments/                  ← experiment 単位で分離
│   │   └── exp-20260417-w01/
│   │       ├── manifest.yaml             ← 固定条件（slot↔param_set紐付け、初期条件、model_version）
│   │       ├── scoreboard.json           ← 集計結果（複合スコア・ランキング・ゲート判定）
│   │       ├── promotion_decision.json   ← 昇格判断・却下理由の記録
│   │       ├── notes.md                  ← 人間の所感・備考
│   │       ├── slot-1/                   ← run 単位で分離
│   │       │   ├── state.json
│   │       │   ├── portfolio_history.json
│   │       │   ├── thinking.json
│   │       │   └── bot_YYYYMMDD.log
│   │       ├── slot-2/ ... slot-5/
│   │       └── (次週: exp-20260424-w02/ を新規作成)
│   └── prod/                         ← 本番は experiment とは分離
│       ├── state.json
│       ├── portfolio_history.json
│       ├── thinking.json
│       └── bot_YYYYMMDD.log
└── core/
    └── strategy/
```

### 保存ポリシー

| 何を | どこに | 不変性 | 備考 |
|---|---|---|---|
| **Param Set** | `param_sets/ps-NNN.yaml` | **不変** | 一度作ったら書き換えない。修正は新ID発行 |
| **Experiment定義** | `logs/experiments/exp-XXXXX/manifest.yaml` | **凍結** | experiment開始後は変更不可 |
| **集計結果** | `logs/experiments/exp-XXXXX/scoreboard.json` | 追記 | 週次評価時に書き込み |
| **昇格判断** | `logs/experiments/exp-XXXXX/promotion_decision.json` | 追記 | 昇格/却下の判断とその理由 |
| **run別ログ** | `logs/experiments/exp-XXXXX/slot-N/` | 追記のみ | experiment単位で完全分離 |
| **本番ログ** | `logs/prod/` | 追記のみ | experimentとは独立 |
| **市場データ** | `logs/shared/market_data.db` | 共有 | Feeder が書き込み、全botが読み取り |

### 全成果物へのメタ情報埋め込み

`state.json` / `portfolio_history.json` / `thinking.json` / `bot_*.log` の全てに、以下の識別子を含める。
ファイル単体で拾った時に正体不明にならないため。

```json
{
  "run_id": "dryrun-slot-1-20260417",
  "experiment_id": "exp-20260417-w01",
  "model_id": "ma-rsi-rotation-v1",
  "model_version": "1.0.0",
  "policy_id": "rotation-riskbudget-fullexit",
  "param_set_id": "ps-001",
  "mode": "dryrun",
  ...
}
```

`bot_*.log` は各行の先頭に `run_id` を付与し、grep で特定 run のログだけ抽出可能にする。

---

## X. 必須観測項目（全run共通でログに残す）

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

## XI. 本番運用ルール

| 項目 | ルール |
|---|---|
| 昇格 | 手動判断。ゲート条件クリア + 2週間以上の安定上位 |
| 昇格後監視 | 昇格直後は監視強化期間（1週間）。想定外のturnover/DD/API異常で即ロールバック |
| ロールバック | DD閾値超過 or 連敗閾値到達で前のParam Setに即時復帰 |
| 記録 | 昇格/ロールバックの日時・理由・前後のparam_set_idを記録 |

---

## XII. 探索サイクルと昇格判断の分離

| | 周期 | 目的 |
|---|---|---|
| **探索更新** | 1週間 | 5台のうち下位を入れ替え、新Param Setを投入 |
| **本番昇格判断** | 2〜4週間以上 | 安定して上位にいる組み合わせのみ昇格候補 |
| **本番ロールバック** | 即時（条件ベース） | DD閾値超過 or 連敗閾値で前の設定に戻す |

---

## XIII. 実装ロードマップ

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

## 現行運用の実測データ（2026-04-17 取得）

> Phase 0 の仮値設定根拠として記録。観測期間: 2026-04-15 17:50 ~ 2026-04-17 11:00（フル24hは4/16のみ）
> 条件: ma_rsi v2.0.0, 10min足, 6銘柄, exit_mode=full, position_pct=0.12, rebalance_threshold=0.05, 初期資産 ¥100,000

### 売買回数

| 粒度 | 値 |
|---|---|
| 総イベント数 | 567件（1.7日間） |
| 4/16 フル24h | 353件 |
| サイクルあたり平均 | 3.1件（中央値3, 最大10） |
| 週換算推定 | ~2,500件/週 |

### reason別 内訳（4/16 フル24h）

| reason | 件数 | 構成比 | 代金 |
|---|---|---|---|
| buy | 201 | 57% | ¥1,255,129 |
| rebalance_sell | 121 | 34% | ¥341,949 |
| exit_sell | 25 | 7% | ¥248,042 |
| sell_unknown（初期） | 6 | 2% | ¥17,268 |

### rebalance sell の特徴

| 項目 | 値 |
|---|---|
| 発火サイクル率 | 62%（全182サイクル中112で発火） |
| 平均代金 | ¥2,958 |
| 中央値代金 | ¥1,927 |
| 最小代金 | ¥237 |

→ **deadband=5%が10min足×6銘柄では頻繁すぎる。小額rebalanceが過剰回転の主因。**

### turnover

| 指標 | 値 |
|---|---|
| 日次turnover（4/16） | 1,903%（¥1.86M / ¥97.9K） |
| 日次片側turnover | 951% |
| 週次片側turnover（推定） | ~6,660% |

→ **ローテーションbotとしても異常に高い。deadband拡大が最優先課題。**

### 手数料

| 項目 | 値 |
|---|---|
| 総手数料（1.7日） | ¥1,944 |
| 日次推定 | ¥650/日 |
| 週次推定 | ¥4,550/週 |
| 総資産比 | ~0.65%/日 = ~4.6%/週 |

---

## Phase 0 成果物チェックリスト

Phase 0 完了時に以下が全て埋まっていること:

- [ ] breadth_weight 現行値と探索範囲 — 現行: 0.40（仮値）、探索範囲: 0.20〜0.60
- [ ] top1_weight 現行値と探索範囲 — 現行: 0.30（仮値）、探索範囲: 0.10〜0.50
- [ ] total_weight 現行値と探索範囲 — 現行: 0.30（仮値）、探索範囲: 0.10〜0.50
- [x] rebalance_deadband 現行値と探索範囲 — 現行: 0.05（5%超過で発火）→ **要拡大**。実測でサイクルの62%でrebalance発火、中央値代金¥1,927と小額連発。探索範囲: 0.05〜0.20
- [x] weekly trade cap 閾値 — 実測: ~2,500件/週（10min×6銘柄）。3段階設定:
  - 通常上限: 500件/週
  - 過熱警告: 750件/週
  - 即停止: 1,000件/週
  - ※ rebalance deadband 拡大後の現実的な値。現行のまま（deadband=5%）では全slot即停止レベル
- [x] turnover 計算式 — **週次片側turnover = 0.5 × (週次買い代金 + 週次売り代金) ÷ 週次平均総資産**
  - 実測参考: 現行片側 ~6,600%/週（日次 ~950%）、総資産 ~¥98K に対し日次代金 ~¥1.86M
- [x] Sharpe 計算粒度 — **日次 Sharpe**（主要指標）。cycle単位リターンは補助観測用に記録
  - 10min足はノイズ過多、週次はサンプル不足。日次が妥当
- [x] 複合スコアの重み付け数値 — 総損益0.30 / Sharpe0.25 / maxDD0.25 / 連敗0.10 / Turnover0.10（Section VIII-A に記載）
- [x] 昇格ゲート条件の全閾値確定 — maxDD≤15%, 連敗≤7, 週次売買≤500, 片側turnover≤500%, 手数料≤損益20%, 観測≥2週（Section VIII-B に記載）
- [ ] `candle_days` 最適化（3→1）: `get_klines` は毎サイクル3日分×6銘柄=18回取得しているが、SQLiteに `INSERT OR IGNORE` しているため大半が重複。初回以降は当日分だけで十分（18→6回/cycle）。Feeder導入前に単独で適用可能
- [ ] Dashboard API呼び出し削減: 現行はbot+dashboardで最大158回/10min、うちdashboardが89%（get_ticker×120 + get_assets×20）。キャッシュTTL延長やWebSocket化を検討

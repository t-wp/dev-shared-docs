# crypto-bot 売買ロジック仕様書

> **目的:** 他の AI・開発者との共同開発のための参照資料。
> ロジックを変更した場合はこのファイルも同時に更新すること。
>
> 最終更新: 2026-04-17
> 対応コード: bot.py v4.0.0 / ma_rsi v2.0.0

---

## 目次

1. [全体アーキテクチャ](#1-全体アーキテクチャ)
2. [1サイクルの処理フロー](#2-1サイクルの処理フロー)
3. [配分計算レイヤー（market regime）](#3-配分計算レイヤーmarket-regime)
4. [シグナル判定（ma_rsi v2.0）](#4-シグナル判定ma_rsi-v20)
5. [注文処理の分岐](#5-注文処理の分岐)
   - [A. exit sell](#a-exit-sell)
   - [B. rebalance sell](#b-rebalance-sell)
   - [C. BUY](#c-buy)
   - [D. HOLD](#d-hold)
6. [リスク管理](#6-リスク管理)
7. [注文量計算](#7-注文量計算)
8. [手数料計算](#8-手数料計算)
9. [state.json の更新ルール](#9-statejson-の更新ルール)
10. [設定パラメータ一覧](#10-設定パラメータ一覧)
11. [変更履歴](#11-変更履歴)

---

## 1. 全体アーキテクチャ

```
config.yaml
    │
    ▼
Bot（bot.py）
  ├─ MarketDataCollector（core/market_data.py）  ← ローソク足取得・DB蓄積
  ├─ Strategy（core/strategy/ma_rsi.py）         ← BUY/SELL/HOLD 判定
  ├─ RiskManager（core/risk_manager.py）         ← BUY 発注前の安全チェック
  └─ ExecutionEngine（core/execution.py）        ← 模擬/本番 注文実行
```

- **dry_run モード**: GMO API を呼ばず `state.json` の仮想残高を操作
- **live モード**: GMO コイン API 経由で実注文（成行）

---

## 2. 1サイクルの処理フロー

サイクル間隔: `interval`（現在 `10min`）。足確定後 `confirm_lag=10` 秒待機して実行。

```
run_once()
  │
  ├─ ① 日次損失リセット（日付変わり検出）
  │
  ├─ ② 全銘柄の MA 乖離スコアを取得
  │     → _calc_risk_budget()  → risk_budget（0〜1.0）
  │     → _calc_momentum_alloc() → alloc_map（銘柄ごとの配分率）
  │
  ├─ ③ 全銘柄の最新価格を事前取得（ポートフォリオ評価の整合性確保）
  │
  └─ ④ 各銘柄を順番に _run_once_symbol() で処理
         │
         ├─ ローソク足取得（DB から 200 本）
         ├─ シグナル計算（strategy.calc_signal）
         ├─ portfolio_value / target_qty 計算
         ├─ peak_jpy 更新（ハイウォーターマーク）
         │
         ├─ [A] sig == SELL  → exit sell（最優先）
         ├─ [B] 保有 > target → rebalance sell
         ├─ [C] sig == BUY   → リスクチェック → 差分購入
         └─ [D] sig == HOLD  → スキップ

  全銘柄処理後:
    → thinking.json 一括保存
    → state.json 保存
    → portfolio_history.json 追記
```

---

## 3. 配分計算レイヤー（market regime）

### 3-1. MA 乖離スコア（銘柄ごと）

```
score[銘柄] = (短期MA - 長期MA) / 長期MA
```

- スコア > 0 → 上昇トレンド中（配分対象）
- スコア ≤ 0 → 下降トレンド中（配分 0%）

### 3-2. risk_budget（総投入率上限）

```
breadth_component = (正スコア銘柄数 / 全銘柄数) × breadth_weight
top1_component    = min(最大スコア / top1_ref, 1.0) × top1_weight
total_component   = min(スコア合計 / total_ref, 1.0) × total_weight

raw_score  = clip(breadth + top1 + total, 0.0, 1.0)
risk_budget = min_budget + (max_budget - min_budget) × raw_score
```

| パラメータ | 現在値 | 備考 |
|---|---|---|
| `breadth_weight` | 0.40 | ⚠️ 仮値 |
| `top1_weight` | 0.30 | ⚠️ 仮値 |
| `total_weight` | 0.30 | ⚠️ 仮値 |
| `top1_ref` | 0.010 | ⚠️ 仮値（MA乖離率1%基準） |
| `total_ref` | 0.030 | ⚠️ 仮値（合計3%基準） |
| `min_budget` | 0.0 | 全銘柄下降時は 0%（JPY全待機） |
| `max_budget` | 1.0 | 全強相場で 100% まで許容 |

### 3-3. 各銘柄への配分率

```
alloc_pct[銘柄] = risk_budget × (銘柄スコア / 正スコア合計)
```

- 全銘柄スコア ≤ 0 または risk_budget ≤ 0 → 全銘柄 alloc=0%（JPY全待機）
- 浮動小数点誤差を防ぐため正規化（合計が risk_budget に一致するよう補正）

---

## 4. シグナル判定（ma_rsi v2.0）

**ソース:** `core/strategy/ma_rsi.py`

### 入力パラメータ

| パラメータ | 現在値 | 意味 |
|---|---|---|
| `short_period` | 5 | 短期MA期間 |
| `long_period` | 25 | 長期MA期間 |
| `rsi_period` | 14 | RSI計算期間（Wilder平滑化） |
| `rsi_overbought` | 70 | BUYブロック閾値（RSIがこれ以上→HOLD） |
| `rsi_oversold` | 30 | SELLブロック閾値（RSIがこれ以下→HOLD） |

**最低必要ローソク数:** `max(long_period + 1, rsi_period × 2)` 本

### シグナル判定ルール（優先度順）

| 優先度 | 条件 | 出力 |
|---|---|---|
| 1 | ゴールデンクロス AND RSI < overbought | **BUY** |
| 2 | ゴールデンクロス AND RSI ≥ overbought | HOLD |
| 3 | デッドクロス AND RSI > oversold | **SELL** |
| 4 | デッドクロス AND RSI ≤ oversold | HOLD |
| 5 | 上昇トレンド継続（short > long）AND RSI < overbought | **BUY** |
| 6 | 上昇トレンド継続 AND RSI ≥ overbought | HOLD |
| 7 | 下降トレンド継続（short < long）AND RSI > oversold | **SELL** |
| 8 | 下降トレンド継続 AND RSI ≤ oversold | HOLD |
| 9 | それ以外 | HOLD |

- **ゴールデンクロス:** 前足 `short ≤ long` かつ 現足 `short > long`
- **デッドクロス:** 前足 `short ≥ long` かつ 現足 `short < long`
- **クロスよりトレンド継続が頻繁に発生**するため、実際の注文の大半はルール 5/7 由来

---

## 5. 注文処理の分岐

### 分岐の優先順位

```
A. exit sell    （SELL シグナル）
B. rebalance sell（保有 > 目標、SIGNALに関わらず）
C. BUY          （BUY シグナル、かつリスクOK）
D. HOLD/スキップ
```

> **A は B より優先。** rebalance sell 発火サイクルは同銘柄の BUY をスキップする（分岐が排他）。

---

### A. exit sell

**発火条件:** `sig.action == SELL`

```
保有量 == 0 → スキップ（ゼロ保有 SELL は実行しない）
保有量 > 0  → exit_mode に応じた量を決定して売却
```

**exit_mode の動き:**

| exit_mode | 売却量 | 説明 |
|---|---|---|
| `full`（現在設定） | 保有全量 | シグナルが出たら即全量決済 |
| `half_repeat` | `floor(保有 × sell_ratio / lot_step) × lot_step` | 毎回 50% ずつ繰り返し売却 |
| `half_once` | 初回のみ 50%、2回目以降はスキップ | エントリーを残しながら一部利確 |

- `sell_ratio` = 0.50（`half_repeat` / `half_once` で使用）
- SELL 注文はリスクチェック免除（利確・損切りをブロックしない）

---

### B. rebalance sell

**発火条件:** `sig.action != SELL` かつ `保有量 > target_qty`

```
target_qty == 0 の場合（完全撤退）:
  → デッドバンド・最小金額チェックなし
  → 保有を全量売却（min_size / lot_step 以上であれば）

target_qty > 0 の場合（通常リバランス）:
  → 超過率チェック: 保有 ≤ target × (1 + rebalance_threshold) → スキップ
  → 丸め後の売却量 × 価格 < rebalance_min_jpy → スキップ
  → 超過分 = 保有 - target
  → 売却量 = floor(超過分 / lot_step) × lot_step
  → 売却量 < min_size → スキップ
```

| パラメータ | 現在値 | 意味 |
|---|---|---|
| `rebalance_threshold` | 0.05（5%） | デッドバンド。超過率がこれ以下なら売らない |
| `rebalance_min_jpy` | ¥500 | 最小売却金額。これ未満なら売らない |

---

### C. BUY

**発火条件:** `sig.action == BUY` かつリスクチェック通過

**注文量計算:**

```
target_qty  = portfolio_total × alloc_pct / 価格
gap         = target_qty - 現在保有
order_size  = floor(gap / lot_step) × lot_step

JPYキャップ:
  max_affordable = floor(JPY残高 × 0.99 / 価格 / lot_step) × lot_step
  order_size     = min(order_size, max_affordable)

order_size < min_size → スキップ
  └─ JPY < min_size × 価格 → 「JPY残高不足」
  └─ それ以外             → 「配分枠充足」（既に目標に近い）
```

- `portfolio_total` = JPY残高 + 全保有銘柄の時価評価額（全銘柄の最新価格で計算）
- BUY後に `half_once` フラグ（`half_sold`）をリセット

---

### D. HOLD

BUY でも SELL でもない場合、または B・C の各スキップ条件に該当する場合。
state.json は更新しない（thinking.json のみ記録）。

---

## 6. リスク管理

**ソース:** `core/risk_manager.py`

**適用対象:** BUY のみ。SELL は常に免除。

チェックは以下の順に実行し、最初の NG で即停止:

### チェック一覧

| # | チェック名 | 停止条件 | 停止スコープ |
|---|---|---|---|
| 1 | 日次最大損失 | `日次損失 / portfolio_value ≥ max_daily_loss_pct（5%）` | 当日の全銘柄 BUY |
| 2 | 最大連敗数 | `銘柄別連敗数 ≥ max_consecutive_losses（5回）` | 該当銘柄のみ |
| 3 | 最大ドローダウン | `(peak_jpy - portfolio_value) / peak_jpy ≥ max_drawdown_pct（20%）` | 全銘柄 BUY |
| 4 | 未約定注文（本番のみ） | `未約定注文数 ≥ max_open_orders（1件）` | 該当銘柄のみ |
| 5 | ボラティリティ急変 | `直近 10 本の高値・安値スプレッド / 平均 ≥ volatility_threshold（10%）` | 該当銘柄のみ |

- `peak_jpy`: ハイウォーターマーク（上方向のみ更新）
- `daily_loss_jpy`: 日付が変わるとリセット
- `consecutive_losses`: 勝ったらリセット、銘柄別に管理

---

## 7. 注文量計算

### BUY 注文量

```python
raw        = portfolio_total × alloc_pct / 価格
qty        = floor(raw / lot_step) × lot_step   # lot_step 切り捨て
qty        = round(qty, 8)                       # 浮動小数点誤差対策
if qty < min_size: return 0.0

# JPYキャップ
max_aff    = floor(JPY × 0.99 / 価格 / lot_step) × lot_step
order_size = min(qty, max_aff)
```

### SELL 注文量

| モード | 計算式 |
|---|---|
| `full` | `hold_size`（全量） |
| `half_repeat` | `floor(hold_size × 0.50 / lot_step) × lot_step`（0なら全量） |
| `half_once` | 初回: 同上。2回目以降: 0（スキップ） |

### rebalance sell 注文量

```python
excess = position_qty - target_qty
size   = floor(excess / lot_step) × lot_step
size   = round(size, 8)
if size < min_size: return 0.0
if target_qty > 0 and size × price < rebalance_min_jpy: return 0.0
```

---

## 8. 手数料計算

```
手数料 = 価格 × 数量 × taker_fee（0.05% = 0.0005）
```

**PnL（SELL時）:**
```
pnl = (売価 - 買価) × 数量 - (買い手数料 + 売り手数料)
```

- 手数料は BUY・SELL 両方で計上（二重計上ではなく両サイドの実コスト）
- 日次損失・total_pnl にはこの pnl が反映される

---

## 9. state.json の更新ルール

| フィールド | 更新タイミング | 更新内容 |
|---|---|---|
| `virtual_jpy` | BUY / SELL 実行後 | BUY: 減算（cost + fee）、SELL: 加算（proceeds - fee） |
| `positions[銘柄].virtual_qty` | BUY / SELL 実行後 | 保有量を加減 |
| `positions[銘柄].virtual_buy_price` | BUY 実行後 | 加重平均買値に更新。全量売却後はリセット（0.0） |
| `positions[銘柄].half_sold` | SELL / BUY 実行後 | `half_once` モード専用フラグ。BUY時にリセット |
| `peak_jpy` | 毎サイクル（全銘柄処理前） | portfolio_value > peak_jpy のときのみ上書き |
| `total_pnl` | SELL 実行後 | pnl を加算（累積） |
| `daily_loss_jpy` | SELL 損失時 / 日付変わり | 損失発生で加算、翌日 0 にリセット |
| `consecutive_losses` | SELL 実行後 | 損失: +1、利益: 0 にリセット（銘柄別） |
| `trade_count` | BUY / SELL 実行後 | +1 |
| `running` | ダッシュボード操作 | false のときスキップ |

---

## 10. 設定パラメータ一覧

### config.yaml 全パラメータ

**基本設定**

| パラメータ | 現在値 | 変更可否 | 説明 |
|---|---|---|---|
| `interval` | `10min` | 可（要再起動） | 時間足（5min / 10min / 15min / 30min / 1hour） |
| `candle_days` | 3 | 可 | ローソク足の取得日数 |
| `confirm_lag` | 10 | 可 | 足確定後の待機秒数 |

**銘柄設定（symbols）**

| 銘柄 | min_size | lot_step |
|---|---|---|
| BTC | 0.00001 | 0.00001 |
| ETH | 0.001 | 0.0001 |
| XRP | 1 | 1 |
| SOL | 0.01 | 0.01 |
| DOGE | 10 | 1 |
| ADA | 1 | 1 |

**注文設定（order）**

| パラメータ | 現在値 | 説明 |
|---|---|---|
| `taker_fee` | 0.0005 | 成行手数料率（0.05%） |
| `position_pct` | 0.12 | 1銘柄あたり目安投入率（market regime で上書き） |
| `rebalance_threshold` | 0.05 | リバランスデッドバンド（超過率 5% 以下は売らない） |
| `rebalance_min_jpy` | 500 | リバランス最小売却金額（¥500 未満は売らない） |
| `sell_ratio` | 0.50 | half モードの売却割合 |
| `exit_mode` | `full` | 出口戦略（full / half_repeat / half_once） |

**market regime / risk_budget**

| パラメータ | 現在値 | 状態 |
|---|---|---|
| `min_budget` | 0.0 | 確定 |
| `max_budget` | 1.0 | 確定 |
| `breadth_weight` | 0.40 | ⚠️ 仮値 |
| `top1_weight` | 0.30 | ⚠️ 仮値 |
| `total_weight` | 0.30 | ⚠️ 仮値 |
| `top1_ref` | 0.010 | ⚠️ 仮値 |
| `total_ref` | 0.030 | ⚠️ 仮値 |

**戦略パラメータ（strategy.params）**

| パラメータ | 現在値 | 状態 |
|---|---|---|
| `short_period` | 5 | 確定（Step 9 で最適化予定） |
| `long_period` | 25 | 確定（同上） |
| `rsi_period` | 14 | 確定（同上） |
| `rsi_overbought` | 70 | 確定（同上） |
| `rsi_oversold` | 30 | 確定（同上） |

**リスク管理（risk）**

| パラメータ | 現在値 | 停止スコープ |
|---|---|---|
| `max_daily_loss_pct` | 5.0% | 当日全銘柄 BUY 停止 |
| `max_consecutive_losses` | 5 回 | 該当銘柄のみ停止 |
| `max_drawdown_pct` | 20.0% | 全銘柄 BUY 停止 |
| `max_open_orders` | 1 件 | 本番のみ、該当銘柄停止 |
| `volatility_window` | 10 本 | — |
| `volatility_threshold` | 10.0% | 該当銘柄 BUY 停止 |

---

## 11. 変更履歴

| 日付 | バージョン | 変更内容 |
|---|---|---|
| 2026-04-17 | bot v4.0.0 / ma_rsi v2.0.0 | 初版作成。現行ロジックを全記載 |
| 2026-04-16 | — | exit_mode=full を config フラグ化（full / half_repeat / half_once） |
| 2026-04-16 | — | SELL 時リスクチェック免除（利確・損切りをブロックしない） |
| 2026-04-16 | — | market_prices 事前取得バグ修正（peak_jpy 5.4億膨張の根因解消） |
| 2026-04-16 | — | rebalance sell 実装（完全撤退 alloc=0% 含む） |
| 2026-04-16 | — | risk budget 層実装（breadth/top1/total の3要素、動的総投入率） |
| 2026-04-16 | — | _calc_order_size: JPY残高ベース → portfolio総額ベースに修正 |

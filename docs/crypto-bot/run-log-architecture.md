# ログ保存構造と責務分離

> **策定日:** 2026-04-20
> **ステータス:** 確定（LogLayout 移行完了）

---

## ディレクトリ構成

```
logs/
  {context}/                    ← prod / experiments/exp-001/slots/slot-1 など
    records/                    ← 正本・機械向け記録
      state.json                ← bot 稼働状態（running, positions, portfolio）
      thinking.json             ← サイクルごとの推論記録（銘柄別）
      trade_events/
        YYYY-MM.jsonl           ← 約定観測（月別 JSONL、source of truth）
    summaries/                  ← ダッシュボード表示専用（人間向け）
      summaries.json            ← 旧 events.json 相当
    debug/                      ← 障害調査用生ログ
      bot_YYYYMMDD.log
    cache/                      ← 再生成可能な派生データ
      portfolio_history/
        YYYY-MM-DD.jsonl        ← 日別ポートフォリオ推移
    ai_summaries/               ← AI half-day summary（派生物）
      YYYY-MM-DD_HH.json        ← 08:30 / 20:30 の window summary
```

---

## 責務分離の原則

| ディレクトリ | 役割 | 正本か | 消えてよいか |
|---|---|---|---|
| `records/` | bot の稼働記録・判断の正本 | **正本** | ❌ |
| `summaries/` | ダッシュボード表示用。読み替え可 | 表示用 | △（再生成可能） |
| `debug/` | 障害調査用。業務上の真実にしない | デバッグ用 | ✅ |
| `cache/` | 再生成可能な派生データ | 派生 | ✅ |
| `ai_summaries/` | AI または fallback が生成する要約 | 派生 | ✅ |

---

## 正本の書き込みルール

- **書き込みは常に `records/` へ**。flat（log_dir 直下）には書かない
- `state.json`: bot サイクルごとに上書き
- `thinking.json`: サイクルごとに全銘柄分を上書き
- `trade_events/YYYY-MM.jsonl`: 約定時に追記（append）
- **フォールバックコードなし**: `_resolve_read()` / `migrate_from_flat()` は削除済み

---

## cycle_payload と window_payload

AI サマリー機能のデータフローは `ai-summary-pipeline.md` を参照。

| データ種別 | 正本か | 置き場 |
|---|---|---|
| `cycle_payload` | **正本**（thinking.json の各サイクル記録） | `records/thinking.json` |
| `window_payload` | 集約値（半日分の cycle を集計） | 生成時のみ・永続化不要 |
| `window_summary` | 派生物（AI または fallback） | `ai_summaries/` |

---

## 変更履歴

- 2026-04-20: LogLayout 移行完了。bot.py / dashboard.py ともに records/ を正本として読み書き。旧フラットファイルと fallback コードを全削除

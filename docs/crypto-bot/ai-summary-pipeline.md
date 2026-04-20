# AI Half-Day Summary パイプライン設計

> **策定日:** 2026-04-20
> **ステータス:** 設計確定。実装は Issue #26 で追う

---

## 目的と位置づけ

- **目的**: 半日単位の cycle 集約と要約を自動生成し、観測コストを下げる
- **性質**: observability の補助機能。売買判断の正本ではない
- **制約**: bot 本体の稼働を止めない。AI 失敗時は fallback へ落ちる

---

## データフロー

```
records/thinking.json   ← cycle_payload（正本）
        ↓  半日分を集約
  window_payload         ← 生成時のみ使用。永続化不要
        ↓  AI または fallback が要約
  window_summary         ← 派生物
        ↓  保存
ai_summaries/YYYY-MM-DD_HH.json
```

---

## 各データ種別の定義

### cycle_payload（正本）
- **実体**: `records/thinking.json` の各サイクル記録
- 銘柄ごとのシグナル・判断・数量・market_regime 等
- bot が毎サイクル書き込む。summary プロセスは読み取り専用

### window_payload（集約値）
- **生成タイミング**: 08:30 / 20:30 の固定時刻
- **対象期間**: 直前の half-day window（前 20:30〜当 08:30、または前 08:30〜当 20:30）
- **集約内容**: BUY/SELL 件数、skip 件数・理由、all-negative サイクル比率、portfolio 増減、銘柄ごとの最終状態
- 永続化しない（summary 生成の中間データ）
- **interval が変わっても意味が崩れない**設計: サイクル数ではなく時刻ベースで window を切る

### window_summary（派生物）
- AI が window_payload を自然言語でまとめたもの
- AI 失敗時は fallback（window_payload の数値テキスト化）で代替
- **保存先**: `ai_summaries/YYYY-MM-DD_HH.json`（HH = 08 or 20）
- **フォーマット**:
  ```json
  {
    "window_start": "2026-04-20T20:30:00+09:00",
    "window_end": "2026-04-21T08:30:00+09:00",
    "interval": "1hour",
    "cycle_count": 12,
    "summary_text": "...",
    "generated_by": "ai" | "fallback",
    "window_payload_snapshot": { ... }
  }
  ```

---

## 実行タイミング

- **08:30 JST / 20:30 JST** の固定時刻
- cron または bot 内のタイマーで起動（bot 本体ループとは独立したプロセス推奨）
- 実行タイミングが interval に依存しない → interval が変わっても summary の窓は変わらない

---

## AI 失敗時の fallback

1. AI API タイムアウト / エラー → fallback へ切り替え
2. fallback: window_payload の数値をテンプレートテキスト化
3. `generated_by: "fallback"` で記録
4. bot 本体への影響: **なし**（summary プロセスは独立）

---

## 実装スコープ（Issue #26）

- [ ] window 切り出しロジック（時刻ベース）
- [ ] cycle_payload の読み取り（records/thinking.json から集約）
- [ ] window_payload の構築
- [ ] AI API 呼び出し + fallback
- [ ] ai_summaries/ への保存
- [ ] cron / タイマー設定
- [ ] ダッシュボードへの表示（任意・後回し可）

---

## 参照

- ログ保存構造: `run-log-architecture.md`
- Issue: https://github.com/t-wp/crypt-bot/issues/26

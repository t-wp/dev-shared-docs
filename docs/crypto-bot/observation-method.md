# 観測手順・確認順・レポート方針

> **策定日:** 2026-04-20

---

## 原則

- **発生有無を問われたらログ全文検索を先に行う**（thinking.json / state.json だけを見て答えることは禁止）
- 確認の正本優先順: **実ファイル直読 > MD5 ハッシュ確認 > read_file ツール**
- `read_file` ツールはキャッシュ問題あり。完了報告の根拠には使わない（詳細: user memory の file-verification.md）

---

## ログの正本確認順

```
1. records/thinking.json     — サイクルごとの推論（各銘柄のシグナル・判断・数量）
2. records/trade_events/     — 約定観測の正本（月別 JSONL）
3. debug/bot_YYYYMMDD.log    — 生ログ（全行、障害調査用）
4. summaries/summaries.json  — ダッシュボード表示用（参照は最後でよい）
```

---

## 確認コマンド早見表

```powershell
# ログ全文検索（発生有無の確認）
Select-String -Path "Z:\crypto-bot\logs\prod\debug\bot_*.log" -Pattern "検索キーワード" -Encoding UTF8

# thinking.json の最新状態
Get-Content "Z:\crypto-bot\logs\prod\records\thinking.json" -Encoding UTF8 | ConvertFrom-Json

# state.json の現在状態
Get-Content "Z:\crypto-bot\logs\prod\records\state.json" -Encoding UTF8 | ConvertFrom-Json

# ファイルのタイムスタンプ確認（更新確認）
Get-Item "Z:\crypto-bot\logs\prod\records\state.json" | Select-Object LastWriteTime

# ローカル/NAS ハッシュ一致確認
(Get-FileHash "D:\File\projects\crypto-bot\FILE" -Algorithm MD5).Hash
(Get-FileHash "Z:\crypto-bot\FILE" -Algorithm MD5).Hash
```

```bash
# SSH 経由（NAS 実機）
ssh t102910@192.168.0.148 "tail -20 ~/crypto-bot/logs/prod/debug/bot_20260420.log"
ssh t102910@192.168.0.148 "stat -c '%n %Y %y' ~/crypto-bot/logs/prod/records/state.json"
```

---

## 観測パターン早見表

| 確認したいこと | 検索パターン |
|---|---|
| all-negative 発生 | `全銘柄下降または市場低温` |
| exit sell 発火 | `SELL exit` |
| rebalance sell 発火 | `SELL rebalance` |
| BUY 実行 | `模擬 BUY` または `BUY` |
| 日次損失停止 | `日次損失上限` |
| DD 停止 | `ドローダウン` |
| 停止中スキップ | `停止中` |

---

## interval を変えるときの注意

- interval は bot 起動時の設定で決まる。サイクル途中での変更はしない
- interval を変えたら **別 experiment_id** に切り出す
- 同一 experiment の中で slot ごとに異なる interval を混ぜてはいけない
- interval 比較は「同一 interval 内での param_set 比較」→「interval champion 同士の比較」という段階で行う

---

## レポート時の参照順

1. このファイル（確認手順・コマンド）
2. `observation-log.md`（過去の発生記録）
3. `run-log-architecture.md`（正本の置き場確認）
4. GitHub Issues（実務タスクの判断状況）

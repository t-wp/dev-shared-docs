# dev-shared-docs — Copilot 共通ルール

このリポジトリは複数プロジェクトから共通参照される設計ドキュメント置き場である。
プロジェクト固有の不変ルールは各プロジェクトの `docs/<project>/copilot-instructions.md` に置く。

---

## リポジトリ構成

```
.github/copilot-instructions.md   ← このファイル（共通ルール）
docs/
  crypto-bot/
    copilot-instructions.md       ← crypto-bot 固有の不変ルール ★必読
    current-phase.md              ← 現在フェーズ・作業状況
    roadmap.md                    ← 全体地図とフェーズ定義
    observation-log.md            ← 観測結果の要約
```

---

## プロジェクト別指示

### crypto-bot

crypto-bot の開発時は **必ず** 以下のファイルも参照すること：

- `docs/crypto-bot/copilot-instructions.md` — 不変ルール・設計思想・用語定義
- `docs/crypto-bot/current-phase.md` — 現在フェーズ・今の作業状況（毎回の主役）
- `docs/crypto-bot/roadmap.md` — 全体地図とフェーズ定義
- `docs/crypto-bot/observation-log.md` — 観測結果の要約

---

## 共通ルール

- プロジェクト固有の内容をこのファイルに書かない
- ドキュメントの追加は最小限にする
- `docs/` はコード非依存。コード変更前提の記述を書かない

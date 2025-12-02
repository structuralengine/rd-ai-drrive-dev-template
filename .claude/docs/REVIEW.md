# AI Factory テンプレート レビュー履歴

## 概要

AI Factory は GitHub Issues 駆動の自動開発パイプラインテンプレートです。
このドキュメントは過去のレビュー履歴と解決された問題点を記録しています。

---

## 現在のバージョン

### 完成した機能

1. **8つの専門AIエージェント**
   - Product Manager（要件定義・仕様策定）
   - Critic（仕様書レビュー）
   - Issue Planner（Issue分解）
   - QA Engineer（テスト設計）
   - Coder（実装）
   - Tech Lead（コード監査・変異テスト）
   - Validator（統合検証）
   - Scrum Master（知識蒸留・継続改善）

2. **5つのコマンド**
   - `/prepare` - 企画フェーズ一気通貫
   - `/auto` - 自動連続処理
   - `/run` - 実装パイプライン
   - `/merge` - PRマージ + 統合検証
   - `/kaizen` - 継続的改善（知識蓄積）

3. **4つのルールファイル**
   - `python.md` - Pythonコーディング規約
   - `testing.md` - テストのベストプラクティス
   - `libraries.md` - 外部ライブラリの使い方
   - `architecture.md` - 設計パターン

4. **コード品質保証**
   - 静的解析（ruff/mypy）
   - 変異テスト（mutmut）
   - TDD（テスト駆動開発）

---

## リファクタリング履歴

### 最新のリファクタリング（2025年）

**問題点:**
1. `auto.md` で `claude -p "/run"` を使用していた（headless modeでスラッシュコマンドを呼ぶのは非推奨）
2. 全エージェントに `tools` フィールドがなかった
3. コマンドが別コマンドを呼ぶ複雑な依存関係

**解決内容:**
1. 中間コマンド（/spec, /critique, /breakdown, /test, /impl, /review, /sync）を削除
2. ワークフローコマンドがTask tool経由でエージェントを直接呼び出す構造に変更
3. `/compact` の活用でコンテキストオーバーフローを防止
4. ルールファイルの分類（python, testing, libraries, architecture）

**削除したファイル:**
- コマンド: spec.md, critique.md, breakdown.md, test.md, impl.md, review.md, sync.md
- エージェント: librarian.md

詳細は [REFACTORING_GUIDE.md](REFACTORING_GUIDE.md) を参照。

---

## 現在のファイル構成

```
.claude/
├── agents/                     # 8 AIエージェント
│   ├── pm.md
│   ├── critic.md
│   ├── planner.md
│   ├── qa.md
│   ├── coder.md
│   ├── tech_lead.md
│   ├── validator.md
│   └── scrum_master.md
│
├── commands/                   # 5 コマンド
│   ├── prepare.md
│   ├── auto.md
│   ├── run.md
│   ├── merge.md
│   └── kaizen.md
│
├── rules/                      # 4 ルールファイル
│   ├── python.md
│   ├── testing.md
│   ├── libraries.md
│   └── architecture.md
│
├── docs/                       # ドキュメント
│   ├── AGENTS.md
│   ├── COMMANDS.md
│   ├── WORKFLOW.md
│   ├── QUICKSTART.md
│   ├── USAGE.md
│   ├── REVIEW.md
│   └── REFACTORING_GUIDE.md
│
├── factory/                    # コアシステム
│   ├── memos/
│   └── notify.py
│
└── worktrees/                  # Git Worktree
```

---

## 今後の改善候補

1. **並列処理**: 複数Issueの同時処理
2. **メトリクス**: 開発効率の可視化
3. **カスタムエージェント**: ユーザー定義エージェントのサポート強化

---

**AI Factory** - GitHub Issues駆動の完全自動開発パイプライン

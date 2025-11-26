---
description: Break down specification into issues
---
@Issue_Planner

**仕様書をIssueに分解**

承認された仕様書を読んで、実装可能なIssueに分解して作成します。

**前提**: `/spec` と `/critique` が完了していること

---

## 入力

- `docs/specs/project-*.md` または `docs/specs/spec-*.md`（最新の承認済み仕様書）

---

## 出力

1. **GitHub Issues**: 各機能単位のIssue（`label:todo` 付き）
2. **設計書ファイル**: 各Issueに対応する `docs/specs/feature-{issue_id}.md`

---

## 処理フロー

### ステップ1: 仕様書の読み込み

最新の承認済み仕様書（`docs/specs/project-*.md` または `docs/specs/spec-*.md`）を読み込む。

### ステップ2: 機能分割の設計

仕様書を分析し、以下の観点で機能を分割する：

- **独立性**: 他の機能に依存せず単独で実装・テスト可能
- **適切な粒度**: 1つのIssueで1〜2時間程度で実装可能なサイズ
- **明確な完了条件**: テストで検証可能な成果物
- **依存関係の特定**: 機能間の依存関係を明確にし、実装順序を決定

### ステップ3: GitHub Issue作成

各機能について以下の形式でIssueを作成：

```bash
gh issue create --title "[機能名]" --body "$(cat <<'EOF'
## 概要
[機能の1-2行説明]

## 要件
- [要件1]
- [要件2]

## 受け入れ条件
- [ ] [テスト可能な条件1]
- [ ] [テスト可能な条件2]

## 依存
<!-- 依存関係がある場合のみ記載。ない場合はこのセクションを削除 -->
- Blocked by #[依存先Issue番号]

## 関連
- 仕様書: docs/specs/project-XXX.md
- 設計書: docs/specs/feature-{この Issue番号}.md

---
🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)" --label todo
```

**依存関係の記載ルール**:
- 依存先Issueがある場合は `## 依存` セクションに `Blocked by #N` 形式で記載
- 複数の依存先がある場合は複数行で記載
- 依存関係がない場合は `## 依存` セクション自体を省略

### ステップ4: 設計書ファイル作成

**重要**: Issue作成後、そのIssue番号を使って `docs/specs/feature-{issue_id}.md` を作成する。

```markdown
# Feature {issue_id}: [機能名]

## 概要
[機能の説明]

## 入力仕様
- [入力データの形式・ソース]

## 出力仕様
- [出力データの形式・保存先]

## 処理ロジック
1. [処理ステップ1]
2. [処理ステップ2]

## エラー処理
- [異常系の扱い]

## テスト観点
- [正常系テストケース]
- [異常系テストケース]

## 実装ヒント
- [推奨するモジュール構成]
- [参考にすべき既存コード]

---
🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

---

## 出力例

Issue #5 を作成した場合：

1. **GitHub Issue #5**: タイトルと本文
2. **設計書**: `docs/specs/feature-5.md`

---

## 完了条件

- [ ] 全ての機能がGitHub Issueとして作成されている
- [ ] 各Issueに `todo` ラベルが付いている
- [ ] 各Issueに対応する `docs/specs/feature-{issue_id}.md` が作成されている
- [ ] 設計書に入力仕様・出力仕様・処理ロジック・テスト観点が記載されている
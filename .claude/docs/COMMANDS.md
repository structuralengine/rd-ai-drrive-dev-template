# AI Factory コマンドリファレンス

このガイドでは、AI Factory の5個のコマンドについて説明します。

## 目次

- [コマンド概要](#コマンド概要)
- [推奨ワークフロー](#推奨ワークフロー)
- [各コマンドの詳細](#各コマンドの詳細)
  - [1. /prepare - 企画フェーズ](#1-prepare---企画フェーズ)
  - [2. /auto - 自動連続処理](#2-auto---自動連続処理)
  - [3. /run - 実装パイプライン](#3-run---実装パイプライン)
  - [4. /merge - PRマージ](#4-merge---prマージ)
  - [5. /kaizen - 継続的改善](#5-kaizen---継続的改善)

---

## コマンド概要

| コマンド | 説明 | 引数 | 定義ファイル |
|---------|------|------|------------|
| `/prepare` | 企画フェーズ一気通貫 | なし | [prepare.md](../commands/prepare.md) |
| `/auto` | TODO Issue自動連続処理 | なし | [auto.md](../commands/auto.md) |
| `/run` | 実装パイプライン | Issue ID | [run.md](../commands/run.md) |
| `/merge` | PRマージ + 統合検証 | PR番号（省略可） | [merge.md](../commands/merge.md) |
| `/kaizen` | 学びの記録 | Issue ID | [kaizen.md](../commands/kaizen.md) |

---

## 推奨ワークフロー

```bash
# 1. 企画フェーズ（ヒアリング → 仕様書 → レビュー → Issue分解）
/prepare

# 2. 実装フェーズ（全Issueを自動処理）
/auto

# または、特定のIssueのみ処理
/run 42
```

### フロー図

```
/prepare
   │
   ├── ヒアリング（対話）
   ├── @Product_Manager（仕様書作成）
   ├── @Critic（レビュー）
   └── @Issue_Planner（Issue分解）
         ↓
     GitHub Issues (label: todo)
         ↓
/auto
   │
   └── 各Issueに対して:
         │
         ├── /run {id}
         │     ├── @QA_Engineer（テスト作成）
         │     ├── @Coder（実装）
         │     ├── @Tech_Lead（レビュー）
         │     └── PR作成
         │
         ├── /compact（コンテキスト圧縮）
         │
         ├── /merge
         │     └── @Validator（統合検証）
         │
         └── /compact（コンテキスト圧縮）
```

---

## 各コマンドの詳細

### 1. /prepare - 企画フェーズ

**定義ファイル**: [../commands/prepare.md](../commands/prepare.md)

**シンタックス**:
```bash
/prepare
```

**説明**:
ヒアリング → 仕様書作成 → レビュー → Issue分解 を一連の流れで実行します。

**フロー**:

```
/prepare
│
├── ヒアリングフェーズ（対話あり）
│     必須確認項目:
│     - 入力仕様
│     - 出力仕様
│     - 処理ロジック
│     - エラー処理
│     - スコープ外
│     - 成功基準
│
├── 要件サマリー提示 → 承認待ち
│
└── 自動実行フェーズ（確認なし）
      ├── @Product_Manager → 仕様書作成
      ├── @Critic → レビュー（APPROVED まで繰り返し）
      └── @Issue_Planner → Issue分解
```

**出力**:
- `docs/specs/spec-*.md` - 仕様書
- GitHub Issues（label: `todo`）

---

### 2. /auto - 自動連続処理

**定義ファイル**: [../commands/auto.md](../commands/auto.md)

**シンタックス**:
```bash
/auto
```

**説明**:
`label:todo` かつ `no:assignee` のIssueを全て処理するまで繰り返し実行します。

**フロー**:

```
while (対象Issueあり):
    1. gh issue list で対象Issue検索
    2. 見つからなければ終了
    3. 楽観的ロックでアサイン
    4. /run {issue_id} を実行
    5. /compact でコンテキスト圧縮
    6. /merge を実行
    7. /compact でコンテキスト圧縮
    8. 次のIssueへ
```

**重要**:
- 完全自動実行モード（途中で確認を求めない）
- 各Issue処理後に `/compact` でコンテキスト圧縮
- 楽観的ロックで並列実行時の競合を防止

---

### 3. /run - 実装パイプライン

**定義ファイル**: [../commands/run.md](../commands/run.md)

**シンタックス**:
```bash
/run {issue_id}
```

**説明**:
指定されたIssueに対して、テスト生成から実装、レビュー、PR作成までを一気通貫で実行します。

**フロー**:

| フェーズ | 内容 | エージェント |
|----------|------|-------------|
| 0. 準備 | Issue取得、アサイン、Worktree作成 | - |
| 1. テスト生成 | TDDでテストを先に作成 | @QA_Engineer |
| 2. 実装 | テストを通過するコード作成 | @Coder |
| 3. レビュー | 静的解析、変異テスト | @Tech_Lead |
| 4. PR作成 | commit, push, PR作成 | - |
| 5. Kaizen | 学びの記録 | @Scrum_Master |
| 6. クリーンアップ | Worktree削除 | - |

**成功時**:
- PRが作成される
- Issueに完了コメントが投稿される

**失敗時**:
- Issueに失敗コメントが投稿される
- `failed` ラベルが付与される

---

### 4. /merge - PRマージ

**定義ファイル**: [../commands/merge.md](../commands/merge.md)

**シンタックス**:
```bash
/merge [pr_number]
```

**説明**:
指定されたPR（または最も古いtodo IssueのPR）をmainにマージし、統合検証を実行します。

**フロー**:

| フェーズ | 内容 |
|----------|------|
| 1. PR選択 | 引数またはtodo Issueから自動選択 |
| 2. マージ前チェック | state, mergeable 確認 |
| 3. マージ実行 | `gh pr merge --merge --delete-branch` |
| 4. 状態更新 | ラベル更新 |
| 5. 統合検証 | @Validator による pytest/ruff/mypy |

**検証失敗時**:
- バグIssue が自動作成される（label: `bug`, `todo`）

---

### 5. /kaizen - 継続的改善

**定義ファイル**: [../commands/kaizen.md](../commands/kaizen.md)

**シンタックス**:
```bash
/kaizen {issue_id}
```

**説明**:
指定されたIssueの開発プロセスで得られた学びを収集し、適切なルールファイルに蓄積します。

**入力**:
- `../factory/memos/issue-{id}-*.md` - 各エージェントの作業メモ

**出力先の分類**:

| 学びの内容 | 出力先 |
|------------|--------|
| Python文法・スタイル | `../rules/python.md` |
| テスト（pytest、モック） | `../rules/testing.md` |
| 外部ライブラリの使い方 | `../rules/libraries.md` |
| 設計・アーキテクチャ | `../rules/architecture.md` |
| プロジェクト固有の方針 | `CLAUDE.md`（ルート） |

**判断基準**:
- **追加する**: 次回以降も役立つ汎用的な知識
- **追加しない**: 今回のIssue固有の内容

---

## 次のステップ

- [AGENTS.md](AGENTS.md) - 各エージェントの詳細
- [WORKFLOW.md](WORKFLOW.md) - コマンドがどのように連携するか

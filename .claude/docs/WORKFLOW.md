# AI Factory ワークフロー詳細

このガイドでは、AI Factory の内部動作を技術的に説明します。

## 目次

- [全体アーキテクチャ](#全体アーキテクチャ)
- [企画フェーズ（/prepare）](#企画フェーズprepare)
- [実装フェーズ（/run, /auto）](#実装フェーズrun-auto)
- [マージフェーズ（/merge）](#マージフェーズmerge)
- [エラーハンドリング](#エラーハンドリング)
- [コンテキスト管理](#コンテキスト管理)

---

## 全体アーキテクチャ

### システム構成

```
.claude/
├── commands/        # 5コマンド
│   ├── prepare.md   # 企画フェーズ
│   ├── auto.md      # 自動連続処理
│   ├── run.md       # 実装パイプライン
│   ├── merge.md     # PRマージ
│   └── kaizen.md    # 学びの記録
│
├── agents/          # 8エージェント
│   ├── pm.md        # Product Manager
│   ├── critic.md    # Critic
│   ├── planner.md   # Issue Planner
│   ├── qa.md        # QA Engineer
│   ├── coder.md     # Coder
│   ├── tech_lead.md # Tech Lead
│   ├── validator.md # Validator
│   └── scrum_master.md # Scrum Master
│
├── rules/           # 4ルールファイル
│   ├── python.md    # Pythonコーディング規約
│   ├── testing.md   # テストのベストプラクティス
│   ├── libraries.md # 外部ライブラリの使い方
│   └── architecture.md # 設計パターン
│
├── docs/            # ドキュメント（このファイル）
│   ├── AGENTS.md
│   ├── COMMANDS.md
│   ├── WORKFLOW.md
│   └── REFACTORING_GUIDE.md
│
└── factory/         # 自動化ツール
    ├── memos/       # 作業メモ
    └── notify.py    # Slack通知
```

### 3層アーキテクチャ

```
┌─────────────────────────────────────────────────┐
│ Layer 1: オーケストレーション                    │
│   /prepare, /auto                               │
│   - 複数コマンド/エージェントの調整               │
│   - /compact によるコンテキスト管理              │
└─────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│ Layer 2: ワークフロー                            │
│   /run, /merge, /kaizen                         │
│   - 1つのタスクの完全な処理フロー                │
│   - エージェントを直接呼び出し（Task tool経由）   │
└─────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│ Layer 3: 専門家（エージェント）                  │
│   @Product_Manager, @Coder, @QA_Engineer, etc.  │
│   - 専門的な判断と実行                          │
│   - 独立したコンテキスト                         │
└─────────────────────────────────────────────────┘
```

---

## 企画フェーズ（/prepare）

### フロー図

```
/prepare
│
├── ヒアリング（対話）
│     │
│     ├── 入力仕様を確認
│     ├── 出力仕様を確認
│     ├── 処理ロジックを確認
│     ├── エラー処理を確認
│     ├── スコープ外を確認
│     └── 成功基準を確認
│
├── 要件サマリー → 承認待ち
│
└── 自動実行（確認なし）
      │
      ├── @Product_Manager（Task tool経由）
      │     └── docs/specs/spec-*.md 作成
      │
      ├── @Critic（Task tool経由）
      │     ├── APPROVED → 次へ
      │     └── REQUEST_CHANGES → @Product_Manager に修正依頼（最大3回）
      │
      └── @Issue_Planner（Task tool経由）
            └── gh issue create --label todo
```

### 出力

- 仕様書: `docs/specs/spec-{timestamp}.md`
- GitHub Issues（label: `todo`）

---

## 実装フェーズ（/run, /auto）

### /run のフロー

```
/run {issue_id}
│
├── フェーズ0: 準備
│     ├── gh issue view {id}
│     ├── 依存関係チェック（Blocked by #N）
│     └── gh issue edit --add-assignee @me
│
├── フェーズ1: Worktree作成
│     └── git worktree add .claude/worktrees/task-{id} -b feature/issue-{id}
│
├── フェーズ2: テスト生成（TDD）
│     └── @QA_Engineer（Task tool経由）
│           └── tests/test_feature_{id}.py 作成
│
├── フェーズ3: 実装
│     └── @Coder（Task tool経由）
│           ├── src/ 配下に実装
│           └── pytest → 失敗なら修正（最大3回）
│
├── フェーズ4: レビュー
│     └── @Tech_Lead（Task tool経由）
│           ├── ruff check
│           ├── mypy
│           ├── mutmut（変異テスト）
│           └── gh issue comment（結果投稿）
│
├── フェーズ5: PR作成
│     ├── git add -A && git commit
│     ├── git push --force-with-lease
│     ├── gh pr create
│     └── gh issue comment（完了報告）
│
├── フェーズ6: Kaizen
│     └── @Scrum_Master（Task tool経由）
│           └── 学びを ../rules/*.md に記録
│
└── フェーズ7: クリーンアップ
      └── git worktree remove
```

### /auto のフロー

```
/auto
│
└── while (対象Issueあり):
      │
      ├── 1. gh issue list --search "label:todo no:assignee"
      │
      ├── 2. 見つからなければ終了
      │
      ├── 3. 楽観的ロック
      │     ├── アサイン前確認
      │     ├── gh issue edit --add-assignee @me
      │     ├── 2秒待機
      │     └── 再確認（競合検出）
      │
      ├── 4. /run {issue_id}
      │
      ├── 5. /compact（コンテキスト圧縮）
      │
      ├── 6. /merge（/run成功時のみ）
      │
      ├── 7. /compact（コンテキスト圧縮）
      │
      └── 8. 次のIssueへ
```

### Git Worktree

各Issueごとに独立した作業環境を作成：

```
.claude/worktrees/
└── task-{id}/
    ├── .git         # worktree専用
    ├── docs/
    ├── src/
    └── tests/
```

**メリット**:
- メインブランチを汚さない
- 失敗時のクリーンアップが容易
- 将来的に並列実行が可能

---

## マージフェーズ（/merge）

### フロー

```
/merge [pr_number]
│
├── フェーズ1: PR選択
│     ├── 引数あり → 指定PR
│     └── 引数なし → 最も古いtodo IssueのPR
│
├── フェーズ2: マージ前チェック
│     ├── state == OPEN
│     └── mergeable == MERGEABLE
│
├── フェーズ3: マージ実行
│     └── gh pr merge --merge --delete-branch
│
├── フェーズ4: 状態更新
│     └── gh issue edit --remove-label todo
│
└── フェーズ5: 統合検証
      └── @Validator（Task tool経由）
            ├── git checkout main && git pull
            ├── uv run pytest
            ├── uv run ruff check
            ├── uv run mypy
            │
            ├── 成功 → PASSED
            └── 失敗 → バグIssue作成
```

---

## エラーハンドリング

### /run のエラー処理

| エラー | 対応 |
|--------|------|
| Worktree作成失敗 | エラー終了、Issueに報告 |
| テスト生成失敗 | エラー終了、Issueに報告 |
| 実装失敗（3回リトライ後） | エラー終了、Issueに報告 |
| レビュー失敗 | @Coder に修正依頼 |
| Push失敗 | エラー終了、Issueに報告 |

**失敗時の処理**:
1. Issueに失敗コメント投稿
2. `todo` ラベル削除
3. `failed` ラベル付与
4. Worktreeクリーンアップ

### /merge のエラー処理

| エラー | 対応 |
|--------|------|
| PRが見つからない | エラー終了 |
| マージ不可（コンフリクト） | エラー終了 |
| 統合検証失敗 | バグIssue作成、処理続行 |

---

## コンテキスト管理

### /compact の重要性

長時間実行時にコンテキストが肥大化するのを防止：

```
Issue #1 処理
  └── ログが蓄積
        ↓
/compact（圧縮）
        ↓
Issue #2 処理
  └── 新しいログ
        ↓
/compact（圧縮）
        ↓
...
```

### サブエージェントのコンテキスト

Task tool 経由で呼ばれたエージェントは独立したコンテキストを持つ：

```
[親] /run
    ├── 親のコンテキスト（ログが蓄積）
    │
    └── Task(@QA_Engineer)
          └── 独自のコンテキスト（親とは分離）
```

**注意**: サブエージェントはサブエージェントを呼べない（1階層のみ）

---

## 次のステップ

- [AGENTS.md](AGENTS.md) - 各エージェントの詳細
- [COMMANDS.md](COMMANDS.md) - コマンドリファレンス
- [REFACTORING_GUIDE.md](REFACTORING_GUIDE.md) - 設計の背景と理由

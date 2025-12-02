---
description: Implement the feature
arguments:
  - name: id
    description: Issue ID to implement
    required: true
---

**Issue #{$ARGUMENTS} の実装パイプライン実行**

指定されたIssueに対して、テスト生成から実装、レビュー、PR作成までを一気通貫で実行します。

## 重要: 完全自動実行モード（確認なし）

このコマンドは **一気通貫で自動実行** します。**途中でユーザーに確認を求めない。**

**禁止事項**:
- 「この実装でよいですか？」と聞かない
- 「PRを作成してよいですか？」と聞かない
- 「続行しますか？」と聞かない
- エラー以外の理由で停止しない

---

## Slack通知

各フェーズの開始・完了時にSlack通知を送信する。

```bash
python .claude/factory/notify.py "メッセージ" --level info || true
```

---

## フェーズ0: 準備

**Slack通知**: 🎯 タスク開始 #{issue_id}

1. `gh issue view {issue_id}` でIssueの内容を取得
2. Issueが存在し、openであることを確認
3. **依存関係チェック**: `Blocked by #N` がある場合、#N がクローズ済みか確認
   - 未完了なら: アサイン解除してスキップ
4. `gh issue edit {issue_id} --add-assignee @me` で自分をアサイン

---

## フェーズ1: Git Worktree作成

`.claude/worktrees/task-{issue_id}` にworktreeを作成。

```bash
git worktree add .claude/worktrees/task-{issue_id} -b feature/issue-{issue_id}
```

以降の作業はこのworktree内で実行。

---

## フェーズ2: テスト生成（TDD）

**@QA_Engineer** にテストコード作成を依頼する（Task tool経由でサブエージェントとして呼び出し）。

**依頼内容**:
- Issue #{issue_id} の内容を渡す
- `tests/` ディレクトリにテストファイルを作成
- 正常系・異常系・境界値テストを含める

**確認**:
```bash
uv run pytest --collect-only  # 構文エラーがないことを確認
```

---

## フェーズ3: 実装

**@Coder** に実装を依頼する（Task tool経由でサブエージェントとして呼び出し）。

**依頼内容**:
- Issue #{issue_id} の内容を渡す
- 作成されたテストファイルのパスを渡す
- テストを通過する実装を作成

**テスト実行**:
```bash
uv run pytest -v
```

**テスト失敗の場合**: @Coder に修正を依頼（最大3回リトライ）

---

## フェーズ4: コードレビュー

**@Tech_Lead** にコードレビューを依頼する（Task tool経由でサブエージェントとして呼び出し）。

**依頼内容**:
- 実装されたコードのレビュー
- 静的解析（ruff, mypy）の実行
- 変異テスト（mutmut）の実行（可能であれば）

**問題がある場合**: @Coder に修正を依頼

---

## フェーズ5: PR作成

### 5-1. 変更をコミット

```bash
git add -A
git commit -m "feat: Resolve #{issue_id} - [Issueタイトル]"
```

### 5-2. プッシュ

```bash
git push origin HEAD --force-with-lease
```

### 5-3. PR作成

```bash
gh pr create --title "feat: Resolve #{issue_id}" --body "..."
```

### 5-4. Issueに完了コメントを投稿

```bash
gh issue comment {issue_id} --body "## ✅ 実装完了\nPR #{pr_number} を作成しました。"
```

**Slack通知**: 🚀 PR作成完了 #{issue_id}

---

## フェーズ6: Kaizen

**@Scrum_Master** に学びの記録を依頼する（Task tool経由でサブエージェントとして呼び出し）。

**依頼内容**:
- 作業中の学びを収集
- 汎用的な学びを適切なルールファイルに追記
  - Python関連 → `.claude/rules/python.md`
  - テスト関連 → `.claude/rules/testing.md`
  - ライブラリ関連 → `.claude/rules/libraries.md`
  - 設計関連 → `.claude/rules/architecture.md`

---

## フェーズ7: クリーンアップ

worktreeを削除:
```bash
git worktree remove .claude/worktrees/task-{issue_id}
git worktree prune
```

---

## エラー時の動作

各フェーズで失敗した場合:

1. **Issueに失敗コメントを投稿**
2. **ラベル変更**: `todo` → `failed`
3. **worktreeクリーンアップ**

**Slack通知**: ❌ 実装失敗 #{issue_id}

---

## 最終報告（全フェーズ完了後）

1. **処理したIssue**: #{issue_id}
2. **作成したPR**: PR番号とURL
3. **テスト結果**: pass/fail
4. **レビュー結果**: 指摘事項と対応
5. **Kaizen**: 記録した学び

## 完了条件

- PRが作成されている
- Issueに完了コメントが投稿されている
- worktreeがクリーンアップされている

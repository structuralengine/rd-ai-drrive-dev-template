---
description: Merge PR to main branch
arguments:
  - name: pr_number
    description: PR number to merge (optional, defaults to oldest todo issue's PR)
    required: false
---

**PRをmainブランチにマージ**

指定されたPR（または最も古いtodo IssueのPR）をmainにマージし、統合検証を実行します。

## 重要: 完全自動実行モード（確認なし）

このコマンドは **一気通貫で自動実行** します。**途中でユーザーに確認を求めない。**

**禁止事項**:
- 「マージしてよいですか？」と聞かない
- 「続行しますか？」と聞かない
- エラー以外の理由で停止しない

---

## フェーズ1: PR選択

### 引数ありの場合
指定されたPR番号を使用。

### 引数なしの場合
最も若いIssue番号のPRを自動選択：

```bash
ISSUE_NUMBER=$(gh issue list --search "label:todo state:open" --json number --jq ".[0].number")
PR_NUMBER=$(gh pr list --search "closes #${ISSUE_NUMBER}" --json number --jq ".[0].number")
```

PRが見つからない場合はエラー終了。

---

## フェーズ2: マージ前チェック

1. PRのステータス確認
   ```bash
   gh pr view ${PR_NUMBER} --json state,mergeable,mergeStateStatus
   ```
   - `state` が `OPEN` であること
   - `mergeable` が `MERGEABLE` であること

2. CIチェックの確認（存在する場合）

問題がある場合はエラー終了（マージしない）。

---

## フェーズ3: マージ実行

```bash
gh pr merge ${PR_NUMBER} --merge --delete-branch
```

**Slack通知**: 🔀 マージ完了 PR #${PR_NUMBER}

---

## フェーズ4: 状態更新

### 4-1. Issue のクローズ確認

PRに `Closes #N` が含まれていれば、GitHubが自動でIssueをクローズする。

### 4-2. ラベル更新

```bash
gh issue edit ${ISSUE_NUMBER} --remove-label todo
```

---

## フェーズ5: 統合検証

**@Validator** に統合検証を依頼する（Task tool経由でサブエージェントとして呼び出し）。

**依頼内容**:
- mainブランチに切り替えて検証
- テスト実行（pytest）
- 静的解析（ruff, mypy）
- インポート確認

### 検証成功時

**Slack通知**: ✅ 統合検証完了 PR #${PR_NUMBER}

### 検証失敗時

**バグIssueを作成**：

```bash
gh issue create --title "🐛 マージ後検証エラー (PR #${PR_NUMBER})" --body "..." --label bug --label todo
```

**Slack通知**: ❌ 統合検証失敗 PR #${PR_NUMBER}

---

## 最終報告（全フェーズ完了後）

1. **マージしたPR**: PR番号とURL
2. **クローズしたIssue**: Issue番号
3. **検証結果**: PASSED/FAILED
4. **作成したバグIssue**: （失敗時のみ）

---

## エラー時の動作

| エラー | 対応 |
|--------|------|
| PRが見つからない | エラー終了、ユーザーに報告 |
| マージ不可（コンフリクト等） | エラー終了、ユーザーに報告 |
| マージ失敗 | エラー終了、ユーザーに報告 |
| 検証失敗 | バグIssue作成、処理続行 |

---

## 完了条件

- PRがmainにマージされている
- 関連Issueがクローズされている
- 統合検証が実行されている（成功/失敗問わず）

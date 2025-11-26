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

- ユーザーへの確認なしで全フェーズを連続実行
- 途中で停止しない（エラー時を除く）
- 全フェーズ完了後に最終報告を行う

**禁止事項**:
- 「マージしてよいですか？」と聞かない
- 「続行しますか？」と聞かない
- エラー以外の理由で停止しない

---

## Slack通知

各フェーズの開始・完了時にSlack通知を送信する。
環境変数 `AI_FACTORY_WEBHOOK` が設定されている場合のみ動作。

```bash
python .claude/factory/notify.py "メッセージ" --level info || true
```

---

## フェーズ1: PR選択

### 引数ありの場合
指定されたPR番号を使用。

### 引数なしの場合
最も若いIssue番号のPRを自動選択：

```bash
# label:todo のIssueで、PRが存在するものを検索
ISSUE_NUMBER=$(gh issue list --search "label:todo state:open" --json number --jq ".[0].number")

# IssueにリンクされたPRを取得
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
   ```bash
   gh pr checks ${PR_NUMBER}
   ```

問題がある場合はエラー終了（マージしない）。

---

## フェーズ3: マージ実行

```bash
gh pr merge ${PR_NUMBER} --merge --delete-branch
```

- `--merge`: マージコミットを作成
- `--delete-branch`: マージ後にブランチを削除

**Slack通知**: 🔀 マージ完了 PR #${PR_NUMBER}

```bash
python .claude/factory/notify.py "🔀 マージ完了 PR #${PR_NUMBER}" --title "PR #${PR_NUMBER}" --level info || true
```

---

## フェーズ4: 状態更新

### 4-1. Issue のクローズ確認

PRに `Closes #N` が含まれていれば、GitHubが自動でIssueをクローズする。
念のため確認：

```bash
gh issue view ${ISSUE_NUMBER} --json state --jq '.state'
# → "CLOSED" であることを確認
```

### 4-2. ラベル更新

```bash
gh issue edit ${ISSUE_NUMBER} --remove-label todo
```

---

## フェーズ5: ドキュメント同期

`/sync` コマンドを実行してドキュメントを更新する。

- アーキテクチャドキュメントを更新
- 必要に応じてREADMEを更新

```bash
# mainブランチに切り替え
git checkout main
git pull origin main
```

`@Librarian` がドキュメントを更新：
- 入力: `src/`
- 出力: `docs/architecture/current_system.md`

更新があればコミット：
```bash
git add docs/
git commit -m "docs: Update architecture documentation after PR #${PR_NUMBER}"
git push origin main
```

---

## フェーズ6: 統合検証

`@Validator` による統合検証を実行。

### 検証項目

1. **テスト実行**
   ```bash
   uv run pytest -v --tb=short
   ```

2. **静的解析**
   ```bash
   uv run ruff check src/
   uv run mypy src/
   ```

3. **統合テスト**
   ```bash
   uv run python -c "from src import main; print('Import OK')"
   ```

4. **回帰テスト**
   - 既存のテストスイート全体を実行

### 検証成功時

**Slack通知**: ✅ 統合検証完了 PR #${PR_NUMBER}

```bash
python .claude/factory/notify.py "✅ 統合検証完了 PR #${PR_NUMBER}" --title "PR #${PR_NUMBER}" --level success || true
```

### 検証失敗時

**バグIssueを作成**：

```bash
gh issue create --title "🐛 マージ後検証エラー (PR #${PR_NUMBER})" --body "$(cat <<'EOF'
## 概要
PR #${PR_NUMBER} のマージ後、統合検証で問題が発見されました。

## 元のPR/Issue
- PR: #${PR_NUMBER}
- Issue: #${ISSUE_NUMBER}

## 検証結果
- pytest: [passed/failed]
- ruff: [passed/failed]
- mypy: [passed/failed]

## エラー詳細
```
[エラーメッセージ]
```

## 再現手順
1. main ブランチをチェックアウト
2. `uv run pytest` を実行

## 推奨アクション
- [修正のヒント]

---
🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)" --label bug --label todo
```

**Slack通知**: ❌ 統合検証失敗 PR #${PR_NUMBER}

```bash
python .claude/factory/notify.py "❌ 統合検証失敗 PR #${PR_NUMBER}" --title "PR #${PR_NUMBER}" --level error || true
```

---

## 最終報告（全フェーズ完了後）

全フェーズ完了後、ユーザーに以下を報告:

1. **マージしたPR**: PR番号とURL
2. **クローズしたIssue**: Issue番号
3. **ドキュメント更新**: 更新したファイル一覧
4. **検証結果**: PASSED/FAILED
5. **作成したバグIssue**: （失敗時のみ）

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
- ドキュメントが更新されている
- 統合検証が実行されている（成功/失敗問わず）

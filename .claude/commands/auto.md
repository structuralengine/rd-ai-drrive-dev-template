---
description: Auto-process all todo issues
---

**TODO Issueの自動連続処理**

`label:todo` かつ `no:assignee` のIssueを若い番号順に取得し、すべて処理するまで繰り返し実行します。

## 重要: 完全自動実行モード（確認なし）

このコマンドは **完全自動で実行** します。**途中でユーザーに確認を求めない。**

**禁止事項**:
- 「続行してよいですか？」と聞かない
- 「次のIssueに進みますか？」と聞かない
- エラー以外の理由で停止しない

---

## 処理フロー

```
while (対象Issueあり):
    1. gh issue list で対象Issue検索
    2. 見つからなければ終了
    3. 楽観的ロックでアサイン
    4. /run {issue_id} を実行
    5. /compact でコンテキストを圧縮
    6. /merge を実行（/run成功時）
    7. /compact でコンテキストを圧縮
    8. 次のIssueへ
```

**重要**: 各Issue処理後に `/compact` を実行してコンテキストを圧縮する。
これにより長時間実行でもコンテキストオーバーフローを防ぐ。

---

## ステップ0: 開始通知

```bash
python .claude/factory/notify.py "🏭 自動処理開始" --title "AI Factory" --level info || true
```

---

## ステップ1: 対象Issue検索

```bash
gh issue list --search "label:todo no:assignee state:open sort:created-asc" --limit 1 --json number --jq ".[0].number"
```

- `label:todo` - todoラベル付き
- `no:assignee` - 未アサイン
- `state:open` - オープン状態
- `sort:created-asc` - 作成日昇順（若い番号順）

---

## ステップ2: 終了判定

Issueが見つからない場合（空文字列が返る場合）、**ループを終了して最終報告へ**。

---

## ステップ3: 楽観的ロック（競合防止）

並列実行時に同じIssueを取り合わないようにする:

```bash
# 1. アサイン前に確認
gh issue view {id} --json assignees --jq ".assignees | length"
# → "0" でなければスキップして次のIssueへ

# 2. 自分をアサイン
gh issue edit {id} --add-assignee @me

# 3. 2秒待機後に再確認
sleep 2
gh issue view {id} --json assignees --jq ".assignees | length"
# → "1" でなければ競合発生、アサイン取り消してスキップ
```

---

## ステップ4: /run 実行（同一セッション内）

アサイン成功後、`/run {issue_id}` を直接実行:

```
/run {issue_id}
```

**重要**: `/run` はエージェントを Task tool 経由で呼び出すため、
サブエージェントのコンテキストは独立している。
ただし、親コンテキストにはログが蓄積されるため、
処理完了後に `/compact` で圧縮する。

---

## ステップ5: コンテキスト圧縮

`/run` 完了後（成功・失敗問わず）、コンテキストを圧縮:

```
/compact
```

これにより:
- 処理ログが要約される
- コンテキストウィンドウの消費を抑える
- 長時間実行でも安定動作

---

## ステップ6: /merge 実行（/run成功時のみ）

`/run` が成功した場合、`/merge` を実行:

```
/merge
```

処理内容:
- 対象PRを自動選択（最も古いtodo IssueのPR）
- mainブランチへマージ
- 統合検証（@Validator）
- Issueクローズ、ラベル更新

**`/run` 失敗時は `/merge` をスキップ。**

---

## ステップ7: コンテキスト圧縮（再度）

`/merge` 完了後も `/compact` を実行:

```
/compact
```

---

## ステップ8: 次のIssueへ

**ステップ1に戻って次のIssueを検索。**

---

## エラーハンドリング

| エラー | 対応 | Issueへの記録 |
|--------|------|--------------|
| アサイン失敗 | スキップして次のIssueへ | なし（競合のため） |
| `/run` 失敗 | 次のIssueへ | 失敗コメント + `failed`ラベル |
| GitHub CLI認証エラー | 処理を中断して最終報告へ | なし |

---

## 並列実行

複数ターミナルで `/auto` を同時実行可能。楽観的ロックにより、同じIssueを重複処理しない。

```bash
# ターミナル1
/auto

# ターミナル2（別セッション）
/auto
```

---

## 最終報告（全Issue処理完了後）

**Slack通知**: 📊 自動処理完了

```bash
python .claude/factory/notify.py "📊 自動処理完了: 成功 N件 / 失敗 M件" --title "AI Factory" --level success || true
```

全Issue処理完了後、ユーザーに以下を報告:

1. **処理したIssue数**: N件
2. **成功したIssue**: 一覧（Issue番号、PR番号）
3. **失敗したIssue**: 一覧（Issue番号、失敗理由）
4. **残りのTODO Issue**: あれば一覧

## 完了条件

- `label:todo` かつ `no:assignee` のIssueが0件になる
- または、GitHub CLI認証エラーが発生する

---
description: Kaizen - Record learnings to rule files
arguments:
  - name: id
    description: Issue ID
    required: true
---

**Issue #{$ARGUMENTS} の学びを記録**

@Scrum_Master に学びの収集と記録を依頼する（Task tool経由でサブエージェントとして呼び出し）。

---

## 入力

- `.claude/factory/memos/issue-{id}-*.md` （各エージェントの作業メモ）
- Issue #{id} の処理中に得られた知見

---

## 出力先の分類

学びの内容に応じて、適切なルールファイルに追記する：

| 学びの内容 | 出力先 |
|------------|--------|
| Python文法・スタイル・ベストプラクティス | `.claude/rules/python.md` |
| テスト（pytest、モック、パターン） | `.claude/rules/testing.md` |
| 外部ライブラリの使い方・注意点 | `.claude/rules/libraries.md` |
| 設計・アーキテクチャ・パターン | `.claude/rules/architecture.md` |
| プロジェクト固有の方針・ルール | `CLAUDE.md`（ルート） |

---

## 判断基準

### 追加する学び

- 次回以降も役立つ汎用的な知識
- 特定のライブラリやツールの注意点
- よくあるエラーとその解決策
- ベストプラクティス

**例**:
- 「pydantic v2 では `Field(default_factory=list)` の書き方が変更された」→ `libraries.md`
- 「`conftest.py` にfixtureを書くと複数テストファイルで共有できる」→ `testing.md`
- 「循環インポートを避けるために型ヒントは `TYPE_CHECKING` ブロック内に書く」→ `python.md`

### 追加しない学び

- 今回のIssue固有の内容
- 一時的なワークアラウンド
- 個人的な好み

**例**:
- 「Issue #42 のバグを修正した」→ 追加しない
- 「今回はテストを3回リトライした」→ 追加しない

---

## フォーマット

各ルールファイルへの追記は、既存のセクションに合わせて記載する。

**例（testing.md への追記）**:

```markdown
## pytest

- fixtureは `conftest.py` に書く
- `@pytest.mark.parametrize` で境界値テストを網羅
- **【NEW】** `pytest.raises` でコンテキストマネージャを使うとエラーメッセージも検証できる
```

---

## 完了条件

- 作業メモが分析されている
- 汎用的な学びが適切なルールファイルに追記されている
- Issue固有の内容は除外されている

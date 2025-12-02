# テストのベストプラクティス

## pytest

- fixtureは `conftest.py` に書く
- `@pytest.mark.parametrize` で境界値テストを網羅
- テスト名は日本語可（わかりやすさ優先）

## モック

- 外部API呼び出しは必ずモック化
- `unittest.mock.patch` のスコープに注意
- `Mock` の `return_value` と `side_effect` を使い分ける

## 構造

- Arrange-Act-Assert パターンを使用
- Given-When-Then スタイルのDocstring
- 1テストケース = 1つの検証項目

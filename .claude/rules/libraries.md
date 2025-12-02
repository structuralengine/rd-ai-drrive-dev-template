# 外部ライブラリのベストプラクティス

## HTTP クライアント

- `requests` より `httpx` を推奨（async対応）

## データ検証

- `pydantic` v2 を使用
- `Field(default_factory=list)` の書き方に注意

## 日付・時刻

- `datetime` より `pendulum` を検討（タイムゾーン処理が容易）

## パス操作

- `os.path` より `pathlib.Path` を使用

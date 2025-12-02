# AI Factory エージェントガイド

このガイドでは、AI Factory の8つの専門AIエージェントについて説明します。

## 目次

- [エージェント概要](#エージェント概要)
- [各エージェントの詳細](#各エージェントの詳細)
  - [1. Product Manager](#1-product-manager)
  - [2. Critic](#2-critic)
  - [3. Issue Planner](#3-issue-planner)
  - [4. QA Engineer](#4-qa-engineer)
  - [5. Coder](#5-coder)
  - [6. Tech Lead](#6-tech-lead)
  - [7. Validator](#7-validator)
  - [8. Scrum Master](#8-scrum-master)

---

## エージェント概要

AI Factory は8つの専門AIエージェントが協調して動作します。

### エージェント一覧

| エージェント | 役割 | フェーズ | 定義ファイル |
|------------|------|---------|------------|
| **Product Manager** | 要件分析・仕様策定 | 企画 | [pm.md](../agents/pm.md) |
| **Critic** | 仕様書レビュー | 企画 | [critic.md](../agents/critic.md) |
| **Issue Planner** | 仕様書のIssue分解 | 企画 | [planner.md](../agents/planner.md) |
| **QA Engineer** | テスト設計・実装 | 実装 | [qa.md](../agents/qa.md) |
| **Coder** | 機能実装 | 実装 | [coder.md](../agents/coder.md) |
| **Tech Lead** | コード監査・変異テスト | 実装 | [tech_lead.md](../agents/tech_lead.md) |
| **Validator** | マージ後統合検証 | マージ | [validator.md](../agents/validator.md) |
| **Scrum Master** | 知識蒸留・継続改善 | 実装 | [scrum_master.md](../agents/scrum_master.md) |

### ワークフロー図

```
企画フェーズ（/prepare）
┌─────────────────────────────────────────────┐
│  @Product_Manager → @Critic → @Issue_Planner │
│       仕様書作成      レビュー     Issue分解   │
└─────────────────────────────────────────────┘
                    ↓
実装フェーズ（/run）
┌─────────────────────────────────────────────┐
│  @QA_Engineer → @Coder → @Tech_Lead          │
│    テスト作成     実装     コードレビュー      │
└─────────────────────────────────────────────┘
                    ↓
マージフェーズ（/merge）
┌─────────────────────────────────────────────┐
│  @Validator                                  │
│    統合検証                                  │
└─────────────────────────────────────────────┘
                    ↓
改善フェーズ（/kaizen）
┌─────────────────────────────────────────────┐
│  @Scrum_Master                              │
│    学びの記録                                │
└─────────────────────────────────────────────┘
```

---

## 各エージェントの詳細

### 1. Product Manager

**定義ファイル**: [../agents/pm.md](../agents/pm.md)

**役割**:
- ユーザーの要望を詳しくヒアリング
- 曖昧な要件を具体化し、エッジケースを洗い出す
- `docs/specs/spec-{timestamp}.md` に仕様書を作成

**使用ツール**: Read, Write, Edit, Bash

**出力ファイル**: `docs/specs/spec-*.md`

---

### 2. Critic

**定義ファイル**: [../agents/critic.md](../agents/critic.md)

**役割**:
- 仕様書をレビューし、欠陥や改善点を指摘
- 妥協のない品質基準を維持
- `APPROVED` または `REQUEST_CHANGES` を判定

**使用ツール**: Read

**判定基準**:
- 明確さ: 曖昧な表現がないか
- 実現可能性: 技術的に実装可能か
- 一貫性: 矛盾がないか
- 網羅性: エッジケースの考慮

---

### 3. Issue Planner

**定義ファイル**: [../agents/planner.md](../agents/planner.md)

**役割**:
- 承認された仕様書を読む
- 実装可能な単位に分解（1 Issue = 1日以内）
- `gh issue create --label todo` でIssue作成

**使用ツール**: Read, Bash

**出力**: GitHub Issues（label: `todo`）

---

### 4. QA Engineer

**定義ファイル**: [../agents/qa.md](../agents/qa.md)

**役割**:
- Issueと仕様書に基づき、網羅的なテストケースを作成
- 正常系・異常系・境界値テストを重視
- `tests/` ディレクトリにpytestテストコードを作成

**使用ツール**: Read, Write, Edit, Bash

**出力ファイル**: `tests/test_feature_*.py`

**テスト設計の原則**:
- Arrange-Act-Assert パターン
- 1テストケース = 1つの検証項目
- 外部依存はモック化

---

### 5. Coder

**定義ファイル**: [../agents/coder.md](../agents/coder.md)

**役割**:
- Issueと仕様書に基づき、高品質なコードを実装
- テストを通過するコードを作成
- コーディング規約（`../rules/python.md`）を遵守

**使用ツール**: Read, Write, Edit, Bash, Grep, Glob

**出力ファイル**: `src/*`

**制約**:
- PEP 8 準拠
- Google-style Docstring（日本語）
- タイプヒント必須

---

### 6. Tech Lead

**定義ファイル**: [../agents/tech_lead.md](../agents/tech_lead.md)

**役割**:
- 静的解析（ruff, mypy）を実行
- 変異テスト（mutmut）でテスト品質を検証
- 結果をGitHub Issueにコメント

**使用ツール**: Read, Bash

**判定基準**:
| チェック | 基準 |
|----------|------|
| ruff | エラー0件 |
| mypy | エラー0件 |
| mutmut | スコア80%以上 |

---

### 7. Validator

**定義ファイル**: [../agents/validator.md](../agents/validator.md)

**役割**:
- PRがmainにマージされた後、システム全体を検証
- pytest, ruff, mypy を実行
- 失敗時はバグIssueを作成

**使用ツール**: Read, Bash

**出力**: `PASSED` または `FAILED`（失敗時はバグIssue作成）

---

### 8. Scrum Master

**定義ファイル**: [../agents/scrum_master.md](../agents/scrum_master.md)

**役割**:
- 作業メモから汎用的な学びを抽出
- 適切なルールファイルに追記

**使用ツール**: Read, Write, Edit

**出力先の分類**:
| 学びの内容 | 出力先 |
|------------|--------|
| Python文法・スタイル | `../rules/python.md` |
| テスト（pytest、モック） | `../rules/testing.md` |
| 外部ライブラリの使い方 | `../rules/libraries.md` |
| 設計・アーキテクチャ | `../rules/architecture.md` |
| プロジェクト固有の方針 | `CLAUDE.md`（ルート） |

---

## 次のステップ

- [COMMANDS.md](COMMANDS.md) - 各コマンドの詳細リファレンス
- [WORKFLOW.md](WORKFLOW.md) - エージェント間の協調動作

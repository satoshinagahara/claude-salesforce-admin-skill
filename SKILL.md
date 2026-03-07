---
name: salesforce-admin
description: "Claude Code専用のSalesforce管理スキル。データの登録・更新（単件/バルク）、カスタムオブジェクト・フィールド・レイアウト・権限セット・Apex・フロー・Agentforce・Data360（Data Cloud）などの操作をCLIで実行。本番環境対応。"
---

# salesforce-admin Skill

## Description
Claude Code専用のSalesforce管理スキル。データの登録・更新（単件/バルク）、カスタムオブジェクト・フィールド・レイアウト・権限セット・Apex・フロー・Agentforce・Data360（Data Cloud）などの操作をCLIで実行。本番環境対応。

## Overview
このスキルはSalesforce CLIを使って以下の操作を実行する：

- **データ操作（DML）**: レコードの登録・更新・削除（単件／バルク）
- **メタデータ操作**: カスタムオブジェクト・フィールド・ページレイアウト・レコードタイプ・権限セット・プロファイル・Apex・フロー・自動化のデプロイ・変更
- **フロー作成**: レコードトリガーフロー・スケジュールフロー・自動起動フローをflow-meta.xml として新規作成しデプロイ可能。条件分岐・ループ・Apexアクション呼び出し・レコード操作に対応。Screen Flow（画面あり）は複雑なため基本的にUIで作成を推奨。
- **Data360（Data Cloud）**: データストリーム・データモデル（DMO/DLO）・データマッピング・Identity Resolution（統合ルール）の設定・確認。Connect REST API (`sf api request rest`) を使用。
- **Agentforce × Data Cloud連携**: Data CloudのデータをAgentforceエージェントのグラウンディングデータとして活用。Data Graph・Semantic Search・RAG の設定。

## Prerequisites
- `sf` コマンド（Salesforce CLI）がインストール済みであること
- JWT認証設定ファイル（`sf-config.json`）が存在すること

## Execution Flow

### Step 1: 設定ファイルの読み込みと認証

以下の優先順でsf-config.jsonを探す：
1. `~/sf-config.json`（ホームディレクトリ直下）
2. カレントディレクトリの `sf-config.json`

```bash
# 設定ファイルを読み込んでJWT認証を実行
sf org login jwt \
  --client-id "<consumer_key>" \
  --jwt-key-file "<key_file_path>" \
  --username "<username>" \
  --instance-url "<instance_url>"
```

認証成功後、`--target-org <username>` または `-o <username>` を全コマンドに付与する。

### Step 2: 安全確認（本番環境では必須）

**本番環境での操作は以下を必ず実行すること：**

1. `references/safety-production.md` を読み込む
2. 操作内容・影響範囲・ロールバック可否をユーザーに提示
3. ユーザーの明示的な承認を得てから実行
4. 破壊的操作（削除・権限変更）は特に慎重に確認

### Step 3: 操作タイプの判定と参照ファイルの読み込み

ユーザーの指示から操作タイプを判定し、対応するreferenceファイルを読み込む：

| 操作タイプ | referenceファイル | 判定キーワード |
|---|---|---|
| データ登録・更新・削除 | `references/data-dml.md` | 登録、作成、更新、削除、インポート、CSV、レコード |
| カスタムオブジェクト・フィールド | `references/metadata-objects.md` | カスタムオブジェクト、項目追加、フィールド、必須、参照関係 |
| ページレイアウト・レコードタイプ | `references/metadata-ui.md` | ページレイアウト、レコードタイプ、レイアウト変更 |
| 権限セット・プロファイル | `references/metadata-security.md` | 権限セット、プロファイル、アクセス権、ユーザー権限 |
| Apex・フロー・自動化 | `references/metadata-automation.md` | Apex、フロー、トリガー、ワークフロー、プロセスビルダー |
| Lightning Web Component | `references/metadata-lwc.md` | LWC、Lightning Web Component、コンポーネント作成、lwc、UI部品 |
| Agentforce / GenAI Agent | `references/metadata-agentforce.md` | Agentforce、Agent、エージェント、GenAiPlugin、GenAiFunction、GenAiPlanner、Bot、BotVersion、AIエージェント、agentSpec |
| Data360 データストリーム・取り込み | `references/data-streams.md` | データストリーム、Ingestion、取り込み、コネクタ、データソース、Data Cloud、Data360、CDP |
| Data360 データモデル・マッピング・統合 | `references/data-model.md` | データモデル、DMO、DLO、Data Lake Object、マッピング、統合ルール、Identity Resolution、名寄せ |
| Agentforce × Data Cloud連携・RAG | `references/agentforce-integration.md` | グラウンディング、Data Graph、Semantic Search、ベクトル検索、RAG、Retriever、Search Index、Data Library、チャンキング、Prompt Template、ナレッジ検索 |

複数タイプにまたがる場合は複数ファイルを読み込む。

### Step 4: 操作の実行

referenceファイルのパターンに従ってsf CLIコマンドを生成・実行する。

**メタデータ操作の基本フロー：**
```
retrieve（現状取得）→ 変更（XMLファイル編集）→ validate（検証）→ deploy（デプロイ）
```

**必ず `--dry-run` または `validate` を先に実行し、ユーザーに結果を提示してからdeployする。**

### Step 5: 結果の報告

- 成功・失敗・警告を明確に区別して報告
- 失敗時はエラーメッセージと考えられる原因を提示
- デプロイ結果はJob IDを記録して追跡可能にする

## Error Handling

| エラー | 原因と対処 |
|---|---|
| `INVALID_LOGIN` / `JWT` エラー | instance_urlがMy Domain URLでない。`sf-config.json`のinstance_urlを確認 |
| `INSUFFICIENT_ACCESS` | 権限不足。実行ユーザーの権限セット・プロファイルを確認 |
| `DUPLICATE_VALUE` | 重複レコード。External IDや既存レコードを確認 |
| `FIELD_CUSTOM_VALIDATION_EXCEPTION` | 入力規則違反。フィールド値を確認 |
| `Deploy error` | メタデータ不整合。validateの結果を確認 |
| `Cannot delete field` | 削除不可フィールド（参照関係あり等）。依存関係を先に解消 |
| `DATA_CLOUD_NOT_ENABLED` | Data Cloudが未有効化。Setup → Data Cloud で有効化 |
| `SSOT_*` エラー | Data Cloud固有エラー。エラーメッセージに従って対処 |

## Output Format

- **データ操作**: 成功件数・失敗件数・エラー詳細をテキストで報告
- **メタデータ操作**: デプロイ結果（Component追加/変更/エラー一覧）をテキストで報告
- ユーザーが指定した場合はCSV/JSON形式でも出力可

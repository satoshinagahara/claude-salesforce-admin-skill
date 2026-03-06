# Claude Code — Salesforce Admin Skill

Claude Code 用の Salesforce 管理スキルです。Salesforce CLI (`sf`) を使って、データ操作・メタデータ操作・LWC作成などを自然言語で指示するだけで実行できます。

## 主な機能

| 操作カテゴリ | できること |
|---|---|
| **データ操作 (DML)** | レコードの登録・更新・削除（単件／バルクCSV）|
| **カスタムオブジェクト・フィールド** | 作成・変更・削除・リレーション設定 |
| **ページレイアウト・レコードタイプ** | レイアウト変更・セクション追加・フィールド配置 |
| **権限セット・プロファイル** | FLS設定・オブジェクト権限・ユーザー割り当て |
| **Apex・フロー・自動化** | クラス作成・トリガー・フロー有効化/無効化 |
| **Lightning Web Component** | コンポーネント作成・デプロイ |

## 前提条件

- [Salesforce CLI](https://developer.salesforce.com/tools/salesforcecli) (`sf`) がインストール済みであること
- JWT認証用の設定ファイル (`sf-config.json`) が準備済みであること

### sf-config.json の形式

```json
{
  "consumer_key": "YOUR_CONNECTED_APP_CONSUMER_KEY",
  "key_file_path": "/path/to/server.key",
  "username": "your-user@example.com",
  "instance_url": "https://your-domain.my.salesforce.com"
}
```

設定ファイルは以下のいずれかに配置してください（上から優先）：
1. `~/Desktop/WORK/CLAUDE/COWORK/SALESFORCE/sf-config.json`
2. `~/sf-config.json`
3. カレントディレクトリの `sf-config.json`

## インストール方法

Claude Code のスキルディレクトリにこのリポジトリをクローンします。

```bash
git clone https://github.com/satoshinagahara/claude-salesforce-admin-skill \
  ~/.claude/skills/salesforce-admin
```

## 使い方

Claude Code 上で `/salesforce-admin` とタイプするか、以下のように自然言語で指示します：

```
Account オブジェクトに「担当部署」というテキスト項目を追加してください

取引先レコードをCSVから100件インポートしてください

商談ページに新しいセクションを追加して、カスタム項目3つを配置してください

BOMという名前のカスタムオブジェクトを作成してください
```

## ファイル構成

```
salesforce-admin/
├── SKILL.md                       # スキル定義（Claude Code が読み込む）
├── README.md                      # このファイル
└── references/
    ├── data-dml.md                # データ操作リファレンス
    ├── metadata-objects.md        # オブジェクト・フィールド操作
    ├── metadata-ui.md             # ページレイアウト・レコードタイプ
    ├── metadata-security.md       # 権限セット・プロファイル
    ├── metadata-automation.md     # Apex・フロー・自動化
    ├── metadata-lwc.md            # Lightning Web Component
    └── safety-production.md       # 本番環境安全確認プロトコル
```

## 本番環境での安全機能

本番 org への操作時は `safety-production.md` のプロトコルが自動的に適用されます：

- メタデータ変更前に **Validate** を実行してユーザーに提示
- 破壊的操作（削除・バルク更新）前に **バックアップ** を取得
- 操作前に **影響範囲をユーザーに提示**、明示的な承認後に実行

## 既知の制約・注意事項

- **Product2 は Master-Detail の親になれない** → Lookup + SetNull で代替
- **`--metadata` フラグは source-backed コンポーネントに使用不可** → `--source-dir` を使う
- **カスタムフィールドのデプロイ後は FLS 設定が必要** → 権限セットで付与
- **Master-Detail 項目は fieldPermissions に含められない** → 除外すること

## ライセンス

MIT

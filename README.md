# Claude Code — Salesforce Admin Skill

Claude Code 用の Salesforce 管理スキルです。Salesforce CLI (`sf`) を使って、データ操作・メタデータ操作・LWC作成などを自然言語で指示するだけで実行できます。

---

## ⚠️ 利用環境について（重要）

> **このスキルは Developer Edition または Sandbox 環境での利用を推奨します。**

本スキルは Salesforce CLI を通じて org に直接操作を行います。**本番環境（Production）での使用は十分な理解と慎重な運用が必要です。**

| 環境 | 推奨度 | 備考 |
|---|---|---|
| **Developer Edition** | ✅ 推奨 | 無料で取得可能。本番データへの影響なし |
| **Sandbox** | ✅ 推奨 | 本番の設定を引き継いだテスト環境 |
| **本番 (Production)** | ⚠️ 上級者向け | 誤操作がビジネスに直結。自動安全確認が適用されるが要注意 |

Developer Edition org は [Salesforce 開発者登録](https://developer.salesforce.com/signup) から無料で取得できます。

---

## 主な機能

| 操作カテゴリ | できること |
|---|---|
| **データ操作 (DML)** | レコードの登録・更新・削除（単件／バルクCSV）|
| **カスタムオブジェクト・フィールド** | 作成・変更・削除・リレーション設定 |
| **ページレイアウト・レコードタイプ** | レイアウト変更・セクション追加・フィールド配置 |
| **権限セット・プロファイル** | FLS設定・オブジェクト権限・ユーザー割り当て |
| **Apex・フロー・自動化** | クラス作成・トリガー・フロー有効化/無効化 |
| **Lightning Web Component** | コンポーネント作成・デプロイ |
| **Agentforce** | AIエージェントの作成・設定、トピック・アクション・プロンプトテンプレート |
| **Data360（Data Cloud）** | データストリーム・データモデル（DMO/DLO）・マッピング・統合ルール |
| **Agentforce × Data Cloud連携** | グラウンディングデータ・Data Graph・Semantic Search・RAG |

---

## 前提条件

1. [Salesforce CLI](https://developer.salesforce.com/tools/salesforcecli) (`sf`) がインストール済みであること
2. Salesforce org に **接続アプリケーション（Connected App）** が作成済みであること
3. JWT認証用の設定ファイル (`sf-config.json`) が準備済みであること

---

## Salesforce 側のセットアップ

このスキルは **JWT Bearer Flow** を使って Salesforce に接続します。以下の手順で一度だけセットアップが必要です。

### Step 1: RSA 鍵ペアの生成

ターミナルで以下を実行します：

```bash
# 作業ディレクトリを作成
mkdir -p ~/sf-jwt-keys && cd ~/sf-jwt-keys

# 秘密鍵と自己署名証明書を生成（有効期間365日）
openssl req -x509 -newkey rsa:2048 \
  -keyout server.key \
  -out server.crt \
  -days 365 -nodes \
  -subj "/CN=salesforce-jwt"
```

生成されるファイル：
- `server.key` — 秘密鍵（**絶対に公開しないこと**）
- `server.crt` — 証明書（Salesforceにアップロードする）

### Step 2: Salesforce に接続アプリケーションを作成

1. Salesforce の **設定（Setup）** を開く
2. 検索バーで「**アプリケーションマネージャ**」を検索
3. 「**新規接続アプリケーション**」をクリック
4. 以下を入力：

| 項目 | 値 |
|---|---|
| 接続アプリケーション名 | `Claude Code JWT` （任意）|
| API 参照名 | `Claude_Code_JWT` |
| 取引先責任者メール | 自分のメールアドレス |
| OAuth 設定の有効化 | ✅ チェック |
| コールバック URL | `http://localhost:1717/OauthRedirect` |
| OAuth 範囲 | `完全なアクセス (full)` を追加 |
| デジタル署名を使用 | ✅ チェック → `server.crt` をアップロード |

5. 保存後、「**コンシューマ鍵（Consumer Key）**」をメモする

### Step 3: 接続アプリケーションのポリシー設定

1. 作成したアプリを開き「**ポリシーを管理**」をクリック
2. **OAuth ポリシー** → 「許可されているユーザー」を `管理者が承認したユーザーは事前承認済み` に変更
3. **プロファイル** タブで、接続を許可するプロファイル（例: `システム管理者`）を追加

> ⚠️ このポリシー設定をしないと JWT 認証で `INVALID_LOGIN` エラーが発生します。

### Step 4: sf-config.json を作成

```json
{
  "consumer_key": "ここにコンシューマ鍵を貼り付け",
  "key_file_path": "/Users/yourname/sf-jwt-keys/server.key",
  "username": "your-username@example.com",
  "instance_url": "https://your-domain.my.salesforce.com"
}
```

> **instance_url** は My Domain URL（`https://xxx.my.salesforce.com`）を使うこと。`login.salesforce.com` では JWT 認証に失敗します。

設定ファイルは以下のいずれかに配置してください（上から優先）：
1. `~/sf-config.json`（ホームディレクトリ直下）
2. カレントディレクトリの `sf-config.json`

### Step 5: 接続テスト

```bash
sf org login jwt \
  --client-id "YOUR_CONSUMER_KEY" \
  --jwt-key-file ~/sf-jwt-keys/server.key \
  --username your-username@example.com \
  --instance-url https://your-domain.my.salesforce.com

# 成功すると org 情報が表示される
sf org display --target-org your-username@example.com
```

---

## インストール方法

Claude Code のスキルディレクトリにこのリポジトリをクローンします。

```bash
git clone https://github.com/satoshinagahara/claude-salesforce-admin-skill \
  ~/.claude/skills/salesforce-admin
```

---

## 使い方

Claude Code 上で `/salesforce-admin` とタイプするか、以下のように自然言語で指示します：

```
Account オブジェクトに「担当部署」というテキスト項目を追加してください

取引先レコードをCSVから100件インポートしてください

商談ページに新しいセクションを追加して、カスタム項目3つを配置してください

BOMという名前のカスタムオブジェクトを作成してください

Agentforceの取引先サマリー用Employee Agentを作成してください

AccountオブジェクトのData Cloudデータストリームを設定してください
```

---

## ファイル構成

```
salesforce-admin/
├── SKILL.md                       # スキル定義（Claude Code が読み込む）
├── README.md                      # このファイル（日本語 README）
├── README_en.md                   # English README
└── references/
    ├── data-dml.md                # データ操作リファレンス
    ├── metadata-objects.md        # オブジェクト・フィールド操作
    ├── metadata-ui.md             # ページレイアウト・レコードタイプ
    ├── metadata-security.md       # 権限セット・プロファイル
    ├── metadata-automation.md     # Apex・フロー・自動化
    ├── metadata-lwc.md            # Lightning Web Component
    ├── metadata-agentforce.md     # Agentforce（GenAI Agent）操作
    ├── data-streams.md            # Data360: データストリーム・Ingestion API
    ├── data-model.md              # Data360: DMO/DLO・マッピング・統合ルール
    ├── agentforce-integration.md  # Agentforce × Data Cloud 連携
    └── safety-production.md       # 本番環境安全確認プロトコル
```

---

## 本番環境での安全機能

本番 org への操作時は `safety-production.md` のプロトコルが自動的に適用されます：

- メタデータ変更前に **Validate** を実行してユーザーに提示
- 破壊的操作（削除・バルク更新）前に **バックアップ** を取得
- 操作前に **影響範囲をユーザーに提示**、明示的な承認後に実行

---

## 既知の制約・注意事項

- **`--metadata` フラグは source-backed コンポーネントに使用不可** → `--source-dir` を使う
- **カスタムフィールドのデプロイ後は FLS 設定が必要** → 権限セットで付与
- **Master-Detail 項目は fieldPermissions に含められない** → 除外すること

---

## 免責事項

本スキルは Salesforce CLI を通じて Salesforce org に直接操作を行います。ご利用にあたっては以下の点をご了承ください。

- 本スキルの使用によって生じたデータの損失・破損・意図しない変更、およびその他いかなる損害についても、作者は一切の責任を負いません。
- AI（Claude）による自動操作を含むため、意図しない操作が実行される可能性があります。特に本番環境（Production）での使用は十分に理解したうえで自己責任にて行ってください。
- 本スキルは現状有姿（as-is）で提供されており、動作の正確性・完全性・特定目的への適合性について、いかなる保証も行いません。
- Salesforce の仕様変更・API の変更等により、予告なく動作しなくなる場合があります。

**Use at your own risk.**

---

## ライセンス

MIT

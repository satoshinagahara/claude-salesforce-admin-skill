# Data Streams & Ingestion Reference

## 概要
Data360（Data Cloud）へのデータ取り込み方法。データストリームはData Cloudにデータを流し込むパイプライン。

---

## 1. データストリームの種類

| 種別 | 説明 | 主な用途 |
|---|---|---|
| **CRM Connector** | SalesforceオブジェクトからData Cloudへ自動同期 | Account, Contact, Lead等の標準/カスタムオブジェクト |
| **Ingestion API** | 外部システムからREST APIでデータを投入 | 外部DB、SaaS、IoTデータ |
| **MuleSoft Connector** | MuleSoftを経由した外部連携 | エンタープライズ統合 |
| **Marketing Cloud Connector** | Marketing Cloudからのデータ連携 | メールエンゲージメント等 |
| **Commerce Cloud Connector** | Commerce Cloudからのデータ連携 | ECデータ |
| **Google Cloud Storage / S3 / Azure** | クラウドストレージからのバッチ取り込み | 大量の外部データ |

---

## 2. CRM Connector データストリームの確認・設定

### 2-1. 現在のデータストリーム一覧を取得

```bash
# データストリーム一覧
sf api request rest \
  --method GET \
  --url "/services/data/v62.0/ssot/data-streams" \
  --target-org <username>
```

レスポンス例：
```json
{
  "data": [
    {
      "name": "Account",
      "category": "CrmConnector",
      "objectName": "Account",
      "status": "Active",
      "refreshFrequency": "Every_6_Hours"
    }
  ]
}
```

### 2-2. データストリームの詳細

```bash
sf api request rest \
  --method GET \
  --url "/services/data/v62.0/ssot/data-streams/<streamName>" \
  --target-org <username>
```

### 2-3. CRM Connector の設定（UIベース）

CRM Connectorデータストリームの新規作成はUIから行う：
1. Setup → Data Cloud → Data Streams → New
2. "Salesforce CRM" を選択
3. 対象オブジェクトとフィールドを選択
4. 更新頻度を設定（Every 6 Hours / Every Hour 等）
5. Save & Deploy

**CLIでの操作**: 作成後の状態確認・メタデータ取得は可能。

---

## 3. Ingestion API によるデータ取り込み

### 3-1. Ingestion API コネクタの作成

まずData Cloud Setup UIで Ingestion API コネクタを作成する：
1. Setup → Data Cloud → Data Streams → New → Ingestion API
2. コネクタ名と説明を入力
3. スキーマ（YAML/OpenAPI形式）をアップロードまたは定義
4. Save

### 3-2. Ingestion API のスキーマ定義（OpenAPI形式）

```yaml
openapi: 3.0.3
info:
  title: External Data Schema
  version: "1.0"
components:
  schemas:
    ExternalEvent:
      type: object
      required:
        - id
        - timestamp
      properties:
        id:
          type: string
        timestamp:
          type: string
          format: date-time
        category:
          type: string
        value:
          type: number
```

### 3-3. Ingestion API でデータ投入（Streaming Insert）

```bash
# Step 1: Ingestion API トークン取得（通常のOAuth2トークンを使用）
# JWT認証済みの場合、sf api request rest でそのまま使える

# Step 2: ストリーミングインサート
sf api request rest \
  --method POST \
  --url "/api/v1/ingest/sources/<connectorName>/<objectName>" \
  --body '{
    "data": [
      {
        "id": "ext-001",
        "timestamp": "2026-03-07T00:00:00Z",
        "category": "web_visit",
        "value": 100
      }
    ]
  }' \
  --target-org <username>
```

**注意**: Ingestion API のベースパスは `/api/v1/ingest/` であり、Connect API の `/services/data/` とは異なる。

### 3-4. Ingestion API でバルクデータ投入（CSV）

```bash
# Step 1: バルクジョブ開始
sf api request rest \
  --method POST \
  --url "/api/v1/ingest/jobs" \
  --body '{
    "object": "<objectName>",
    "sourceName": "<connectorName>",
    "operation": "upsert"
  }' \
  --target-org <username>

# Step 2: CSVデータアップロード
# レスポンスの jobId を使用
sf api request rest \
  --method PUT \
  --url "/api/v1/ingest/jobs/<jobId>/batches" \
  --body @data.csv \
  --header "Content-Type: text/csv" \
  --target-org <username>

# Step 3: ジョブ完了を通知
sf api request rest \
  --method PATCH \
  --url "/api/v1/ingest/jobs/<jobId>" \
  --body '{"state": "UploadComplete"}' \
  --target-org <username>

# Step 4: ジョブ状態確認
sf api request rest \
  --method GET \
  --url "/api/v1/ingest/jobs/<jobId>" \
  --target-org <username>
```

---

## 4. データストリームの状態管理

### 4-1. 同期ステータスの確認

```bash
# データストリームのジョブ履歴
sf api request rest \
  --method GET \
  --url "/services/data/v62.0/ssot/data-streams/<streamName>/jobs" \
  --target-org <username>
```

### 4-2. リフレッシュ頻度

| 値 | 説明 |
|---|---|
| `Every_Hour` | 毎時同期 |
| `Every_6_Hours` | 6時間毎（デフォルト） |
| `Every_12_Hours` | 12時間毎 |
| `Every_24_Hours` | 日次 |

---

## 5. メタデータによるデータストリーム管理

### 5-1. package.xml でデータストリーム取得

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>*</members>
        <name>ExternalDataSource</name>
    </types>
    <types>
        <members>*</members>
        <name>ExternalDataTranObject</name>
    </types>
    <version>62.0</version>
</Package>
```

```bash
sf project retrieve start \
  --manifest package.xml \
  --target-org <username>
```

---

## 6. Amazon S3 Cloud Storage Connector

### 6-1. セットアップ手順

1. Data Cloud Setup → Other Connectors → New → Amazon S3
2. Access Key ID / Secret Access Key / Bucket Name / Root Folder を設定
3. Test Connection → Save
4. データストリームを作成（ファイル指定）

### 6-2. IAMポリシー（最小権限）

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Resource": [
                "arn:aws:s3:::<bucket-name>",
                "arn:aws:s3:::<bucket-name>/*"
            ]
        }
    ]
}
```

### 6-3. ワイルドカードファイル取り込み

- ファイル名に `*.csv` を指定すると、フォルダ内の全CSVを取り込める
- 全ファイルのスキーマ（カラム名・型）が統一されている必要がある
- 新しいファイルがバケットに追加されると、次回のスケジュール実行時に自動取り込み

### 6-4. 差分管理

- Incrementalモードではファイルの最終更新日時で差分を検出
- Primary Key指定でUpsert動作（同一キーは更新、新規は挿入）
- ファイル上書きアップロードでファイル全体が再取り込み対象になる

---

## 7. DLOカテゴリの重要性（設計上の最重要判断）

DLO作成時のカテゴリ選択がDMOマッピング先とID解決参加可否を決定する。**作成後の変更不可。**

| カテゴリ | マッピング可能なDMO | Identity Resolution | 用途 |
|---|---|---|---|
| **プロファイル** | Individual, Contact Point Email/Phone/Address | **参加可能** | 人物・顧客マスター情報 |
| **エンゲージメント** | イベント系DMO、カスタムエンゲージメントDMO | **参加不可** | 行動・インタラクションデータ |
| **その他** | Account, Product等の参照データ系 | 参加不可 | マスターデータ、参照テーブル |

### 外部データにメール等の識別情報があり名寄せしたい場合

**DLOを2つに分割する（正攻法）:**
1. プロファイルDLO（プロファイルカテゴリ）: メール+名前 → Individual + Contact Point Email
2. エンゲージメントDLO（エンゲージメントカテゴリ）: 回答内容等 → カスタムDMO → Contact Point Emailへリレーション

**代替: Data Graphのリレーション結合で代替**
- カスタムDMOのメール項目からContact Point Emailへリレーションを定義
- Data Graphで結合（統合プロファイルは生成されないが、データ結合は可能）

---

## 8. データストリーム再作成時のAPI名変更

DLOのデータストリームを削除→再作成すると、カラムのAPI名（`default_0__c` 〜 `default_N__c`）が**振り直される**。

- CSVに列を追加・削除した場合も同様に変更される
- queryv2やApexでAPI名をハードコードしている場合、**全箇所の修正が必要**
- 対策: データストリーム再作成後は `SELECT * FROM xxx__dlm LIMIT 1` でメタデータを確認し、API名のマッピングを更新する

---

## 9. よくあるエラーと対処

| エラー | 原因 | 対処 |
|---|---|---|
| `Schema validation failed` | Ingestion APIスキーマが不正 | OpenAPI 3.0形式を確認、required/typeの整合性チェック |
| `Data stream not found` | ストリーム名が間違い | `GET /ssot/data-streams` で正確な名前を確認 |
| `Connector not active` | コネクタが無効状態 | UIでコネクタをActivateする |
| `Rate limit exceeded` | Ingestion APIのレート制限 | バッチサイズを調整、間隔を空ける |
| `Field mapping required` | フィールドマッピング未設定 | Data ModelでDLO→DMOのマッピングを設定 |
| `/ssot/` API が `NOT_FOUND` | Data Cloudは有効でもAPIユーザーに `CDPAdmin` 権限セットが未割り当て | `CDPAdmin` をPermissionSetAssignmentで割り当てる |

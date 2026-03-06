# Data DML Operations Reference

## 概要
Salesforce CLIを使ったレコードのCRUD操作パターン。

---

## 1. 単件操作

### 1-1. レコード新規作成（Create）

```bash
# 基本形式
sf data create record \
  --sobject <ObjectAPIName> \
  --values "<Field1>=<Value1> <Field2>=<Value2>" \
  --target-org <username>

# 例：取引先を1件作成
sf data create record \
  --sobject Account \
  --values "Name='株式会社サンプル' Phone='03-0000-0000' BillingCity='東京都'" \
  --target-org <username>

# 例：商談を1件作成
sf data create record \
  --sobject Opportunity \
  --values "Name='サンプル商談' StageName='Prospecting' CloseDate=2026-06-30 AccountId=0015j000000XXXXX" \
  --target-org <username>
```

**出力**: 作成されたレコードのID

---

### 1-2. レコード更新（Update）

```bash
# 基本形式（RecordIdを必ず指定）
sf data update record \
  --sobject <ObjectAPIName> \
  --record-id <RecordId> \
  --values "<Field1>=<NewValue1>" \
  --target-org <username>

# 例：商談フェーズを更新
sf data update record \
  --sobject Opportunity \
  --record-id 0065j000000XXXXX \
  --values "StageName='Negotiation' Amount=5000000" \
  --target-org <username>
```

**注意**: RecordIdは事前にSOQLで確認すること。

---

### 1-3. レコード削除（Delete）

```bash
# 基本形式
sf data delete record \
  --sobject <ObjectAPIName> \
  --record-id <RecordId> \
  --target-org <username>

# 例：リードを1件削除
sf data delete record \
  --sobject Lead \
  --record-id 00Q5j000000XXXXX \
  --target-org <username>
```

**⚠️ 本番環境での削除は safety-production.md の確認手順を必ず実施すること。**

---

### 1-4. レコード取得（Get）

```bash
# 単件取得（RecordIdで指定）
sf data get record \
  --sobject <ObjectAPIName> \
  --record-id <RecordId> \
  --target-org <username>
```

---

## 2. バルク操作（CSV）

### 2-1. CSVフォーマット

```csv
# Accountのバルクインポート用CSV例
Name,Phone,BillingCity,BillingState
株式会社A,03-1111-1111,千代田区,東京都
株式会社B,06-2222-2222,北区,大阪府
```

**ルール**:
- 1行目はAPIフィールド名（日本語不可）
- 文字コードはUTF-8
- 日付フォーマット: `YYYY-MM-DD`
- Booleanは `true` / `false`

---

### 2-2. バルク挿入（Insert）

```bash
sf data import bulk \
  --sobject <ObjectAPIName> \
  --file <path/to/data.csv> \
  --target-org <username>

# 例
sf data import bulk \
  --sobject Account \
  --file ~/Desktop/accounts.csv \
  --target-org <username>
```

---

### 2-3. バルクUpsert（Insert/Update混在）

```bash
# External IDフィールドが必要
sf data upsert bulk \
  --sobject <ObjectAPIName> \
  --file <path/to/data.csv> \
  --external-id <ExternalIdFieldName> \
  --target-org <username>

# 例：External ID（ExternalId__c）でUpsert
sf data upsert bulk \
  --sobject Account \
  --file ~/Desktop/accounts.csv \
  --external-id ExternalId__c \
  --target-org <username>
```

**注意**: External IDフィールドは事前にSalesforceの「外部ID」チェックが必要。

---

### 2-4. バルク削除（Delete）

```bash
# CSVにIdフィールドのみ含める
# Id
# 001XXXXXXXXXXXXX
# 001YYYYYYYYYYYYY

sf data delete bulk \
  --sobject <ObjectAPIName> \
  --file <path/to/ids.csv> \
  --target-org <username>
```

**⚠️ バルク削除は取り消し不可。実行前にバックアップを取ること（safety-production.md参照）。**

---

### 2-5. バルク操作のジョブ状態確認

```bash
# バルク操作は非同期。Job IDで状態を確認
sf data bulk status \
  --job-id <JobId> \
  --target-org <username>

# 完了まで待機する場合
sf data bulk status \
  --job-id <JobId> \
  --target-org <username> \
  --wait 10
```

---

## 3. SOQL実行（確認・検索用）

```bash
# レコードIDや存在確認のためのSOQL実行
sf data query \
  --query "SELECT Id, Name FROM Account WHERE Name = '株式会社A'" \
  --target-org <username>

# JSON形式で取得
sf data query \
  --query "SELECT Id, Name, StageName FROM Opportunity WHERE IsClosed = false" \
  --json \
  --target-org <username>

# CSVで出力
sf data query \
  --query "SELECT Id, Name FROM Lead LIMIT 100" \
  --result-format csv \
  --target-org <username>
```

---

## 4. よくあるエラーと対処

| エラー | 原因 | 対処 |
|---|---|---|
| `REQUIRED_FIELD_MISSING` | 必須フィールドが未指定 | 対象オブジェクトの必須フィールドをSOQLで確認 |
| `DUPLICATE_VALUE` | 外部IDや一意制約の重複 | 既存レコードをSOQLで確認してUpsertに切替 |
| `INVALID_FIELD` | フィールドAPI名が誤り | スキーマ確認: `sf sobject describe --sobject <Object>` |
| `STRING_TOO_LONG` | フィールドの最大文字数超過 | 文字数を確認して値を調整 |
| `FIELD_CUSTOM_VALIDATION_EXCEPTION` | 入力規則エラー | Salesforce UIで該当オブジェクトの入力規則を確認 |
| `CANNOT_UPDATE_CONVERTED_LEAD` | 変換済みリードは更新不可 | 商談・取引先で操作 |
| `制限つき選択リスト項目の値が不適切` | 選択リスト（Picklist）に存在しない値を指定した | フィールドXML (`<valueSet>`) またはdescribeで有効値を事前確認する |

**選択リスト値の事前確認方法**:
```bash
# フィールドXMLで確認（ローカルに取得済みの場合）
cat force-app/main/default/objects/<Object__c>/fields/<Field__c>.field-meta.xml

# describeで確認
sf sobject describe --sobject <Object__c> --target-org <username> \
  | python3 -c "
import json,sys
d=json.load(sys.stdin)
for f in d['fields']:
    if f['name']=='<Field__c>' and f.get('picklistValues'):
        for v in f['picklistValues']:
            print(v['value'])
"
```

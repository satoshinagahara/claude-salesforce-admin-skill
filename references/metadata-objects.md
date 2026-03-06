# Metadata: Custom Objects & Fields Reference

## 概要
カスタムオブジェクトの作成・変更、カスタム項目の追加・変更をSalesforce CLIで実行するパターン。

**基本フロー**: retrieve（現状取得）→ XMLを編集 → validate → deploy

---

## 1. 作業ディレクトリの準備

```bash
# プロジェクトディレクトリを作成（作業用・一時的）
mkdir -p ~/sf-admin-work && cd ~/sf-admin-work
sf project generate --name sf-admin-project --template empty
cd sf-admin-project
```

---

## 2. カスタムオブジェクトの作成

### 2-1. 既存オブジェクトのRetrieve（参考用）

```bash
# 既存のカスタムオブジェクトを取得して参考にする
sf project retrieve start \
  --metadata "CustomObject:<ExistingObject__c>" \
  --target-org <username>
```

### 2-2. 新規カスタムオブジェクトのXML作成

ファイルパス: `force-app/main/default/objects/<ObjectName__c>/<ObjectName__c>.object-meta.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>オブジェクトラベル</label>
    <pluralLabel>オブジェクトラベル（複数）</pluralLabel>
    <nameField>
        <label>名前</label>
        <type>Text</type>
    </nameField>
    <deploymentStatus>Deployed</deploymentStatus>
    <sharingModel>ReadWrite</sharingModel>
    <description>オブジェクトの説明</description>
    <enableActivities>true</enableActivities>
    <enableBulkApi>true</enableBulkApi>
    <enableHistory>false</enableHistory>
    <enableReports>true</enableReports>
    <enableSearch>true</enableSearch>
    <enableSharing>true</enableSharing>
    <enableStreamingApi>true</enableStreamingApi>
</CustomObject>
```

**sharingModelの選択肢**:
- `Private`: 非公開（オーナーのみ）
- `Read`: 閲覧のみ共有
- `ReadWrite`: 読み書き共有
- `ControlledByParent`: 親オブジェクトに依存（Master-Detail時）

---

## 3. カスタム項目（フィールド）の追加

### 3-1. 既存フィールドのRetrieve

```bash
# オブジェクト全体（フィールド含む）を取得
sf project retrieve start \
  --metadata "CustomObject:<ObjectName__c>" \
  --target-org <username>
```

### 3-2. フィールドXMLの配置先

`force-app/main/default/objects/<ObjectName__c>/fields/<FieldName__c>.field-meta.xml`

### 3-3. フィールドタイプ別テンプレート

**テキスト（Text）**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>FieldName__c</fullName>
    <label>フィールドラベル</label>
    <type>Text</type>
    <length>255</length>
    <required>false</required>
    <unique>false</unique>
    <externalId>false</externalId>
</CustomField>
```

**数値（Number）**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>FieldName__c</fullName>
    <label>フィールドラベル</label>
    <type>Number</type>
    <precision>18</precision>
    <scale>0</scale>
    <required>false</required>
</CustomField>
```

**通貨（Currency）**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>FieldName__c</fullName>
    <label>フィールドラベル</label>
    <type>Currency</type>
    <precision>18</precision>
    <scale>2</scale>
    <required>false</required>
</CustomField>
```

**日付（Date）**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>FieldName__c</fullName>
    <label>フィールドラベル</label>
    <type>Date</type>
    <required>false</required>
</CustomField>
```

**チェックボックス（Checkbox）**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>FieldName__c</fullName>
    <label>フィールドラベル</label>
    <type>Checkbox</type>
    <defaultValue>false</defaultValue>
</CustomField>
```

**選択リスト（Picklist）**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>FieldName__c</fullName>
    <label>フィールドラベル</label>
    <type>Picklist</type>
    <required>false</required>
    <valueSet>
        <valueSetDefinition>
            <sorted>false</sorted>
            <value>
                <fullName>値1</fullName>
                <default>false</default>
                <label>値1</label>
            </value>
            <value>
                <fullName>値2</fullName>
                <default>false</default>
                <label>値2</label>
            </value>
        </valueSetDefinition>
    </valueSet>
</CustomField>
```

**参照関係（Lookup）**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>RelatedObject__c</fullName>
    <label>参照先ラベル</label>
    <type>Lookup</type>
    <referenceTo>Account</referenceTo>
    <relationshipLabel>関連レコード名（複数）</relationshipLabel>
    <relationshipName>RelationshipName</relationshipName>
    <required>false</required>
</CustomField>
```

**Master-Detail（親子関係）**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>ParentObject__c</fullName>
    <label>親オブジェクトラベル</label>
    <type>MasterDetail</type>
    <referenceTo>ParentObject__c</referenceTo>
    <relationshipLabel>子レコード名（複数）</relationshipLabel>
    <relationshipName>RelationshipName</relationshipName>
    <relationshipOrder>0</relationshipOrder>
</CustomField>
```

---

## 4. Validate（検証）とDeploy（デプロイ）

### 4-1. 必ずValidate先行

```bash
# 検証のみ（実際には変更しない）
sf project deploy validate \
  --source-dir force-app \
  --target-org <username>
```

結果を確認し、エラーがなければdeployを実行。

### 4-2. Deploy実行

```bash
sf project deploy start \
  --source-dir force-app \
  --target-org <username>

# 特定ディレクトリのみデプロイ（オブジェクト単位）
sf project deploy start \
  --source-dir force-app/main/default/objects/MyObject__c \
  --target-org <username>
```

**⚠️ `--metadata` フラグはsource-backedのコンポーネントには使えない。**
`--metadata "CustomField:..."` を指定すると `No source-backed components present in the package` エラーになる。
ローカルXMLファイルがある場合は必ず `--source-dir` を使うこと。

### 4-3. 特定フィールドのみデプロイ

```bash
# フィールドXMLを直接指定
sf project deploy start \
  --source-dir force-app/main/default/objects/MyObject__c/fields/MyField__c.field-meta.xml \
  --target-org <username>
```

### 4-4. デプロイ後のFLS設定（必須）

**カスタムオブジェクトとフィールドをデプロイしただけではSOQLやApexからアクセスできない場合がある。**
フィールドレベルセキュリティ（FLS）が未設定だと `No such column` エラーになる。

デプロイ後は必ず権限セットでFLSを付与すること（`metadata-security.md` 参照）:

```bash
# 権限セットでオブジェクト権限+フィールド権限を付与してデプロイ
# → 詳細は metadata-security.md の「FLS設定セット」パターンを参照
```

確認方法:
```bash
# Tooling APIでフィールドの存在確認（FLS問題の切り分けに使う）
sf data query \
  --query "SELECT Id, DeveloperName FROM CustomField WHERE TableEnumOrId = '<ObjectId>'" \
  --use-tooling-api \
  --target-org <username>

# ObjectIdは以下で取得
sf data query \
  --query "SELECT Id, DeveloperName FROM CustomObject WHERE DeveloperName = 'MyObject'" \
  --use-tooling-api \
  --target-org <username>
```

---

## 5. フィールド削除

**⚠️ フィールド削除は本番環境では不可逆。以下手順で実施：**

1. フィールドを参照している入力規則・トリガー・フローを先に無効化/削除
2. safety-production.md の削除確認チェックリストを実施
3. `destructiveChanges.xml` を使ってデプロイ

```xml
<!-- destructiveChanges.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>ObjectName__c.FieldName__c</members>
        <name>CustomField</name>
    </types>
    <version>63.0</version>
</Package>
```

```bash
sf project deploy start \
  --destructive-changes destructiveChanges.xml \
  --target-org <username>
```

---

## 6. よくあるエラーと対処

| エラー | 原因 | 対処 |
|---|---|---|
| `Duplicate field` | 同名フィールドが既存 | 既存フィールドのAPI名を確認してUpdate |
| `Cannot delete field referenced` | 参照されているフィールドは削除不可 | 参照元（入力規則・フロー等）を先に削除 |
| `Invalid fullName` | API名の命名規則違反 | 英数字・アンダースコアのみ、`__c`で終わる |
| `Master Detail not allowed` | Master-Detail制約違反 | 親オブジェクトの共有モデルを確認 |
| `No source-backed components present` | `--metadata`フラグでsource-backedコンポーネントを指定した | `--metadata`の代わりに`--source-dir`でXMLファイルのパスを指定 |
| SOQLで `No such column` (フィールドは存在するのに) | FLS（フィールドレベルセキュリティ）が未設定 | 権限セットでフィールド権限を付与してデプロイ→ユーザーに割り当て |
| `主従関係の子をXXXに追加できません` / `カスケードオプション...を使用して参照関係の子をXXXに追加できません` | Product2・User等の一部標準オブジェクトはMD関係の親になれない | MasterDetailをLookupに変更し、sharingModelをReadWrite、deleteConstraintをSetNullに変更 |

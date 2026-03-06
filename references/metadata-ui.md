# Metadata: Page Layouts & Record Types Reference

## 概要
ページレイアウトとレコードタイプのRetrieve・変更・Deployパターン。

---

## 1. ページレイアウト

### 1-1. 既存ページレイアウトのRetrieve

```bash
# オブジェクト全体をRetrieve（レイアウト含む）
sf project retrieve start \
  --metadata "Layout:<ObjectName>-<LayoutName>" \
  --target-org <username>

# 例：Opportunity標準レイアウト
sf project retrieve start \
  --metadata "Layout:Opportunity-Opportunity Layout" \
  --target-org <username>

# カスタムオブジェクトのレイアウト
sf project retrieve start \
  --metadata "Layout:MyObject__c-MyObject Layout" \
  --target-org <username>
```

**レイアウト名の確認方法：**
```bash
# 利用可能なレイアウト一覧
sf org list metadata \
  --metadata-type Layout \
  --target-org <username>
```

### 1-2. ファイルパス

`force-app/main/default/layouts/<ObjectName>-<LayoutName>.layout-meta.xml`

### 1-3. ページレイアウトXMLの主な構造

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Layout xmlns="http://soap.sforce.com/2006/04/metadata">
    <layoutSections>
        <customLabel>false</customLabel>
        <detailHeading>true</detailHeading>
        <editHeading>true</editHeading>
        <label>セクション名</label>
        <layoutColumns>
            <layoutItems>
                <behavior>Required</behavior>
                <field>FieldAPIName</field>
            </layoutItems>
            <layoutItems>
                <behavior>Edit</behavior>
                <field>AnotherField__c</field>
            </layoutItems>
        </layoutColumns>
        <layoutColumns/>
        <style>TwoColumnsTopToBottom</style>
    </layoutSections>
    <relatedLists>
        <fields>FULL_NAME</fields>
        <relatedList>RelatedObject__c</relatedList>
    </relatedLists>
</Layout>
```

**behaviorの選択肢**:
- `Required`: 必須
- `Edit`: 編集可能
- `Readonly`: 参照のみ

**styleの選択肢**:
- `TwoColumnsTopToBottom`: 2列（上→下）
- `TwoColumnsLeftToRight`: 2列（左→右）
- `OneColumn`: 1列

---

## 2. レコードタイプ

### 2-1. 既存レコードタイプのRetrieve

```bash
# レコードタイプを取得
sf project retrieve start \
  --metadata "RecordType:<ObjectName>.<RecordTypeName>" \
  --target-org <username>

# 例
sf project retrieve start \
  --metadata "RecordType:Opportunity.Enterprise_Deal" \
  --target-org <username>
```

### 2-2. 利用可能なレコードタイプ一覧

```bash
sf org list metadata \
  --metadata-type RecordType \
  --target-org <username>
```

### 2-3. 新規レコードタイプのXML

ファイルパス: `force-app/main/default/objects/<ObjectName__c>/recordTypes/<RecordTypeName>.recordType-meta.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<RecordType xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>RecordTypeName</fullName>
    <active>true</active>
    <description>レコードタイプの説明</description>
    <label>レコードタイプラベル</label>
    <picklistValues>
        <picklist>StageName</picklist>
        <values>
            <fullName>Prospecting</fullName>
            <default>true</default>
        </values>
        <values>
            <fullName>Qualification</fullName>
            <default>false</default>
        </values>
    </picklistValues>
</RecordType>
```

### 2-4. レコードタイプへのページレイアウト割り当て

プロファイルのXMLにレイアウト割り当てを記述する（metadata-security.md参照）。

または、`Profile`メタデータ内の`layoutAssignments`で管理：

```xml
<layoutAssignments>
    <layout>ObjectName-LayoutName</layout>
    <recordType>ObjectName.RecordTypeName</recordType>
</layoutAssignments>
```

---

## 3. ページレイアウトの変更例

### フィールド追加

既存レイアウトXMLの`layoutColumns`内に以下を追加：

```xml
<layoutItems>
    <behavior>Edit</behavior>
    <field>NewField__c</field>
</layoutItems>
```

### 新しいセクション追加

```xml
<layoutSections>
    <customLabel>true</customLabel>
    <detailHeading>true</detailHeading>
    <editHeading>true</editHeading>
    <label>新しいセクション名</label>
    <layoutColumns>
        <layoutItems>
            <behavior>Edit</behavior>
            <field>Field1__c</field>
        </layoutItems>
    </layoutColumns>
    <layoutColumns/>
    <style>TwoColumnsTopToBottom</style>
</layoutSections>
```

---

## 4. Validate → Deploy

```bash
# Validate（必須）
sf project deploy validate \
  --metadata "Layout:<ObjectName>-<LayoutName>" \
  --target-org <username>

# Deploy
sf project deploy start \
  --metadata "Layout:<ObjectName>-<LayoutName>" \
  --target-org <username>

# レコードタイプのDeploy
sf project deploy start \
  --metadata "RecordType:<ObjectName>.<RecordTypeName>" \
  --target-org <username>
```

---

## 5. よくあるエラーと対処

| エラー | 原因 | 対処 |
|---|---|---|
| `Unknown field` | フィールドAPI名が誤り | `sf sobject describe --sobject <Object>` で確認 |
| `Layout not found` | レイアウト名のスペルミス | `sf org list metadata --metadata-type Layout` で確認 |
| `RecordType already exists` | 同名レコードタイプが存在 | 既存を取得してUpdateで対応 |
| `Picklist value not found` | 選択リスト値がオブジェクトに未定義 | オブジェクトの選択リスト値を先に追加 |

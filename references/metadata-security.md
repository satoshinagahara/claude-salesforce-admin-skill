# Metadata: Permission Sets & Profiles Reference

## 概要
権限セット・プロファイルのRetrieve・変更・Deployパターン。

**⚠️ 権限変更は広範囲に影響する。本番操作前に safety-production.md を必ず参照すること。**

---

## 1. 権限セット（Permission Set）

### 1-1. 既存権限セットのRetrieve

```bash
# 特定権限セットを取得
sf project retrieve start \
  --metadata "PermissionSet:<PermSetName>" \
  --target-org <username>

# 利用可能な権限セット一覧
sf org list metadata \
  --metadata-type PermissionSet \
  --target-org <username>
```

### 1-2. 権限セットXMLの構造

ファイルパス: `force-app/main/default/permissionsets/<PermSetName>.permissionset-meta.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<PermissionSet xmlns="http://soap.sforce.com/2006/04/metadata">
    <description>権限セットの説明</description>
    <label>権限セットラベル</label>
    <hasActivationRequired>false</hasActivationRequired>

    <!-- オブジェクト権限 -->
    <objectPermissions>
        <allowCreate>true</allowCreate>
        <allowDelete>false</allowDelete>
        <allowEdit>true</allowEdit>
        <allowRead>true</allowRead>
        <modifyAllRecords>false</modifyAllRecords>
        <object>MyObject__c</object>
        <viewAllRecords>false</viewAllRecords>
    </objectPermissions>

    <!-- フィールド権限 -->
    <fieldPermissions>
        <editable>true</editable>
        <field>MyObject__c.MyField__c</field>
        <readable>true</readable>
    </fieldPermissions>

    <!-- Apex クラスアクセス -->
    <classAccesses>
        <apexClass>MyApexClass</apexClass>
        <enabled>true</enabled>
    </classAccesses>

    <!-- ページアクセス -->
    <pageAccesses>
        <apexPage>MyVisualforcePage</apexPage>
        <enabled>true</enabled>
    </pageAccesses>

    <!-- システム権限 -->
    <userPermissions>
        <enabled>true</enabled>
        <name>ApiEnabled</name>
    </userPermissions>
</PermissionSet>
```

### 1-3. 新規権限セットの作成

上記XMLテンプレートをベースに `force-app/main/default/permissionsets/<NewPermSetName>.permissionset-meta.xml` として保存し、Deployする。

### 1-4. ユーザーへの権限セット割り当て（DML）

```bash
# 権限セットをユーザーに割り当て
sf data create record \
  --sobject PermissionSetAssignment \
  --values "AssigneeId=<UserId> PermissionSetId=<PermSetId>" \
  --target-org <username>

# PermSetIdの確認
sf data query \
  --query "SELECT Id, Name, Label FROM PermissionSet WHERE Name = '<PermSetName>'" \
  --target-org <username>

# UserIdの確認
sf data query \
  --query "SELECT Id, Name, Username FROM User WHERE Username = '<username@example.com>'" \
  --target-org <username>
```

---

## 2. プロファイル（Profile）

**⚠️ プロファイルは組織全体の基盤設定。変更は特に慎重に実施すること。**

### 2-1. 既存プロファイルのRetrieve

```bash
# 特定プロファイルを取得
sf project retrieve start \
  --metadata "Profile:<ProfileName>" \
  --target-org <username>

# 例：標準ユーザープロファイル
sf project retrieve start \
  --metadata "Profile:Standard User" \
  --target-org <username>

# プロファイル一覧
sf org list metadata \
  --metadata-type Profile \
  --target-org <username>
```

### 2-2. プロファイルXMLの主要構造

ファイルパス: `force-app/main/default/profiles/<ProfileName>.profile-meta.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Profile xmlns="http://soap.sforce.com/2006/04/metadata">

    <!-- オブジェクト権限 -->
    <objectPermissions>
        <allowCreate>true</allowCreate>
        <allowDelete>false</allowDelete>
        <allowEdit>true</allowEdit>
        <allowRead>true</allowRead>
        <modifyAllRecords>false</modifyAllRecords>
        <object>MyObject__c</object>
        <viewAllRecords>false</viewAllRecords>
    </objectPermissions>

    <!-- フィールド権限 -->
    <fieldPermissions>
        <editable>true</editable>
        <field>MyObject__c.MyField__c</field>
        <readable>true</readable>
    </fieldPermissions>

    <!-- ページレイアウト割り当て -->
    <layoutAssignments>
        <layout>MyObject__c-MyObject Layout</layout>
    </layoutAssignments>
    <layoutAssignments>
        <layout>MyObject__c-SpecialLayout</layout>
        <recordType>MyObject__c.SpecialRecordType</recordType>
    </layoutAssignments>

    <!-- レコードタイプの可視性 -->
    <recordTypeVisibilities>
        <default>true</default>
        <recordType>MyObject__c.StandardRecordType</recordType>
        <visible>true</visible>
    </recordTypeVisibilities>

    <!-- アプリ表示設定 -->
    <applicationVisibilities>
        <application>MyApp</application>
        <default>false</default>
        <visible>true</visible>
    </applicationVisibilities>

    <!-- タブ表示設定 -->
    <tabVisibilities>
        <tab>MyObject__c</tab>
        <visibility>DefaultOn</visibility>
    </tabVisibilities>

    <!-- システム権限 -->
    <userPermissions>
        <enabled>true</enabled>
        <name>ApiEnabled</name>
    </userPermissions>
</Profile>
```

**tabVisibility の選択肢**:
- `DefaultOn`: デフォルト表示
- `DefaultOff`: デフォルト非表示
- `Hidden`: 非表示（固定）

### 2-3. プロファイルの変更方針

プロファイルの直接変更より**権限セットを追加する方法を推奨**。理由：
- プロファイル変更は全ユーザーに影響
- 権限セットは個別ユーザーに柔軟に割り当て可能
- ロールバックが容易

---

## 3. Validate → Deploy

```bash
# Validate（必須）
sf project deploy validate \
  --metadata "PermissionSet:<PermSetName>" \
  --target-org <username>

# Deploy
sf project deploy start \
  --metadata "PermissionSet:<PermSetName>" \
  --target-org <username>

# プロファイルのDeploy（特に慎重に）
sf project deploy validate \
  --metadata "Profile:<ProfileName>" \
  --target-org <username>

sf project deploy start \
  --metadata "Profile:<ProfileName>" \
  --target-org <username>
```

---

## 4. よくあるエラーと対処

| エラー | 原因 | 対処 |
|---|---|---|
| `Permission dependency not met` | 前提権限が不足 | 依存する権限を先に有効化 |
| `Unknown object` | オブジェクトAPI名が誤り | オブジェクト一覧で確認 |
| `Duplicate value` | 同名権限セットが既存 | 取得して変更で対応 |
| `Cannot modify System Administrator` | システム管理者プロファイルは保護 | 代替プロファイルか権限セットで対応 |
| `Field not accessible` | フィールドへのアクセス権なし | フィールドレベルセキュリティを先に設定 |
| `必須項目 XXX__c.YYY__c にはリリースできません` | Master-Detail関係項目（MD項目）をfieldPermissionsに含めている | MD関係項目はFLS設定が不要（常に親に依存）。`fieldPermissions`からMD項目を除外する |

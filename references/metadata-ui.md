# Metadata: Page Layouts, Record Types & FlexiPages Reference

## 概要
ページレイアウト、レコードタイプ、Lightning レコードページ（FlexiPage）のRetrieve・変更・Deployパターン。

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

---

## 6. Lightning レコードページ（FlexiPage）

### 6-1. 概要

FlexiPage = Lightning レコードページ。ページレイアウト（Classic向け項目配置）とは別に、Lightningコンポーネントの配置を定義する。

- **ページレイアウト**: 項目の配置・関連リストの有無（Classic/Lightning共通のデータ層）
- **FlexiPage**: Lightningコンポーネント（タブ、関連リスト、カスタムLWC等）の配置（Lightning専用のUI層）

レコードタイプ別に異なるUIを提供する場合は**両方**を設定する必要がある。

### 6-2. ファイルパスと基本構造

ファイルパス: `force-app/main/default/flexipages/<PageName>.flexipage-meta.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<FlexiPage xmlns="http://soap.sforce.com/2006/04/metadata">
    <flexiPageRegions>
        <!-- header, main, sidebar などのリージョン -->
    </flexiPageRegions>
    <masterLabel>My Record Page</masterLabel>
    <sobjectType>Account</sobjectType>
    <template>
        <name>flexipage:recordHomeTemplateDesktop</name>
    </template>
    <type>RecordPage</type>
</FlexiPage>
```

### 6-3. 主要テンプレート

| テンプレート | リージョン | 用途 |
|---|---|---|
| `flexipage:recordHomeTemplateDesktop` | header, main, sidebar | 標準的なレコードページ（2カラム） |
| `flexipage:recordHomeFullWidthTemplate` | header, main | フル幅（1カラム） |

### 6-4. リージョンとコンポーネントの構造

```xml
<!-- Regionの種類 -->
<flexiPageRegions>
    <name>header</name>       <!-- ヘッダー（ハイライトパネル） -->
    <type>Region</type>
</flexiPageRegions>
<flexiPageRegions>
    <name>main</name>         <!-- メインカラム -->
    <type>Region</type>
</flexiPageRegions>
<flexiPageRegions>
    <name>sidebar</name>      <!-- サイドバー -->
    <type>Region</type>
</flexiPageRegions>
<flexiPageRegions>
    <name>Facet-xxx</name>    <!-- タブ等のコンテンツ領域 -->
    <type>Facet</type>
</flexiPageRegions>
```

### 6-5. よく使うコンポーネント

```xml
<!-- ハイライトパネル（ヘッダー） -->
<componentName>force:highlightsPanel</componentName>

<!-- 詳細パネル（項目表示） -->
<componentName>force:detailPanel</componentName>

<!-- 全関連リスト（ページレイアウトに定義されたもの全て表示） -->
<componentName>force:relatedListContainer</componentName>

<!-- タブセット -->
<componentName>flexipage:tabset</componentName>
<!-- 個別タブ -->
<componentName>flexipage:tab</componentName>

<!-- 活動パネル -->
<componentName>runtime_sales_activities:activityPanel</componentName>

<!-- Chatter -->
<componentName>forceChatter:recordFeedContainer</componentName>
```

### 6-5a. 関連リストコンポーネントの選択（重要）

FlexiPageで関連リストを表示する方法は3つある。**カスタムLookupによる関連リストには `lst:dynamicRelatedList` を推奨。**

| コンポーネント | ページレイアウト依存 | 用途 |
|---|---|---|
| `force:relatedListContainer` | あり | ページレイアウトの全関連リストを一括表示 |
| `force:relatedListSingleContainer` | あり | ページレイアウトの特定関連リストを1つ表示 |
| **`lst:dynamicRelatedList`** | **なし** | **ページレイアウトに依存せず、直接リレーションを参照して表示** |

**⚠️ `force:relatedListSingleContainer` の問題点:**
- ページレイアウトに関連リストが含まれていないと表示されない
- カスタムLookupの関連リスト（例: `BOM_Part__c.Supplier__c`）はページレイアウトに入っていても表示されないケースがある
- 標準リレーション（`Contacts`, `Opportunities`等）では問題なく動作する

**推奨: `lst:dynamicRelatedList`（動的関連リスト）**

```xml
<componentInstance>
    <componentInstanceProperties>
        <name>actionNames</name>
        <valueList>
            <valueListItems>
                <value>New</value>    <!-- アクションボタン -->
            </valueListItems>
        </valueList>
    </componentInstanceProperties>
    <componentInstanceProperties>
        <name>adminFilters</name>     <!-- 空でOK（フィルタ条件なし） -->
    </componentInstanceProperties>
    <componentInstanceProperties>
        <name>maxRecordsToDisplay</name>
        <value>10</value>
    </componentInstanceProperties>
    <componentInstanceProperties>
        <name>parentFieldApiName</name>
        <value>Account.Id</value>     <!-- 親オブジェクト.Id -->
    </componentInstanceProperties>
    <componentInstanceProperties>
        <name>relatedListApiName</name>
        <value>BOM_Parts_Supplier__r</value>  <!-- 子リレーション名 -->
    </componentInstanceProperties>
    <componentInstanceProperties>
        <name>relatedListDisplayType</name>
        <value>ADVGRID</value>        <!-- ADVGRID=拡張グリッド, GRID=標準 -->
    </componentInstanceProperties>
    <componentInstanceProperties>
        <name>relatedListFieldAliases</name>
        <valueList>
            <valueListItems>
                <value>NAME</value>           <!-- 表示する列 -->
            </valueListItems>
            <valueListItems>
                <value>Part_Name__c</value>
            </valueListItems>
            <valueListItems>
                <value>Unit_Cost__c</value>
            </valueListItems>
        </valueList>
    </componentInstanceProperties>
    <componentInstanceProperties>
        <name>relatedListLabel</name>
        <value>BOM Parts (サプライヤー)</value>  <!-- 表示ラベル -->
    </componentInstanceProperties>
    <componentInstanceProperties>
        <name>showActionBar</name>
        <value>true</value>
    </componentInstanceProperties>
    <componentInstanceProperties>
        <name>sortFieldAlias</name>
        <value>__DEFAULT__</value>
    </componentInstanceProperties>
    <componentInstanceProperties>
        <name>sortFieldOrder</name>
        <value>Default</value>
    </componentInstanceProperties>
    <componentName>lst:dynamicRelatedList</componentName>
    <identifier>lst_dynamicRelatedList</identifier>
</componentInstance>
```

**`lst:dynamicRelatedList` のプロパティ（`force:relatedListSingleContainer` との違い）:**

| プロパティ | 説明 |
|---|---|
| `relatedListApiName` | 子リレーション名（例: `BOM_Parts_Supplier__r`, `Contacts`） |
| `relatedListDisplayType` | `ADVGRID`（拡張）or `GRID`（標準）※ `relatedListComponentOverride`ではない |
| `relatedListFieldAliases` | 表示列を `valueList` で明示指定（ページレイアウトに依存しない） |
| `relatedListLabel` | 関連リストの表示ラベル |
| `maxRecordsToDisplay` | 最大表示件数（`rowsToDisplay`ではない） |
| `actionNames` | 表示するアクションボタン（`valueList`形式） |
| `adminFilters` | 管理者フィルタ（空で全件） |
| `sortFieldAlias` / `sortFieldOrder` | ソート設定 |

### 6-6. タブ構造の組み立てパターン

FlexiPageのタブは3層構造で定義する:

```
1. Facet（タブのコンテンツ）  ← コンポーネントを配置
2. Facet（タブの定義）        ← flexipage:tab で各タブを定義、bodyでFacet1を参照
3. Region（tabsetの配置）     ← flexipage:tabset でFacet2を参照、main/sidebarに配置
```

```xml
<!-- Step 1: タブのコンテンツ用Facet -->
<flexiPageRegions>
    <itemInstances>
        <componentInstance>
            <componentName>force:detailPanel</componentName>
            <identifier>force_detailPanel</identifier>
        </componentInstance>
    </itemInstances>
    <name>Facet-detail</name>
    <type>Facet</type>
</flexiPageRegions>

<!-- Step 2: タブ定義用Facet -->
<flexiPageRegions>
    <itemInstances>
        <componentInstance>
            <componentInstanceProperties>
                <name>active</name>
                <value>true</value>  <!-- デフォルトタブ -->
            </componentInstanceProperties>
            <componentInstanceProperties>
                <name>body</name>
                <value>Facet-detail</value>  <!-- Step1のFacet名を参照 -->
            </componentInstanceProperties>
            <componentInstanceProperties>
                <name>title</name>
                <value>Standard.Tab.detail</value>  <!-- 標準ラベル or カスタム文字列 -->
            </componentInstanceProperties>
            <componentName>flexipage:tab</componentName>
            <identifier>detailTab</identifier>
        </componentInstance>
    </itemInstances>
    <name>Facet-tabs</name>
    <type>Facet</type>
</flexiPageRegions>

<!-- Step 3: Regionにtabsetを配置 -->
<flexiPageRegions>
    <itemInstances>
        <componentInstance>
            <componentInstanceProperties>
                <name>tabs</name>
                <value>Facet-tabs</value>  <!-- Step2のFacet名を参照 -->
            </componentInstanceProperties>
            <componentName>flexipage:tabset</componentName>
            <identifier>flexipage_tabset</identifier>
        </componentInstance>
    </itemInstances>
    <name>main</name>
    <type>Region</type>
</flexiPageRegions>
```

### 6-7. FlexiPageのデプロイ

```bash
sf project deploy start \
  --source-dir force-app/main/default/flexipages/<PageName>.flexipage-meta.xml \
  --target-org <username>
```

### 6-8. FlexiPageのレコードタイプ別割り当て（重要）

**⚠️ UIからの割り当てが最も確実。Metadata APIでのデプロイは反映されないケースがある。**

**推奨手順（UI）:**
1. 設定 → Lightning アプリケーションビルダー → 対象ページを開く
2. 右上「有効化 (Activation)」ボタンをクリック
3. 割り当て方法を選択:
   - **組織のデフォルト**: 全アプリ・全プロファイルに適用
   - **アプリケーションのデフォルト**: 特定のLightningアプリにのみ適用
   - **アプリケーション、レコードタイプ、プロファイル**: 最も細かい制御（アプリ×レコードタイプ×プロファイル）

**Metadata APIでの割り当て方法（参考）:**

FlexiPageのレコードタイプ別割り当ては `CustomObject.actionOverrides` では**不可**（`recordType`要素が存在しない）。
`CustomApplication.profileActionOverrides` を使用する:

```xml
<!-- CustomApplication（例: standard__LightningSales.app-meta.xml）内 -->
<profileActionOverrides>
    <actionName>View</actionName>
    <content>My_Record_Page</content>       <!-- FlexiPage名 -->
    <formFactor>Large</formFactor>           <!-- Large=デスクトップ, Small=モバイル -->
    <pageOrSobjectType>Account</pageOrSobjectType>
    <profile>システム管理者</profile>          <!-- プロファイル名（必須） -->
    <recordType>Account.Supplier</recordType> <!-- オブジェクト.レコードタイプ -->
    <type>Flexipage</type>
</profileActionOverrides>
```

**Metadata API割り当てのリスクと注意点:**
- デプロイ成功してもUI上で反映されないケースがある（原因不明、環境依存の可能性）
- SDO等のデモ環境では同名アプリ・同名プロファイルが複数存在することがあり、意図しない方に割り当たる
- `profile`フィールドは必須（省略すると `Required field is missing: profile` エラー）
- `CustomObject.actionOverrides` に `recordType` を入れると `Element recordType invalid` エラー
- **結論: FlexiPageのデプロイ自体はCLIで行い、割り当て（Activation）はUIで行うのが最も安全**

### 6-9. FlexiPageのよくあるエラー

| エラー | 原因 | 対処 |
|---|---|---|
| `Element recordType invalid at this location in type ActionOverride` | CustomObject.actionOverridesにrecordTypeを指定 | CustomApplication.profileActionOverridesを使う、またはUIで割り当て |
| `Required field is missing: profile` | CustomApplication.profileActionOverridesでprofile省略 | `<profile>プロファイル名</profile>` を追加 |
| `CustomApplicationを指定する必要があります` | Profile.profileActionOverridesで割り当てを試みた | CustomApplicationメタデータ側で設定する |
| デプロイ成功するがUIに反映されない | 環境依存（同名アプリ/プロファイルの重複等） | UIのLightning App Builder → Activationで手動割り当て |
| `force:relatedListSingleContainer` で関連リストが表示されない | カスタムLookupの関連リストはページレイアウトとの名前マッチに失敗するケースあり | `lst:dynamicRelatedList`（動的関連リスト）に変更する |

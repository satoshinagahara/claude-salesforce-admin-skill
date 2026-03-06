# Metadata: Lightning Web Components (LWC) Reference

## 概要
Lightning Web ComponentのRetrieve・新規作成・変更・Deployパターン。

---

## 1. LWCのファイル構成

```
force-app/main/default/lwc/<componentName>/
├── <componentName>.html         # テンプレート（必須）
├── <componentName>.js           # JavaScriptコントローラー（必須）
├── <componentName>.js-meta.xml  # メタ設定（必須）
├── <componentName>.css          # スタイル（任意）
└── <componentName>.svg          # アイコン（任意）
```

**命名規則**: camelCase（例: `myComponent`）。ハイフン不可。

---

## 2. 既存LWCのRetrieve

```bash
# 特定コンポーネントを取得
sf project retrieve start \
  --metadata "LightningComponentBundle:<componentName>" \
  --target-org <username>

# 全LWC一覧
sf org list metadata \
  --metadata-type LightningComponentBundle \
  --target-org <username>

# 全LWCを一括取得
sf project retrieve start \
  --metadata "LightningComponentBundle" \
  --target-org <username>
```

---

## 3. 新規LWC作成

### 3-1. CLIでスケルトン生成（推奨）

```bash
# プロジェクトディレクトリ内で実行
cd ~/sf-admin-work/sf-admin-project

sf lightning generate component \
  --name <componentName> \
  --type lwc \
  --output-dir force-app/main/default/lwc
```

これで3ファイル（html/js/js-meta.xml）が自動生成される。

### 3-2. 手動作成（テンプレートから）

#### `<componentName>.html`
```html
<template>
    <lightning-card title="コンポーネントタイトル" icon-name="standard:account">
        <div class="slds-m-around_medium">
            <p>{greeting}</p>
            <lightning-button label="クリック" onclick={handleClick}></lightning-button>
        </div>
    </lightning-card>
</template>
```

#### `<componentName>.js`
```javascript
import { LightningElement, api, track, wire } from 'lwc';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

export default class ComponentName extends LightningElement {
    @api recordId;         // 親からの入力プロパティ
    @track greeting = 'Hello, World!';  // リアクティブな内部状態

    handleClick() {
        this.greeting = 'Clicked!';
        this.dispatchEvent(new ShowToastEvent({
            title: '成功',
            message: '処理が完了しました',
            variant: 'success'
        }));
    }
}
```

#### `<componentName>.js-meta.xml`（配置場所の設定）
```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>63.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__RecordPage</target>
        <target>lightning__AppPage</target>
        <target>lightning__HomePage</target>
    </targets>
    <targetConfigs>
        <targetConfig targets="lightning__RecordPage">
            <property name="recordId" type="String" label="レコードID"/>
        </targetConfig>
    </targetConfigs>
</LightningComponentBundle>
```

**targets（配置場所）の主要選択肢**:
- `lightning__RecordPage`: レコード詳細ページ
- `lightning__AppPage`: アプリページ
- `lightning__HomePage`: ホームページ
- `lightning__FlowScreen`: フロー画面
- `lightningCommunity__Page`: Experience Cloud

---

## 4. よく使うLWCパターン

### 4-1. Apexを呼び出すLWC（@wire）

```javascript
import { LightningElement, api, wire } from 'lwc';
import getOpportunities from '@salesforce/apex/OpportunityController.getOpportunities';

export default class OpportunityList extends LightningElement {
    @api accountId;

    @wire(getOpportunities, { accountId: '$accountId' })
    opportunities;

    get hasData() {
        return this.opportunities.data && this.opportunities.data.length > 0;
    }
}
```

```html
<template>
    <template if:true={opportunities.data}>
        <template for:each={opportunities.data} for:item="opp">
            <p key={opp.Id}>{opp.Name} - {opp.StageName}</p>
        </template>
    </template>
    <template if:true={opportunities.error}>
        <p>エラーが発生しました</p>
    </template>
</template>
```

### 4-2. データテーブルを表示するLWC

```html
<template>
    <lightning-datatable
        key-field="Id"
        data={records}
        columns={columns}
        hide-checkbox-column>
    </lightning-datatable>
</template>
```

```javascript
import { LightningElement, wire } from 'lwc';
import getAccounts from '@salesforce/apex/AccountController.getAccounts';

const COLUMNS = [
    { label: '取引先名', fieldName: 'Name', type: 'text' },
    { label: '電話番号', fieldName: 'Phone', type: 'phone' },
    { label: '業種', fieldName: 'Industry', type: 'text' }
];

export default class AccountTable extends LightningElement {
    columns = COLUMNS;

    @wire(getAccounts)
    records;
}
```

### 4-3. フォームベースのLWC（レコード編集）

```html
<template>
    <lightning-record-edit-form
        record-id={recordId}
        object-api-name="Opportunity">
        <lightning-messages></lightning-messages>
        <lightning-input-field field-name="Name"></lightning-input-field>
        <lightning-input-field field-name="StageName"></lightning-input-field>
        <lightning-input-field field-name="CloseDate"></lightning-input-field>
        <lightning-button type="submit" label="保存"></lightning-button>
    </lightning-record-edit-form>
</template>
```

### 4-4. 子コンポーネントへのイベント送信

```javascript
// 親 → 子: @api プロパティで渡す
// 子 → 親: カスタムイベントをdispatch

// 子コンポーネントのJS
handleSave() {
    const event = new CustomEvent('save', {
        detail: { data: this.formData }
    });
    this.dispatchEvent(event);
}
```

```html
<!-- 親コンポーネントのHTML -->
<c-child-component onsave={handleChildSave}></c-child-component>
```

---

## 5. CSSスタイル

```css
/* <componentName>.css */
/* SLDSユーティリティクラスの使用を推奨 */
.custom-container {
    padding: 1rem;
}

/* CSS変数（SLDSトークン）を活用 */
.highlight {
    color: var(--lwc-colorTextActionActive, #0176d3);
    font-weight: var(--lwc-fontWeightBold, 700);
}
```

---

## 6. Validate → Deploy

```bash
# 作業ディレクトリに移動
cd ~/sf-admin-work/sf-admin-project

# Validate（デプロイ前必須）
sf project deploy validate \
  --metadata "LightningComponentBundle:<componentName>" \
  --target-org <username>

# Deploy
sf project deploy start \
  --metadata "LightningComponentBundle:<componentName>" \
  --target-org <username>

# 複数コンポーネントを一括Deploy
sf project deploy start \
  --metadata "LightningComponentBundle:<comp1>,LightningComponentBundle:<comp2>" \
  --target-org <username>
```

---

## 7. デプロイ後の確認

```bash
# デプロイしたコンポーネントが組織に存在するか確認
sf data query \
  --query "SELECT DeveloperName, NamespacePrefix FROM LightningComponentBundle WHERE DeveloperName = '<componentName>'" \
  --target-org <username>
```

---

## 8. よくあるエラーと対処

| エラー | 原因 | 対処 |
|---|---|---|
| `Component not found` | コンポーネント名のミス（大文字小文字） | camelCaseで正確に指定 |
| `Invalid target` | `js-meta.xml`のtarget設定が誤り | 有効なtarget値を使用 |
| `Import not found` | Apexクラスのimportパスが誤り | クラス名・メソッド名を確認 |
| `Template error` | HTMLテンプレートの構文エラー | `<template>`タグの閉じ忘れ等を確認 |
| `Wire adapter error` | @wireのプロパティ名誤り | Apexメソッドの引数名と一致させる |
| `Cannot read properties of undefined` | データロード前にアクセス | `if:true={data}`で条件分岐を追加 |

# metadata-experience-cloud

Experience Cloud（旧Community）/ ExperienceBundleのメタデータ操作知見。
デジタルサイト・パートナーポータル・顧客コミュニティの構築時に参照。

## 1. ExperienceBundleの取得・デプロイ

```bash
# 全ExperienceBundle取得
sf project retrieve start --target-org <user> --metadata ExperienceBundle --output-dir tmp_exp

# 特定サイトのみ
sf project retrieve start --target-org <user> --metadata ExperienceBundle:<siteApiName> --output-dir tmp_exp

# 変更後デプロイ
sf project deploy start --target-org <user> --source-dir tmp_exp/experiences --wait 15

# Publish（反映反映は別コマンド）
sf community publish --target-org <user> --name "<サイト表示名>"
```

**重要**: ExperienceBundleをデプロイしても、サイトに変更が反映されるのは `sf community publish` 実行後。

## 2. ExperienceBundleの構成

```
experiences/<siteApiName>/
├── brandingSets/      # カラー・フォント等のブランディング
├── config/            # サイト設定（nativeConfig.json等）
├── routes/            # URLルーティング (route-*.json)
├── themes/            # テーマ定義
└── views/             # 各ページの定義 (view-*.json)
```

- **route**: URL (`urlPrefix`) と `activeViewId` のマッピング
- **view**: ページ本体。`regions` 配下にLWC/Aura Componentを配置
- **route.routeType** と **view.viewType** は必ず一致させる

## 3. カスタムLWCの配置

view JSONの `regions` 配下に追加：

```json
{
  "regions": [{
    "components": [{
      "componentAttributes": {},
      "componentName": "c:myCustomLwc",
      "id": "<uuid>",
      "renderPriority": "NEUTRAL",
      "renditionMap": {},
      "type": "component"
    }],
    "id": "<uuid>",
    "regionName": "content",
    "type": "region"
  }]
}
```

- `componentName`: `c:<LwcApiName>`
- LWCの `*.js-meta.xml` に `<target>lightningCommunity__Page</target>` が必要

## 4. 新規ページ追加の手順と落とし穴

**基本手順**: views/ に新規JSON作成 → routes/ に対応するJSON作成 → deploy → publish

### 4.1 routeTypeは事前定義値のみ

routeTypeは任意文字列不可。**既存の未使用route（jointbusinessplan-list, mdf, voucher-list 等）を流用する**のが現実解。
Partner Central等のテンプレートには未使用routeが複数用意されている。

未使用routeの調査：
```bash
grep -r "routeType" tmp_exp/experiences/<site>/routes/ | grep -v "account\|contact\|opportunity"
```

### 4.2 viewType == routeType

route/viewペアでこれらが一致しないとデプロイ失敗：
```
エラー: srm_xxx/views/<name>.json の viewType 値と
srm_xxx/routes/<name>.json の routeType 値は一致する必要があり、
両方のファイルの appPageId は同じ SPA を参照する必要があります。
```

### 4.3 pageAccessプロパティはリリース後変更不可

```
エラー: リリース時に PageAccess プロパティを変更することはできません
```

新規route JSONでは `pageAccess` フィールドを含めない（省略）のが安全。

### 4.4 SEO非対応のrouteTypeがある

mdf, voucher-list 等の一部routeTypeは `sfdcHiddenRegion`（SEOリージョン）を持てない：
```
エラー: <routeType> ビューには、sfdcHiddenRegion という領域が含まれています。
これには通常、SEO プロパティが含まれます。ただし、<routeType> は
SEO プロパティをサポートしていません。非表示の領域を削除して、もう一度お試しください。
```

対処：viewのregionsから `sfdcHiddenRegion` を丸ごと削除。`forceCommunity:seoAssistant` コンポーネントも不要。

### 4.5 最小view JSONテンプレート（SEOなし・シンプル1カラム）

```json
{
  "appPageId": "<既存SPAのappPageId>",
  "componentName": "siteforce:sldsOneColLayout",
  "dataProviders": [],
  "id": "<新規UUID>",
  "label": "ページ表示名",
  "regions": [{
    "components": [{
      "componentAttributes": {},
      "componentName": "c:myLwc",
      "id": "<新規UUID>",
      "renderPriority": "NEUTRAL",
      "renditionMap": {},
      "type": "component"
    }],
    "id": "<新規UUID>",
    "regionName": "content",
    "type": "region"
  }],
  "themeLayoutType": "Inner",
  "type": "view",
  "viewType": "<routeTypeと同じ値>"
}
```

### 4.6 最小route JSONテンプレート

```json
{
  "activeViewId": "<対応viewのid>",
  "appPageId": "<view と同じappPageId>",
  "configurationTags": [],
  "devName": "CamelCaseDevName",
  "id": "<既存ルートID or 新規UUID>",
  "label": "メニュー表示名",
  "routeType": "<既存の未使用routeType>",
  "type": "route",
  "urlPrefix": "my-path"
}
```

## 5. レイアウトの選択

| componentName | 用途 |
|---|---|
| `siteforce:sldsOneColLayout` | シンプルな1カラム（SLDS） |
| `siteforce:dynamicLayout` | セクション分割の動的レイアウト |

複雑なページは `dynamicLayout`、LWCを1つだけ配置するなら `sldsOneColLayout` で十分。

## 6. ブランディング（色・フォント変更）

`brandingSets/<templateName>.json` を直接編集：

```json
{
  "ActionColor": "#D21D24",
  "LinkColor": "#D21D24",
  "brandNavigationBackgroundColor": "#D21D24",
  "brandNavigationBarBackgroundColor": "#A61319",
  "PrimaryFont": "Salesforce Sans",
  "HeaderFonts": "Salesforce Sans"
}
```

デプロイ → publish で反映される。

## 7. ナビゲーションメニュー

**ExperienceBundleとは別メタデータ** `NavigationMenu` として管理される。

```bash
sf project retrieve start --target-org <user> --metadata NavigationMenu --output-dir tmp_nav
```

`SFDC_Default_Navigation_<siteName>.navigationMenu-meta.xml` を編集：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<NavigationMenu xmlns="http://soap.sforce.com/2006/04/metadata">
    <container>サイト表示名</container>
    <containerType>Network</containerType>
    <label>Default Navigation</label>
    <navigationMenuItem>
        <label>ホーム</label>
        <position>0</position>
        <publiclyAvailable>false</publiclyAvailable>
        <target>/</target>
        <type>InternalLink</type>
    </navigationMenuItem>
    <!-- 追加項目 -->
</NavigationMenu>
```

- `target`: viewのurlPrefix（`/capacity` 等）
- `type`: `InternalLink` / `ExternalLink` / `MenuLabel`（サブメニュー用）
- サブメニュー: `<subMenu>` 配下に `<navigationMenuItem>` をネスト

デプロイ後、community publishが必要。

## 8. Experience Cloudサイトのユーザ作成

**CommunitiesSettings** を先に有効化しないと外部プロファイルでUser作成ができない：

```bash
sf project retrieve start --target-org <user> --metadata Settings:Communities --output-dir settings_retrieve
```

`settings/Communities.settings-meta.xml` で `enableOotbProfExtUserOpsEnable` を `true` に：

```xml
<enableOotbProfExtUserOpsEnable>true</enableOotbProfExtUserOpsEnable>
```

User作成時の注意点は `references/metadata-security.md` および auto-memory の
`feedback_experience_cloud_user_creation.md` を参照（MIXED_DML分割、PSライセンス不一致時のdestructive再作成など）。

## 9. デバッグ定石

| 症状 | 確認項目 |
|---|---|
| デプロイ後ページが表示されない | `sf community publish` を実行したか |
| ページはあるがLWCが表示されない | LWCの `*.js-meta.xml` に `lightningCommunity__Page` target |
| ポータルユーザで権限エラー | 権限セットのLicenseが `Partner Community` / `Customer Community` と一致しているか |
| routeType関連エラー | 既存未使用routeを流用、viewType==routeTypeを確認 |
| リリース時プロパティ変更不可エラー | `pageAccess` フィールドを削除 |
| SEOリージョン非対応エラー | `sfdcHiddenRegion` と `seoAssistant` を削除 |

## 10. LWC側 — URL query parameter の取得

Experience Cloudのカスタムページで URL query param（例: `/rfq-response?rfqId=xxx` の `rfqId`）を
LWCから読むには `CurrentPageReference` wire を使う：

```javascript
import { LightningElement, api, wire } from 'lwc';
import { CurrentPageReference } from 'lightning/navigation';

export default class MyComponent extends LightningElement {
    @api recordId;      // RecordPage配置時に自動注入
    @api rfqId;         // AppPage/Communityデザインパネルからの明示指定
    urlRfqId;           // URL query paramから取得するフィールド

    @wire(CurrentPageReference)
    wiredPageRef(pageRef) {
        if (pageRef?.state) {
            // Experience Cloudのクエリパラメータは state に入る
            // c__ プレフィックス版もケースによって併用される
            this.urlRfqId = pageRef.state.rfqId || pageRef.state.c__rfqId;
        }
    }

    // fallback チェーンで RecordPage / AppPage / Community の3配置に対応
    get targetRfqId() {
        return this.rfqId || this.recordId || this.urlRfqId;
    }
}
```

ポイント：
- `pageRef.state` に全クエリパラメータが格納される
- 一部コンテキストでは `c__` プレフィックス付きで渡ることがあるため両方チェック推奨
- `@api` プロパティ（recordId/rfqId等）とのfallbackチェーンで配置パターンを吸収できる

## 11. LWC側 — カスタムページへのNavigation

Experience Cloud内のカスタムページへ遷移するには `standard__webPage` 型が最もシンプル：

```javascript
import { NavigationMixin } from 'lightning/navigation';

export default class MyComponent extends NavigationMixin(LightningElement) {
    handleClick(event) {
        const rfqId = event.currentTarget.dataset.rfqId;
        this[NavigationMixin.Navigate]({
            type: 'standard__webPage',
            attributes: { url: `/rfq-response?rfqId=${rfqId}` }
        });
    }
}
```

Navigation型の使い分け：

| 型 | 用途 | 備考 |
|---|---|---|
| `standard__webPage` | **カスタムページ（推奨）** | 相対URL `/path?param=val` でOK、クエリパラメータ自由 |
| `comm__namedPage` | 名前付きComm ページ | `attributes.name` に route devName。クエリは `state` で渡す |
| `standard__recordPage` | 標準レコードページ | Experience Cloudでは標準レコードページアクセス権限が必要。ポータルユーザが標準UIを持たない場合は使えない |

**カスタム応答フォーム等をポータルユーザに提供する用途**では `standard__webPage` + カスタムページ構成が現実解。
ポータルユーザが `standard__recordPage` を使えない環境でも動作する。

## 12. ポータルユーザのサイトメンバー登録（NetworkMemberGroup）

**最重要の詰まりポイント**: Experience Cloudユーザー(User) を作成しただけでは**サイトにログインできない**。
サイト(Network)の「メンバープロファイル」にそのユーザのプロファイルが登録されていないと、

- Login-Asボタンが表示されない（Contactページでサイト選択肢が空）
- `/secur/frontdoor.jsp` でログインしても「このユーザーはどのエクスペリエンスサイトのメンバーでもありません」で弾かれる
- sf cli の otp / sid URL も遷移後に即セッション切れになる

### 設定方法（UI）

Setup > Digital Experiences > All Sites > 該当サイト > **Administration > Members**
→ 「利用可能なプロファイル」から Partner Community User 等を「選択済み」に移動 → 保存

### 設定方法（API / CLI・自動化向け）

`NetworkMemberGroup` オブジェクトに1行insert。

```bash
# 対象Network（サイト）のId確認
sf data query --target-org <user> \
  --query "SELECT Id, Name, UrlPathPrefix FROM Network WHERE Status='Live'"

# 追加したいProfileのId確認
sf data query --target-org <user> \
  --query "SELECT Id, Name FROM Profile WHERE Name = 'Partner Community User'"

# メンバー追加
sf data create record --target-org <user> \
  --sobject NetworkMemberGroup \
  --values "NetworkId=<network-id> ParentId=<profile-id>"

# 確認
sf data query --target-org <user> \
  --query "SELECT Id, NetworkId, ParentId FROM NetworkMemberGroup WHERE NetworkId = '<network-id>'"
```

- `NetworkMemberGroup.ParentId` は Profile の Id（PermissionSetでも可）
- 1プロファイル = 1行

## 13. Experience CloudユーザーのLogin-As正規ルート

**Setup > Users の「ログイン」リンクは内部ユーザー向けで、Experience Cloudサイトを選択させてくれない**。
Portal ユーザーを「サイトメンバーとして」Login-As するには別ルートを使う。

### 正規ルート: Contact（取引先責任者）ページから

1. 管理者でLightning画面にログイン
2. ポータルユーザに紐付く **Contact（取引先責任者）のレコードページ** を開く
3. ヘッダ右側のアクションメニュー（▼）から **「Log in as」** をクリック
4. モーダルに**メンバーになっているサイトの一覧**が表示される（§12設定が効いていればここに出る）
5. 対象サイトをクリック → そのサイトのポータル画面にLogin-As

### 動かない時のチェックリスト

| 症状 | 原因 | 対処 |
|---|---|---|
| User詳細に「どのサイトのメンバーでもありません」 | NetworkMemberGroup未登録 | §12で追加 |
| Contactに「Log in as」ボタンが出ない | サイトが Live でない / Contact 紐付き不足 | Publicサイト状態確認 |
| モーダルの選択肢が空 | 同上（プロファイルがサイトメンバーに未登録） | §12で追加 |
| frontdoor.jsp で「無操作状態のためログアウト」 | otp短命 + MFA/IPロック + サイト未メンバー | §12で追加 + Contact経由に切替 |

### otp/sid方式でのダイレクトLogin-As（補足）

MFA必須orgでは `sf org open --url-only` 経由のotpが短命ですぐ切れるため、**CLIベースのブラウザLogin-Asはdemo orgでは動かないことが多い**。
その場合は：
- Step1: 普段使いのブラウザ認証で管理者ログイン
- Step2: Contactレコード > Log in as > サイト選択

の2段構えが最も確実。

## 14. "Salesforce CDN partner" エラーの誤認防止

Experience Cloud サイトにアクセスすると以下のエラーが出ることがある：

- `An unexpected connection error occurred.`
- `This error originated from Salesforce CDN partner.`
- `Looks like the site is temporarily unavailable`（失敗時のAkamai SNA失敗ページ）

**エラー文言に "CDN partner" とあっても CDN が原因とは限らない**。
Trailblazer Community の実例では、VisualForce ページが**非ActiveユーザへのChatter followリンク**を表示しようとしていたのが真因で、CDN は無関係だった。しかもこのケースは**本番のみ再現・sandbox では無問題**で、**デバッグログにもエラーが出なかった**ため特定が難航した。

### 切り分けの観点

| 症状パターン | 原因候補 | 次のアクション |
|---|---|---|
| 全ユーザ・全ページで常に503 | CDN/SNA failover / Known Issue | サポート案件 or 後述のKnown Issue該当確認 |
| 特定プロファイルのみ失敗 | 権限/プロファイル設定 | プロファイル別FLS/VF/Apex権限確認 |
| 特定ページのみ失敗 | ページ内の壊れた参照（VF/LWC/Apex） | ページビルダーでコンポーネントを外して切り分け |
| 本番のみ再現・sandboxではOK | **データ差異に起因する表示エラー** | 非Activeユーザ参照・欠番リンクなど本番固有のデータ不整合 |
| Apex/VFデバッグログに何も出ない | クライアント/CDN層で弾かれている | ブラウザ開発者ツールでHTTPヘッダ・リダイレクト確認 |

### 調査のコツ

1. **ページビルダーで疑わしいコンポーネントを1つずつ外して Publish** → どのコンポーネントが原因か二分探索
2. **別プロファイル・別ユーザで再現確認** → 権限/データ切り分け
3. **sandbox に同じ構成を複製して比較** → データ差異の切り分け
4. ブラウザの Network タブで **`sna_page` や `server: AkamaiNetStorage` ヘッダ**の有無確認

### 関連 Known Issue（**API経路の話**・HTML配信の503とは別物）

以下は全て「**API/Auth経路で断続的に503**」が発生するKIで、**ブラウザでサイトHTMLを取りに行って常時503** の症状には当てはまらないので混同注意。

| KI | スコープ | ステータス |
|---|---|---|
| [a028c00000iw66AAAQ](https://help.salesforce.com/s/issue?id=a028c00000iw66AAAQ) | **SOAP API** + `Expect: 100-continue` ヘッダ付 + CDN経由 → 断続的503。WA: ヘッダ除外 or CDN無効化 | Closed (Spring'23) |
| [a028c00000yEOm1AAG](https://help.salesforce.com/s/issue?id=a028c00000yEOm1AAG) | **SOAP/REST API** Hyperforce移行後 → 断続的503 `upstream connect error`。WA: リトライロジック | Closed (Spring'24) |
| [a028c00000gAyoTAAS](https://help.salesforce.com/s/issue?id=a028c00000gAyoTAAS) | **Auth Provider** 認証時 AMR null → 503 | Solution Deployed (Spring'21) |

### サイトHTML配信が常時503の場合の別シナリオ

- **SNA failover ページ**（`server: AkamaiNetStorage` / `sna_page:` ヘッダ）が返っている → サイトのビルド成果物 or ExperienceBundle 破損の可能性
- 他サイトは200 OK、特定サイトのみ503 → org全体のCDN問題ではなく**サイト個別**の問題
- 切り分けは §14 上部の「調査のコツ」参照

### 参考

- Trailblazer Community 投稿 (2023-09): https://trailhead.salesforce.com/ja/trailblazer-community/feed/0D54V00007I56F2

# DocuSign Apps Launcher (dfsle) — Salesforce 電子署名連携

DocuSign Apps Launcher (DAL, 名前空間 `dfsle`) を使った SF レコード起点の電子署名フロー構築の要点。AppExchange 現行版は DAL 1 パッケージに eSignature / Gen / Negotiate が同梱（旧スタンドアロン `dsfs` は廃止）。

> 前提: DocuSign アカウント接続・テンプレート作成・PowerForm・Connect 設定は **DocuSign 認証が要る UI 作業**でユーザー側。Claude が担えるのは SF メタデータ（権限割当・レイアウト・フロー・タブ・LWC）と CLI 確認。

## 1. 署名方式の選択（最重要）

| 方式 | 署名UX | 自動書き戻し | demoでの安定性 |
|---|---|---|---|
| **内部Send → メール署名（推奨）** | 署名者はメールリンクで標準DocuSign署名 | ◎ `EnvelopeStatus.SourceId` 経由 | ◎ 確実 |
| Experience Cloud 埋め込み(embedded) | ポータル内VFで署名 | ◎ | △ CDN/メンバー上限で詰まりやすい |
| PowerForm iframe | 任意ページに iframe 埋め込み | ✗ SourceId 付かず | △ |

**demo/PoC は「内部ユーザーが契約レコードから Send → 署名者にメール → 署名 → レコード自動更新」が最も確実。**

## 2. 権限セット（dfsle 提供）

```bash
sf data query -o <org> -q "SELECT Name,Label FROM PermissionSet WHERE NamespacePrefix='dfsle'"
```
- `DocuSign_User` / `DocuSign_Login` / `DocuSign_Administrator` … **DocuSign 正式メンバーになる内部ユーザー向け**
- `DocuSign_Sender`（DSSender）… **Experience Cloud/外部ユーザーが System Sender 経由で送信・署名するため**。メンバー登録は伴わない

### ★落とし穴: ポータル署名者に Login/User を付けない
外部(ポータル)署名者には **`DocuSign_Sender` のみ**付与する。`DocuSign_Login`/`DocuSign_User` を付けると DAL の `dfsle.AddUsersJob` がそのユーザーを DocuSign メンバー登録しようとし、**Developer/Demo アカウントはメンバー数上限が小さい**ため
`The maximum number of members for the account has been exceeded` で失敗 → ポータルの署名ボタン押下が「An unexpected connection error occurred」になる。
エラーは **`dfsle__Log__c`（Docusignログ）** に記録される：
```bash
sf data query -o <org> -q "SELECT Name,CreatedDate,dfsle__Severity__c,dfsle__Class__c,dfsle__Message__c FROM dfsle__Log__c ORDER BY CreatedDate DESC LIMIT 5"
```
（class=`dfsle.AddUsersJob`, severity=`ERROR` が出ていれば本件）

権限割当（接続後、CLIで代行可）:
```bash
sf data create record -o <org> -s PermissionSetAssignment --values "AssigneeId='<userId>' PermissionSetId='<psId>'"
```

## 3. System Sender
非DocuSignユーザー（ポータル署名者）が署名フローを動かすには、DocuSign Apps Launcher の「設定」で **System Sender**（DocuSign管理者ユーザー）を有効化する。封筒は System Sender 名義で送られ、完了証明書には実際の署名者が記録される。

## 4. 書き戻し（dfsle__EnvelopeStatus__c）

DocuSign Connect（DAL が接続時に自動構成）が署名状況を **`dfsle__EnvelopeStatus__c`** に書く。主要項目:

| 項目 | 内容 |
|---|---|
| `dfsle__SourceId__c` | **送信元レコードId**（レコードからSend/署名した封筒にのみ付く。PowerForm等は付かない） |
| `dfsle__DocuSignId__c` | DocuSign Envelope ID |
| `dfsle__Status__c` | Sent / Delivered / Completed / Declined |
| `dfsle__Completed__c` / `dfsle__Sent__c` | 完了 / 送信 日時 |

### 業務オブジェクトへ反映する2系統（併用可）
1. **レコードトリガーフロー**: `dfsle__EnvelopeStatus__c`(RecordAfterSave) で起動し、`SourceId` が対象オブジェクトの keyPrefix なら状況を業務項目へコピー（Completed→自社ステータス=署名済 等）。`SourceId` が業務レコードIdそのものなので Update Records の条件 `Id = {!$Record.dfsle__SourceId__c}` で更新可。
2. **DALテンプレートの「オプション→フィールドの自動更新」**: エンベロープイベント（送信/完了 等）ごとに、データソース(業務オブジェクト)の項目を最大5つ直接更新。Date型は選べないことがある（→フローで補完）。

### 署名済PDF
テンプレート「オプション→文書の書き戻し」を ON にすると、**署名済PDF＋完了証明書がレコードの Files (ContentVersion) に保存**される。確認:
```bash
sf data query -o <org> -q "SELECT ContentDocument.Title,ContentDocument.FileType FROM ContentDocumentLink WHERE LinkedEntityId='<recordId>'"
```

## 5. ボタン（テンプレート公開で生成される WebLink）
DALテンプレートの「カスタムボタン」ステップで、対象オブジェクトに URL ボタン(WebLink, DisplayType=B)が作られページレイアウトの `<customButtons>` に追加される。URL 例:
- メール署名: `$Site.BaseUrl + '/apex/dfsle__sending'` … `isEmbedded=false`（内部Send画面）
- 埋め込み: `$Site.BaseUrl + '/apex/dfsle__embeddedsigning'` … `isEmbedded=true`

### ★Lightning/コミュニティで表示するには platformActionList が必要
`<customButtons>` に入るだけでは Lightning レコードページに出ない。レイアウトの **`<platformActionList>`（Salesforceモバイル/Lightning アクション）** に追加する（→ `metadata-ui.md`）。

### ★1テンプレ＋送信時文書差し替え＋共通アンカーで契約種別を吸収（ボタン増殖回避）
「1テンプレ＝1固定文書」のため種別ごとにテンプレを作るとボタンが増殖するが、回避策がある：
- DALテンプレートの文書自体はロックされ後から差し替え不可。**しかし送信フロー(`dfsle__sending`)で文書をアップロード差し替えすると、テンプレートの署名タブがアンカー文字列(例 `／sig_supplier／`)基準で新文書に自動再配置される**。
- よって **全契約書に共通アンカー文字列を仕込み、送信時にその契約の文書へ差し替える**運用にすれば、**1テンプレ・1ボタンで全契約種別に対応**できる。
- 前提: 署名タブをアンカー基準にすること（固定座標タブだと差し替え後に再配置されない）。テンプレ「フィールドの配置」でアンカー文字列を指定するか、アンカーテキスト近傍にタブを置く。

## 6. embedded(Experience Cloud) が「Service Not Available」になる時
DALの埋め込みボタンは `/srm-prefix/apex/dfsle__embeddedsigning` を新規タブで開くが、**Aura コミュニティ経由のVF配信が CDN(Akamai) 層で失敗**することがある（VFアクセス権があっても）。公式「Prepare Your Salesforce Organization for Experience Cloud」のサイト準備が別途必要で、demoでは安定しないため **メール署名方式を推奨**。詳細な切り分けは `metadata-experience-cloud.md` §14。

## 7. その他の落とし穴
- **署名者の Contact メールは実在受信箱が必須**（架空ドメインだと届かない）。demoは Gmail の +alias が便利。
- **CSP は DAL が `*.docusign.net`/`*.docusign.com` を登録済**（frameSrc=true, All）。追加不要。
  ```bash
  sf data query --use-tooling-api -o <org> -q "SELECT DeveloperName,EndpointUrl,IsApplicableToFrameSrc FROM CspTrustedSite WHERE EndpointUrl LIKE '%docusign%'"
  ```
- **テンプレートのタグ付け画面(account-d.docusign.com)が「接続拒否」**: ブラウザのサードパーティCookieブロックが原因 → `[*.]docusign.com`/`[*.]docusign.net` を許可。
- **1 Salesforce org : 1 DocuSign account**。sandbox refresh 後は Connect 再認証が要る場合あり。
- `help.salesforce.com` の「Permission Sets For DocuSign」は別製品（Salesforce Contracts/CLM）向け。AppExchange dfsle 版とは権限体系が違うので混同しない。

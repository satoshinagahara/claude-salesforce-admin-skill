# Data Model, Mapping & Identity Resolution Reference

## 概要
Data360のデータモデルは3層構造で構成される：
1. **Data Lake Object (DLO)**: データストリームから取り込まれた生データ
2. **Data Model Object (DMO)**: 正規化・統合されたデータモデル（標準/カスタム）
3. **マッピング**: DLO → DMO のフィールドマッピング

DMOに統合されたデータが、Agentforceのグラウンディングやセグメント作成等で活用される。

---

## 1. データモデルの概念

### 1-1. 標準DMO（Standard Data Model Object）

Data Cloud に標準で用意されているDMO：

| DMO名 | API名 | 説明 |
|---|---|---|
| Individual | Individual__dlm | 個人（統合プロファイル） |
| Contact Point Email | ContactPointEmail__dlm | メールアドレス |
| Contact Point Phone | ContactPointPhone__dlm | 電話番号 |
| Contact Point Address | ContactPointAddress__dlm | 住所 |
| Account | Account__dlm | 取引先 |
| Sales Order | SalesOrder__dlm | 受注 |
| Product | Product__dlm | 商品 |
| Engagement | Engagement__dlm | エンゲージメント |

**DMOのサフィックス**: `__dlm`（Data Lake Model）

### 1-2. カスタムDMO

標準DMOに当てはまらないデータ用にカスタムDMOを作成可能。

---

## 2. データモデルの確認

### 2-1. DMO一覧の取得

```bash
# Data Model Object 一覧
sf api request rest \
  --method GET \
  --url "/services/data/v62.0/ssot/data-model-objects" \
  --target-org <username>
```

### 2-2. 特定DMOのフィールド情報

```bash
sf api request rest \
  --method GET \
  --url "/services/data/v62.0/ssot/data-model-objects/<dmoName>" \
  --target-org <username>
```

### 2-3. DLO一覧の取得

```bash
sf api request rest \
  --method GET \
  --url "/services/data/v62.0/ssot/data-lake-objects" \
  --target-org <username>
```

---

## 3. データマッピング（DLO → DMO）

### 3-1. マッピングの概念

データストリームで取り込んだDLOのフィールドを、DMOのフィールドに対応付ける。
例: CRM の `Account.Name` → DMO の `Account__dlm.Name__c`

### 3-2. 現在のマッピング確認

```bash
# マッピング一覧
sf api request rest \
  --method GET \
  --url "/services/data/v62.0/ssot/data-mappings" \
  --target-org <username>

# 特定データストリームのマッピング
sf api request rest \
  --method GET \
  --url "/services/data/v62.0/ssot/data-mappings/<mappingName>" \
  --target-org <username>
```

### 3-3. マッピングの作成・更新（UIベース）

Data Mappingの作成・変更は主にUIで行う：
1. Setup → Data Cloud → Data Streams → 対象ストリームを選択
2. "Data Mapping" タブ
3. ソースフィールド（DLO）とターゲットフィールド（DMO）を対応付け
4. Save

### 3-4. マッピングのベストプラクティス

- **Primary Key** は必ずマッピングする（レコードの一意性担保）
- **DateTime フィールド** のフォーマットを統一する（ISO 8601推奨）
- **マッピング先のDMO** は先に存在している必要がある
- 標準DMOを使える場合はカスタムDMOより標準を優先する

---

## 4. Identity Resolution（統合ルール）

### 4-1. 概念

複数のデータソースから取り込まれた個人データを統合し、一意の Individual プロファイルに名寄せする機能。

統合プロセス：
```
データストリーム → DLO → マッピング → DMO → Identity Resolution → Unified Individual
```

### 4-2. ルールセット（Ruleset）の構成要素

| 要素 | 説明 |
|---|---|
| **Match Rule** | 2つのレコードが同一人物か判定するルール（例: メール完全一致） |
| **Reconciliation Rule** | 重複時にどのソースの値を優先するかのルール |
| **Data Source Priority** | ソースごとの信頼度の優先順位 |

### 4-3. Match Rule の種類

| タイプ | 説明 | 精度 |
|---|---|---|
| **Exact** | 完全一致 | 高（誤マッチ少ないが漏れあり） |
| **Fuzzy** | あいまい一致（名前のゆらぎ等） | 中（カバー範囲広いが誤マッチリスク） |
| **Normalized** | 正規化後の一致（大小文字、スペース等） | 中〜高 |

### 4-4. Identity Resolution の確認

```bash
# Identity Resolution ルールセット一覧
sf api request rest \
  --method GET \
  --url "/services/data/v62.0/ssot/identity-resolution/rulesets" \
  --target-org <username>

# 特定ルールセットの詳細
sf api request rest \
  --method GET \
  --url "/services/data/v62.0/ssot/identity-resolution/rulesets/<rulesetId>" \
  --target-org <username>
```

### 4-5. Identity Resolution ジョブの確認

```bash
# 最新の統合ジョブの状態
sf api request rest \
  --method GET \
  --url "/services/data/v62.0/ssot/identity-resolution/jobs" \
  --target-org <username>
```

### 4-6. Identity Resolution の設定（UIベース）

統合ルールの作成・変更はUIから行う：
1. Setup → Data Cloud → Identity Resolution
2. ルールセットを新規作成 or 編集
3. Match Rule: マッチ対象フィールドとマッチタイプを設定
4. Reconciliation Rule: ソース優先度を設定
5. Save & Run

---

## 5. Data Cloud SOQL

### 5-1. DMOへのクエリ

Data Cloud上のDMOは通常のSOQLでクエリ可能（Data Lake Object経由）：

```bash
sf data query \
  --query "SELECT Id, ssot__Name__c, ssot__Email__c FROM Individual__dlm LIMIT 10" \
  --result-format json \
  --target-org <username>
```

**注意**: DMOフィールドには `ssot__` プレフィックスが付く場合がある。

### 5-2. Data Cloud SOQL の制限

- `GROUP BY` / `ORDER BY` / `HAVING` 対応だが、一部集計関数に制限あり
- `LIKE` は使用可能だが、ワイルドカードのパフォーマンスに注意
- 大量データの場合は `LIMIT` + `OFFSET` でページネーション
- Relationship Query（子→親）は制限がある場合あり

---

## 6. Calculated Insight オブジェクトの参照

Calculated Insightで作成された指標もDMOとしてクエリ可能：

```bash
sf data query \
  --query "SELECT Id, Name__c, Value__c FROM <CalculatedInsightDMO>__dlm LIMIT 10" \
  --result-format json \
  --target-org <username>
```

---

## 7. メタデータによるデータモデル管理

### 7-1. package.xml

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
    <types>
        <members>*</members>
        <name>ExternalDataTranField</name>
    </types>
    <version>62.0</version>
</Package>
```

---

## 8. queryv2 の注意事項

### 8-1. 行の返却順序は非保証

Data Cloud queryv2（`/services/data/v62.0/ssot/queryv2`）は**行の返却順序を保証しない**。

- CSVの並び順（例：親行の後に子行が続く）に依存したロジックは破綻する
- `ORDER BY` 句はサポートされていないため、Apex側でソートするか、構造的に順序不要な設計にする
- 対策: 順序に依存しないキーベースの判定を使う（例：品目コードのプレフィックスで親子関係を推定）

### 8-2. HTTPステータスコード

- queryv2のレスポンスは **200ではなく201（Created）**
- Apexの制約: DML実行後はHTTP calloutが使えない。calloutを先に全て実行し、DMLをまとめて後から実行する設計が必要

---

## 9. よくあるエラーと対処

| エラー | 原因 | 対処 |
|---|---|---|
| `Object not found: XXX__dlm` | DMOが存在しない or マッピング未完了 | データストリーム→マッピングが完了しているか確認 |
| `Field not found: ssot__XXX__c` | フィールド名が不正 | `GET /ssot/data-model-objects/<dmo>` でフィールド名を確認 |
| `Identity resolution job failed` | 統合ルールのエラー | ルールセットの設定を確認、データ品質をチェック |
| `Mapping conflict` | 同一フィールドに複数マッピング | マッピングの重複を解消 |
| `DLO not available` | データストリームが未同期 | ストリームのステータスを確認、手動リフレッシュ |
| `/ssot/` API が `NOT_FOUND` | Data Cloudは有効でもAPIユーザーに `CDPAdmin` 権限セットが未割り当て | `CDPAdmin` をPermissionSetAssignmentで割り当てる |

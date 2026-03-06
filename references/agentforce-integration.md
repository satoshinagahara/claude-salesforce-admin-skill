# Agentforce × Data360（Data Cloud）連携 Reference

## 概要
AgentforceエージェントがData Cloud上の統合データをグラウンディングデータとして参照し、回答精度を高める仕組み。
Agentforceの基本（GenAiFunction/GenAiPlugin/GenAiPlannerBundle等）は `metadata-agentforce.md` を参照。

---

## 1. 連携パターン

| パターン | 説明 | 実装方法 |
|---|---|---|
| **Data Cloud Object アクション** | DMOのレコードを検索・取得するアクション | Flow または Apex で DMO にSOQL |
| **Semantic Search（ベクトル検索）** | 非構造データをベクトル化して意味検索 | Data Cloud Vector Database + Search Index |
| **Data Graph** | 複数DMOを結合したビューをエージェントに提供 | Data Graph 定義 → Apex/Flow で参照 |
| **RAG（Retrieval-Augmented Generation）** | 検索結果をLLMのコンテキストに注入して回答生成 | Prompt Template + Data Cloud グラウンディング |

### アーキテクチャ概要

```
ユーザーの質問
    ↓
Agentforce Agent（GenAiPlannerBundle）
    ↓ トピック選択 → アクション選択
Flow / Apex
    ↓ Data Cloudクエリ
Data Cloud DMO / Data Graph / Vector Search
    ↓ 結果
Agent が回答を生成
```

---

## 2. Data Cloud Object を使ったアクション

### 2-1. Flow で DMO にクエリする

Data Cloud のDMOはFlowの「Get Records」要素から直接参照可能：

1. Flow Builder で新規フロー作成（Auto-launched）
2. Get Records 要素を追加
3. オブジェクトとして `<DMO名>__dlm` を選択
4. フィルタ条件を設定
5. 結果をFlow変数に格納
6. フローを保存・有効化
7. GenAiFunctionとして登録（`invocationTargetType: flow`）

### 2-2. Apex で DMO にクエリする

```java
public class DataCloudQueryAction {
    @InvocableMethod(label='Query Data Cloud' description='Data Cloud DMOを検索')
    public static List<Result> queryDMO(List<Request> requests) {
        String searchKey = requests[0].searchKey;

        List<SObject> records = Database.query(
            'SELECT Id, ssot__Name__c, ssot__Email__c ' +
            'FROM Individual__dlm ' +
            'WHERE ssot__Name__c LIKE \'%' + String.escapeSingleQuotes(searchKey) + '%\' ' +
            'LIMIT 10'
        );

        List<Result> results = new List<Result>();
        Result r = new Result();
        r.records = JSON.serialize(records);
        r.recordCount = records.size();
        results.add(r);
        return results;
    }

    public class Request {
        @InvocableVariable(label='Search Key' required=true)
        public String searchKey;
    }

    public class Result {
        @InvocableVariable(label='Records JSON')
        public String records;
        @InvocableVariable(label='Record Count')
        public Integer recordCount;
    }
}
```

作成後、GenAiFunctionとして登録する手順は `metadata-agentforce.md` セクション6を参照。

---

## 3. Data Graph

### 3-1. Data Graph とは

複数のDMOをリレーションで結合し、1つのビューとして定義する機能。
エージェントが複数のDMOを横断して情報を取得する際に有用。

### 3-2. Data Graph の確認

```bash
sf api request rest \
  --method GET \
  --url "/services/data/v62.0/ssot/data-graphs" \
  --target-org <username>

sf api request rest \
  --method GET \
  --url "/services/data/v62.0/ssot/data-graphs/<graphName>" \
  --target-org <username>
```

### 3-3. Data Graph の作成（UI）

1. Setup → Data Cloud → Data Graphs → New
2. ルートDMOを選択（例: Individual__dlm）
3. 関連DMOを追加（例: ContactPointEmail__dlm, Account__dlm）
4. リレーションを定義
5. Save

### 3-4. Data Graph を Apex から利用

```java
List<SObject> results = Database.query(
    'SELECT ssot__Name__c, ' +
    '  (SELECT ssot__EmailAddress__c FROM ContactPointEmails__dlm) ' +
    'FROM Individual__dlm ' +
    'WHERE ssot__Name__c LIKE \'%田中%\' ' +
    'LIMIT 5'
);
```

---

## 4. Semantic Search（ベクトル検索）& RAG

### 4-1. ベクトル検索の概要

Data Cloud に格納されたテキストデータ（ナレッジ記事、FAQ等）をベクトル化し、ユーザーの質問に意味的に近い情報を検索する。

### 4-2. 前提条件

1. **Einstein Trust Layer** が有効化されていること
2. **Search Index** が Data Cloud 上に作成されていること
3. 対象データが Data Cloud に取り込まれていること

### 4-3. Search Index の作成（UI）

1. Setup → Data Cloud → Vector Database → New Search Index
2. 対象DMOとテキストフィールドを選択
3. チャンク戦略を設定（Fixed Size / Sentence等）
4. Embedding モデルを選択
5. Save & Build

### 4-4. Prompt Template でのグラウンディング

Prompt Template にData Cloudの検索結果を注入する：

1. Setup → Einstein → Prompt Templates → New
2. テンプレートタイプ: "Flex" または "Generate"
3. グラウンディングソースとして "Data Cloud" を選択
4. Search Index を指定
5. テンプレート内で `{!$Grounding.DataCloud.searchResults}` 変数を使用

### 4-5. Prompt Template の構造例

```
あなたは{!$Input:companyName}のカスタマーサポートアシスタントです。

以下のナレッジベースの情報を参考に、顧客の質問に回答してください。

--- ナレッジベース ---
{!$Grounding.DataCloud.searchResults}
--- ここまで ---

顧客の質問: {!$Input:customerQuestion}

回答:
```

### 4-6. GenAiPromptTemplate メタデータ

```xml
<?xml version="1.0" encoding="UTF-8"?>
<GenAiPromptTemplate xmlns="http://soap.sforce.com/2006/04/metadata">
    <activeVersionNumber>1</activeVersionNumber>
    <description>Data Cloudグラウンディング付きプロンプト</description>
    <masterLabel>Customer Support RAG</masterLabel>
    <templateVersions>
        <content>テンプレート本文...</content>
        <inputs>
            <apiName>customerQuestion</apiName>
            <dataType>Text</dataType>
            <definition>顧客からの質問</definition>
        </inputs>
        <primaryModel>sfdc_ai__DefaultGPT4Omni</primaryModel>
        <status>Published</status>
        <versionNumber>1</versionNumber>
    </templateVersions>
    <type>flex</type>
</GenAiPromptTemplate>
```

---

## 5. 全体実装フロー

```
1. Data Cloud にデータを取り込む（データストリーム）         → data-streams.md
2. DLO → DMO にマッピング                                   → data-model.md
3. (オプション) Identity Resolution で名寄せ                 → data-model.md
4. (オプション) Data Graph で複数DMOを結合                   → 本ファイル セクション3
5. (オプション) Search Index でベクトル化                     → 本ファイル セクション4
6. Apex / Flow で DMO にアクセスするロジックを作成           → 本ファイル セクション2
7. GenAiFunction としてアクションを登録                      → metadata-agentforce.md セクション6
8. トピック/Agent に追加                                      → metadata-agentforce.md セクション7-8
```

---

## 6. Data Cloud 権限の設定

### 6-1. 必要な権限セット

| 権限セット | 用途 |
|---|---|
| **Data Cloud Admin** | Data Cloud の管理操作（ストリーム・モデル・ルール設定） |
| **Data Cloud User** | Data Cloud データの参照 |
| **Data Cloud for Agents** | Agentforce が Data Cloud にアクセスするための権限 |

### 6-2. Agent ユーザーへの権限付与

Agentforce のシステムユーザー（`agentuser@...ext`）にも Data Cloud 権限が必要：

```bash
# Data Cloud 関連の権限セットを検索
sf data query \
  --query "SELECT Id, Name FROM PermissionSet WHERE Name LIKE '%DataCloud%' OR Name LIKE '%CDP%'" \
  --target-org <username>

# Agent ユーザーに権限セットを割り当て
sf data create record --sobject PermissionSetAssignment \
  --values "PermissionSetId=<権限セットID> AssigneeId=<AgentユーザーID>" \
  --target-org <username>
```

---

## 7. よくあるエラーと対処

| エラー | 原因 | 対処 |
|---|---|---|
| `DMO not queryable from Agent` | Agent ユーザーに Data Cloud 権限がない | Data Cloud for Agents 権限セットを割り当て |
| `Search Index not ready` | ベクトルインデックスのビルドが未完了 | UI で Build ステータスを確認、完了を待つ |
| `Grounding source not found` | Prompt Template のグラウンディング設定が不正 | Search Index 名とDMO名を確認 |
| `Data Graph query timeout` | Data Graph が複雑すぎる | 結合するDMOを減らす、フィルタを追加 |
| `No unified profiles found` | Identity Resolution が未実行 | 統合ジョブを実行、ルールセットを確認 |

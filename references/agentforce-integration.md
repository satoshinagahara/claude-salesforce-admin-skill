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

## 4. RAG（Retrieval-Augmented Generation）

### 4-1. RAG の概要

RAGは、エージェントがユーザーの質問に回答する際に、事前にインデックス化されたナレッジデータから関連情報を検索（Retrieval）し、その結果をLLMのプロンプトに注入（Augmentation）して回答を生成（Generation）する仕組み。

**RAGの処理フロー：**
```
[オフライン準備]
データ取り込み → チャンキング → ベクトル化（Embedding生成）→ インデックス作成

[オンライン処理（ユーザーの質問時）]
質問をベクトル化 → 類似チャンクを検索 → コンテキスト付きプロンプトを構築 → LLMが回答生成
```

### 4-2. 2つの実装アプローチ

| アプローチ | 説明 | 適用場面 |
|---|---|---|
| **Agentforce Data Library（ADL）** | UIで作成するだけで全パイプラインが自動構築される | 素早くRAGを立ち上げたい場合。Knowledge記事やPDFをベースにした標準的なRAG |
| **Advanced Data360 Setup** | データストリーム・Search Index・Retrieverを個別に設定 | 細かい制御が必要な場合。複数データソース、カスタムRetriever、フィルタリング |

### 4-3. 前提条件

1. **Data360（Data Cloud）** がorgで有効化されていること
2. **Einstein Trust Layer** が有効化されていること
3. RAG用のデータ（Knowledge記事、PDF、テキストファイル等）が用意されていること

---

## 5. Agentforce Data Library（ADL）による簡易RAG実装

### 5-1. ADLとは

Agentforce Data Library（ADL）は、RAGに必要な全コンポーネントをUIから一括で自動構築する機能。
ADLを作成すると以下が自動的にセットアップされる：

- データストリーム
- DLO / DMO とマッピング
- ベクトルデータストア
- Search Index
- Retriever
- Prompt Template
- エージェントアクション（`Answer Questions with Knowledge`）

### 5-2. ADLのデータソース種別

| ソース種別 | 説明 | サイズ制限 |
|---|---|---|
| **Knowledge Articles** | Salesforce Knowledge の記事 | フィールドを選択してチャンキング対象に指定 |
| **Uploaded Files** | PDF、HTML、テキストファイルを直接アップロード | テキスト/HTML: 4MB、PDF: 100MB |
| **Web Search** | 公開Webの検索結果を利用（Spring '25〜） | データの取り込み・インデックス作成はしない |
| **Custom Retriever** | 自作のRetrieverを指定 | Advanced Setup で事前作成が必要 |

### 5-3. ADL作成手順（Knowledge Articles）

1. Setup → Agents → 対象エージェントを選択 → Agent Builder
2. **Data Library** セクション → **New Library**
3. データソースとして **Knowledge** を選択
4. 対象のKnowledgeオブジェクトとチャンキング対象フィールドを選択
5. Save

### 5-4. ADL作成手順（ファイルアップロード）

1. Setup → Agents → 対象エージェントを選択 → Agent Builder
2. **Data Library** セクション → **New Library**
3. データソースとして **Files** を選択
4. **Upload Files** をクリックしPDF/HTML/テキストを選択
5. Save

**保存後の自動処理：**
1. データストリームが作成される
2. DLO/DMOが作成されマッピングされる
3. テキストがチャンキングされる（ファイルの数・サイズ・複雑さに応じて時間が変動）
4. チャンキング完了後、Search Indexが構築される
5. Retrieverが自動生成される
6. `Answer Questions with Knowledge` アクションが利用可能になる

### 5-5. ADLの状態確認

ADL作成後、Search Indexのステータスが `Ready` になるまで待つ必要がある：

- Setup → Data Cloud → Vector Database → Search Indexes
- ステータスが `Ready` になればRAGが利用可能

### 5-6. ADLの制限事項

- ADLごとに1つの自動生成Retrieverが作成される（ADLのgrounding source IDでプリフィルタ）
- ファイルアップロードは10MB以下でIntelligent Context（自動最適化）が適用される
- ADL作成中はData Libraryが一時的に利用不可になる

---

## 6. Advanced Data360 Setup によるRAG実装

ADLでは制御しきれない場合に、各コンポーネントを個別に設定するアプローチ。

### 6-1. Step 1: RAG用データの取り込み

#### Knowledge Articles を使う場合

CRM Connectorで自動同期：
1. Setup → Data Cloud → Data Streams → New → Salesforce CRM
2. Knowledge オブジェクトを選択
3. チャンキング対象のフィールド（記事本文等）を含める
4. 更新頻度を設定 → Save & Deploy

#### 非構造データ（PDF等）を使う場合

1. **Amazon S3 / Google Cloud Storage / Azure Blob** からのバッチ取り込み
   - Setup → Data Cloud → Data Streams → New → Cloud Storage を選択
   - 接続情報を設定、対象バケット/コンテナを指定
   - ファイルタイプマッピングを設定

2. **Ingestion API** での取り込み（`data-streams.md` 参照）

#### 対応ファイルタイプ

| タイプ | 説明 |
|---|---|
| PDF | テキストを自動抽出。画像内テキストはOCR対象外の場合あり |
| HTML | タグを解析してテキスト抽出 |
| TXT / CSV | テキストをそのまま取り込み |
| 音声 / 動画 / 画像 | Data360は対応しているが、RAGでの利用には追加処理が必要 |

### 6-2. Step 2: Search Index の作成

Search Indexは、チャンキング済みデータをベクトル化し、高速な類似検索を可能にするインデックス。

#### Search Index の種類

| タイプ | 説明 | 適用場面 |
|---|---|---|
| **Vector Search** | Embedding（ベクトル）の類似度で検索。意味的に近いチャンクを取得 | 長文の質問、一般的な情報検索 |
| **Hybrid Search** | Vector Search + Keyword Search を組み合わせ、Fusion Rankerで統合ランキング | 専門用語・固有名詞を含む質問。推奨 |

#### Search Index 作成手順（UI）

1. Setup → Data Cloud → Vector Database → **New Search Index**
2. Search Index名を入力
3. **対象DMO** を選択（チャンキング対象のテキストフィールドを含むDMO）
4. **Search Index Type** を選択（Vector or Hybrid）
5. **チャンキング対象フィールド** を選択（テキスト/リッチテキスト型フィールド）
6. **チャンキング戦略** を設定
7. Save → Build

#### チャンキング戦略

| 戦略 | 説明 | 推奨場面 |
|---|---|---|
| **Window-based** | HTMLのブロック要素（`<div>`, `<p>`）やテキストの改行で分割。HTMLがない場合は文単位 | HTML構造を持つKnowledge記事。最も汎用的 |
| **Fixed Size** | 指定したトークン数で均等分割 | テキストに構造がない場合、均一なチャンクサイズが必要な場合 |
| **None（チャンクなし）** | フィールド全体を1つのチャンクとして扱う | 短いテキスト（FAQ等）で分割が不要な場合 |

#### チャンキングの最適化ガイドライン

- **チャンクサイズが大きすぎる** → 検索精度が下がる（関係ない情報が混入）
- **チャンクサイズが小さすぎる** → 文脈が失われる（回答に必要な情報が分断）
- **推奨**: まずWindow-basedで開始し、検索結果の品質を見ながら調整
- Search Indexの **Rebuild** で設定変更を反映可能

### 6-3. Step 3: Retriever の作成

Retrieverは、Search Indexから関連チャンクを検索して返すコンポーネント。

#### Retriever の種類

| タイプ | 説明 | 適用場面 |
|---|---|---|
| **Individual Retriever** | 1つのSearch Indexに接続し、検索・フィルタ・返却チャンク数を設定 | 単一データソースのRAG |
| **Ensemble Retriever** | 複数のIndividual Retrieverを束ねて並列検索し、結果を統合ランキング | 複数データソース（Knowledge + PDF + FAQ等）を横断検索 |

#### Individual Retriever 作成手順（UI）

1. Setup → Data Cloud → Retrievers → **New Individual Retriever**
2. Retriever名を入力
3. **Data Space** を選択（通常はデフォルト）
4. **Data Model Object** を選択
5. **Search Index** を選択
6. **フィルタ** を設定（オプション。例: `Category = 'Hardware'`）
7. **返却フィールド** と **返却チャンク数** を設定
8. Save → Activate（バージョンが自動作成される）

**重要**: Retrieverは **バージョンをActivate** しないとPrompt Templateで使用できない。

#### Ensemble Retriever 作成手順（UI）

1. Setup → Data Cloud → Retrievers → **New Ensemble Retriever**
2. Ensemble Retriever名を入力
3. 含める **Individual Retriever** を選択（複数）
4. Save → Activate

**Ensemble Retriever のメリット：**
- 1つのRetrieverでPrompt Templateをグラウンディングできる（Prompt Template側の設定が簡潔に）
- 複数ソースの結果を動的にリランキング（類似度ベース）
- 無関係な結果が自動的に除外される
- Einstein Requestの消費が効率的

### 6-4. Step 4: Prompt Template の作成とグラウンディング

#### Prompt Template 作成手順（UI）

1. Setup → Einstein → **Prompt Builder** → **New Prompt Template**
2. テンプレートタイプを選択：

| タイプ | 用途 |
|---|---|
| **Flex** | 任意の入力変数を定義可能。最も汎用的 |
| **Field Generation** | 特定のオブジェクトフィールドへの出力専用 |

3. テンプレート名・説明を入力
4. **Add Resource** → **Retriever** を選択
5. 作成した Individual Retriever または Ensemble Retriever を選択
6. プロンプト本文に `{!$Retriever.RetrieverName}` マージフィールドを配置
7. 入力変数を定義（例: `customerQuestion`）
8. **Preview** でテスト → **Activate**

#### Prompt Template の構造例

```
あなたは{!$Input:companyName}のカスタマーサポートアシスタントです。

以下のナレッジベースの情報を参考に、顧客の質問に回答してください。
情報が見つからない場合は「該当する情報が見つかりませんでした」と回答してください。

--- ナレッジベース ---
{!$Retriever.KnowledgeRetriever}
--- ここまで ---

顧客の質問: {!$Input:customerQuestion}

回答:
```

#### グラウンディングのデータソース選択肢

| ソース | マージフィールド | 説明 |
|---|---|---|
| **Retriever（推奨）** | `{!$Retriever.RetrieverName}` | Data Cloud Search Index経由。RAGの標準的な方法 |
| **Related List** | `{!$RelatedList.ObjectName.FieldName}` | CRMの関連リストデータを直接注入 |
| **Flow** | `{!$Flow.FlowName}` | Flowで取得したデータを注入 |
| **Apex** | `{!$Apex.ClassName}` | Apexで取得したデータを注入 |

### 6-5. Step 5: Agentforce アクションとして登録

Prompt Templateをエージェントのアクションとして使う方法は `metadata-agentforce.md` セクション9を参照。

要約：
1. Setup → **Agent Actions** → New Agent Action
2. Reference Action Type: **Prompt Template** → 作成したテンプレートを選択
3. Action Instructions（エージェントがこのアクションを使う条件）を記述
4. Agent Builder でトピックにアクションを追加

### 6-6. GenAiPromptTemplate メタデータ

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

## 7. Search Index の管理

### 7-1. Search Index の状態確認

UI: Setup → Data Cloud → Vector Database → Search Indexes

ステータス：
| ステータス | 意味 |
|---|---|
| **Building** | インデックス構築中。完了まで待つ |
| **Ready** | 利用可能 |
| **Failed** | 構築失敗。設定を確認して再ビルド |

### 7-2. Search Index の再ビルド

チャンキング戦略やフィールドを変更した場合は再ビルドが必要：

UI: Setup → Data Cloud → Vector Database → 対象Search Index → **Rebuild**

**自動リビルド**: Data Cloudはデータ変更時に増分インデックス更新を行う。手動リビルドは設定変更時のみ必要。

### 7-3. CLI/API での Search Index 操作

Search Indexの作成・リビルドは主にUIで行う。API経由での操作は限定的：

```bash
# Data Cloud 関連メタデータの確認
sf api request rest \
  --method GET \
  --url "/services/data/v62.0/ssot/" \
  --target-org <username>
```

**注意**: Search Index の作成・設定変更・リビルドのREST APIは公開ドキュメントが限定的。UIでの操作を推奨。

---

## 8. RAG の評価とチューニング

### 8-1. 検索品質の確認方法

1. **Prompt Builder の Preview 機能**: テスト入力でRetrieverが返すチャンクを確認
2. **Agent Builder の Preview**: エージェントに質問して回答品質を確認
3. **Agent Test**: `sf agent test run` で自動テストを実行

### 8-2. チューニング項目

| 項目 | 調整方法 | 影響 |
|---|---|---|
| **チャンキング戦略** | Search Indexの設定変更 → Rebuild | チャンクの粒度・文脈の保持 |
| **Search Index タイプ** | Vector → Hybrid に変更 | 専門用語の検索精度向上 |
| **返却チャンク数** | Retrieverの設定変更 | 多い＝文脈が豊富だがノイズ増。少ない＝精度高いが漏れリスク |
| **フィルタ条件** | Retrieverにフィルタ追加 | 不要なカテゴリのデータを除外 |
| **Prompt Template** | プロンプト文の改善 | LLMの回答品質・フォーマット |
| **Ensemble Retriever** | 複数Retrieverの組み合わせ | 複数ソースの横断検索、自動リランキング |

### 8-3. RAG のベストプラクティス

- **まずADLで始める** → 動作確認後、必要に応じてAdvanced Setupに移行
- **Hybrid Search を優先** → 専門用語・固有名詞への対応力が高い
- **チャンクサイズは中程度から** → 大きすぎ/小さすぎを避け、結果を見て調整
- **Retrieverのフィルタを活用** → 関連性の低いデータを除外して精度向上
- **Ensemble Retrieverで複数ソースを統合** → Knowledge + PDF + FAQ等を1つのRetrieverにまとめる
- **Prompt Templateで「情報がない場合」の指示を明記** → ハルシネーション防止

---

## 9. 全体実装フロー

### 9-1. ADL方式（推奨・簡易）

```
1. Data360（Data Cloud）が有効化されていることを確認
2. Knowledge記事を作成 or PDFファイルを準備
3. Agent Builder → Data Library → New Library で ADL を作成
4. Search Index が Ready になるのを待つ
5. Answer Questions with Knowledge アクションが自動追加される
6. エージェントをテスト → 有効化
```

### 9-2. Advanced Setup 方式（細かい制御が必要な場合）

```
1. Data Cloud にデータを取り込む（データストリーム）         → data-streams.md
2. DLO → DMO にマッピング                                   → data-model.md
3. (オプション) Identity Resolution で名寄せ                 → data-model.md
4. (オプション) Data Graph で複数DMOを結合                   → 本ファイル セクション3
5. Search Index を作成（Vector or Hybrid）                   → 本ファイル セクション6-2
6. Retriever を作成（Individual or Ensemble）                → 本ファイル セクション6-3
7. Prompt Template を作成しRetrieverでグラウンディング        → 本ファイル セクション6-4
8. Agent Action として登録                                    → 本ファイル セクション6-5
9. Agent Builder でトピックにアクション追加                   → metadata-agentforce.md
```

---

## 10. Data Cloud 権限の設定

### 10-1. 必要な権限セット

| 権限セット | 用途 |
|---|---|
| **Data Cloud Admin** | Data Cloud の管理操作（ストリーム・モデル・ルール設定） |
| **Data Cloud User** | Data Cloud データの参照 |
| **Data Cloud for Agents** | Agentforce が Data Cloud にアクセスするための権限 |

### 10-2. Agent ユーザーへの権限付与

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

## 11. よくあるエラーと対処

| エラー | 原因 | 対処 |
|---|---|---|
| `DMO not queryable from Agent` | Agent ユーザーに Data Cloud 権限がない | Data Cloud for Agents 権限セットを割り当て |
| `Search Index not ready` | ベクトルインデックスのビルドが未完了 | UI で Build ステータスを確認、完了を待つ |
| `Grounding source not found` | Prompt Template のグラウンディング設定が不正 | Search Index 名とDMO名を確認 |
| `Retriever not activated` | Retrieverのバージョンが未Activate | Einstein Studio でRetrieverバージョンをActivate |
| `Data Graph query timeout` | Data Graph が複雑すぎる | 結合するDMOを減らす、フィルタを追加 |
| `No unified profiles found` | Identity Resolution が未実行 | 統合ジョブを実行、ルールセットを確認 |
| `Data Library temporarily unavailable` | ADLのインデックス構築中 | 構築完了まで待つ（ファイル数・サイズに依存） |
| `File exceeds size limit` | アップロードファイルが制限超過 | テキスト/HTML: 4MB以下、PDF: 100MB以下に分割 |
| チャンク検索結果が不正確 | チャンキング戦略が不適切 | Hybrid Searchに変更、チャンクサイズを調整、Rebuild |

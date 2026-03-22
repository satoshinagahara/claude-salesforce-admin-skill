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

#### CRM Connector（Salesforceオブジェクト）を使う場合（実証済み）

1. **Data Cloud アプリ → データストリーム → 新規** → Salesforce CRM を選択
2. 対象オブジェクトを選択し、チャンキング対象のテキストフィールドを含める
3. **オブジェクトカテゴリの選択が必須**（選択しないとエラー）：

| カテゴリ | 用途 |
|---|---|
| プロファイル | 個人・取引先など主体データ |
| エンゲージメント | 行動・イベントデータ |
| **その他** | 上記に当てはまらないデータ（カスタムオブジェクトは通常これ） |

4. プライマリキーはデフォルト（カスタムオブジェクト ID）のままでOK
5. 更新頻度を設定 → **リリース**
6. リリース後、データストリーム詳細画面で **「今すぐ更新」** をクリックして初回同期を実行

**DLO→DMOマッピング（データストリーム作成後）：**
1. データストリーム詳細画面 → 右側の「データマッピング」→ **「開始」**
2. 右側の **「オブジェクトの選択」** → **「カスタムデータモデル」** タブ → **「+ 新規カスタムオブジェクト」**
3. オブジェクト名・API参照名を入力、カテゴリ「その他」→ 全フィールドを選択 → **保存**
4. Einsteinが自動でDLO→DMOのフィールドマッピングを生成する
5. マッピング内容を確認 → **「保存して閉じる」**

#### Knowledge Articles を使う場合

CRM Connectorで自動同期（上記と同じ手順。Knowledgeオブジェクトを選択）

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

#### Search Index 作成手順（UI・実証済み）

**場所: Data Cloud アプリ → 検索インデックス タブ → 新規**（Setup配下ではない）

1. **「高度な設定」** を選択（チャンキングやエンベディングモデルを細かく制御）
2. Search Index名・API参照名を入力
3. **対象DMO** を選択（チャンキング対象のテキストフィールドを含むDMO）
4. **Search Index Type**: **Hybrid** を推奨
5. **チャンキング対象フィールド** を選択（テキスト/リッチテキスト型フィールド）
6. **チャンキング戦略**: Passage Extraction（デフォルト）で開始
7. **チャンクを強化**: テスト段階ではオフ推奨（後から再ビルドで有効化可能）
8. **エンベディングモデル**:

| モデル | 用途 |
|---|---|
| E5 Large V2 Embedding Model | 英語特化 |
| **Multilingual E5 Large Embedding Model** | **日本語を含む多言語対応。日本語データはこちらを選択** |
| Multilingual CLIP Embedding Model | 画像+テキストのマルチモーダル |
| Salesforce Embedding V2 Small | Salesforce独自の軽量モデル |

9. **検索条件の関連項目**: Retrieverが返す追加フィールドを選択（タイトル、ステータス等の属性フィールド）
10. **ランキング**: テスト段階では追加不要
11. Save → Build

**Search Index のステータス遷移:**
作成 → 送信済み → **準備完了**（利用可能）。66レコード程度で約15〜20分。
**準備完了になるまでRetrieverは作成できない**（DMOが選択肢に表示されない）。

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

#### Individual Retriever 作成手順（UI・実証済み）

**場所: Setup → Einstein → Einstein Studio → Retrievers**（Data Cloud配下にはない）

1. **New → 個別レトリーバー** を選択
2. データソース: **Data Cloud** を選択
3. **Data Space**: default
4. **Data Model Object**: 対象DMOを選択（Search Indexが「準備完了」でないと表示されない）
5. **検索インデックス設定**: 作成したSearch Indexを選択
6. **検索条件**: オプション（フィルタが必要な場合のみ）
7. **返却フィールドの設定（重要）**:
   - 最低1つの項目定義が必須（空だとエラー）
   - Chunkテキスト自体は自動で返されるため、ここでは**DMOの属性フィールド**を指定
   - 例: タイトル、ステータス、優先度、ニーズ種別、取引先など
   - 項目名のドロップダウン → 「直接属性」→ フィールドを選択
8. **返却チャンク数**: デフォルト20（10〜20が実用的）
9. **引用設定**: テスト段階ではオフ推奨
10. Save → **有効化ボタンをクリック**（バージョンがActivateされる）

**重要**: Retrieverは **バージョンをActivate（有効化）** しないとPrompt Templateで使用できない。有効化後は即利用可能。

#### Ensemble Retriever 作成手順（UI）

1. Setup → Einstein → Einstein Studio → Retrievers → **New → アンサンブルレトリーバー**
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

### 6-6. GenAiPromptTemplate メタデータ（Retrieverグラウンディング付き・実証済み）

**詳細な構造とデプロイ注意事項は `prompt-templates.md` セクション7-1b を参照。**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<GenAiPromptTemplate xmlns="http://soap.sforce.com/2006/04/metadata">
    <developerName>CustomerSupportRAG</developerName>
    <masterLabel>Customer Support RAG</masterLabel>
    <templateVersions>
        <content>以下のナレッジベースを参考に回答してください。

--- ナレッジベース ---
{!$EinsteinSearch:MyRetrieverApiName.results}
--- ここまで ---

質問: {!$Input:customerQuestion}
回答:</content>
        <inputs>
            <apiName>customerQuestion</apiName>
            <definition>primitive://String</definition>
            <masterLabel>customerQuestion</masterLabel>
            <referenceName>Input:customerQuestion</referenceName>
            <required>true</required>
        </inputs>
        <primaryModel>sfdc_ai__DefaultOpenAIGPT4OmniMini</primaryModel>
        <status>Published</status>
        <templateDataProviders>
            <definition>invocable://getEinsteinRetrieverResults/MyRetrieverApiName</definition>
            <description>MyRetriever</description>
            <label>MyRetriever</label>
            <parameters>
                <definition>primitive://String</definition>
                <isRequired>true</isRequired>
                <parameterName>searchText</parameterName>
                <valueExpression>{!$Input:customerQuestion}</valueExpression>
            </parameters>
            <referenceName>EinsteinSearch:MyRetrieverApiName</referenceName>
        </templateDataProviders>
    </templateVersions>
    <type>einstein_gpt__flex</type>
    <visibility>Global</visibility>
</GenAiPromptTemplate>
```

**注意:**
- Retriever APIの参照名は Einstein Studio → Retrievers で確認（Data Cloud配下にはない）
- マージフィールドは `{!$EinsteinSearch:xxx.results}`（UIでは `{!$Retriever.xxx}` と表示される）
- デプロイ後はPrompt Builder UIで手動Activateが必要

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

---

## 12. RAGの適用判断ガイドライン（実証済み）

RAG（Retriever）はセマンティック検索には強いが、全件分析・集計には不向き。要件に応じて使い分ける。

### 判断フロー

```
「LLMに渡すデータをどう選ぶか？」

全件渡せるサイズ（〜数百件）？ → Data Cloud queryv2で全件取得 → Prompt Templateに直接渡す
  ↓ No
構造化条件でフィルタ可能？ → queryv2でフィルタ → Prompt Templateに直接渡す
  ↓ No
自由テキストの意味検索が必要？ → RAG（Retriever + Search Index）を使う
```

### パターン別の使い分け

| 要件 | 方式 | 理由 |
|---|---|---|
| 全件の傾向分析・集計 | queryv2直接 | 少数派の意見（ネガティブ等）が漏れない |
| 特定条件でのフィルタ分析 | queryv2直接 | 構造化フィールド（会社名、日付等）での正確なフィルタが可能 |
| 対話型の質問応答 | RAG（Retriever） | 「この顧客について教えて」のような自由な質問にはセマンティック検索が適切 |
| 大量データからの関連情報抽出 | RAG（Retriever） | 数千件のナレッジから関連記事を探す等 |

### RAGが不適切なケースの具体例

- **アンケート全件の傾向分析**: Retrieverはチャンク単位の類似度検索なので、メインテーマと類似度が低い意見（「UIがわかりにくい」「回線が途切れた」等）が返されにくい
- **取引先単位のフィルタリング**: company_nameはチャンキング対象フィールドに含まれないため、Retrieverで特定企業の回答だけを取得できない
- **少数派の意見を確実に拾いたい場合**: Retrieverは関連度の高いチャンクを優先するため、少数意見が漏れるリスクがある

### Data Cloud queryv2をApexから呼ぶ場合の注意

- エンドポイント: `POST /services/data/v62.0/ssot/queryv2`
- リクエストボディ: `{"sql":"SELECT ... FROM <DMO名>__dlm WHERE ..."}`
- **HTTPステータスコードは200ではなく201（Created）**
- Apexの制約: DML実行後はHTTP calloutが使えない。calloutを先に全て実行し、DMLをまとめて後から実行する設計が必要

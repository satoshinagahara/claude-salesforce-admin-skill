# Prompt Template (Prompt Builder) Reference

## 概要
Prompt Templateは、Salesforceの生成AI機能をワークフローに統合するための構造化されたプロンプト定義。
Prompt Builder（Einstein 1 Studio）で作成・管理し、**Flow・Apex・REST API・Agentforceアクション**など複数のコンテキストから呼び出せる。

---

## 1. テンプレートタイプと呼び出し元

| テンプレートタイプ | 説明 | 呼び出し元 |
|---|---|---|
| **Field Generation** | レコードの特定フィールドに LLM の生成結果を書き込む | レコードページ, Flow, Apex, REST API, Agentforce |
| **Flex** | 任意の入力変数を定義可能（最大5つ）。最も汎用的 | Flow, Apex, REST API, Agentforce |
| **Record Summary** | レコードの要約を生成 | Agentforce |
| **Sales Email** | 営業メールを生成 | メールコンポーザー, Flow, Apex, REST API, Agentforce |

**Flex テンプレートが最も汎用的で、多くのユースケースに対応可能。**

---

## 2. Prompt Template の作成（UI）

### 2-1. 前提条件

- Einstein Generative AI が有効化されていること（Setup → Einstein Setup）
- **Prompt Template Manager** 権限セットがユーザーに割り当て済みであること

### 2-2. 作成手順

1. Setup → Quick Find → **Prompt Builder**
2. **New Prompt Template**
3. テンプレートタイプを選択（Field Generation / Flex / Record Summary / Sales Email）
4. テンプレート名・API名・説明を入力
5. **入力リソース**を定義（関連オブジェクト、テキスト入力等）
6. プロンプト本文を作成
7. **Preview** でテスト → **Activate**

### 2-3. プロンプト内で使えるデータソース（グラウンディング）

| ソース | マージフィールド例 | 説明 |
|---|---|---|
| **レコードフィールド** | `{!$Input:myCase.Subject}` | 入力レコードのフィールド値 |
| **関連リスト** | `{!$RelatedList:myCase.CaseComments.Records}` | 関連レコードのデータ |
| **Flow** | `{!$Flow.FlowName}` | Flowで取得したデータを注入 |
| **Apex** | `{!$Apex.ClassName}` | Apexで取得したデータを注入 |
| **Retriever** | `{!$Retriever.RetrieverName}` | Data Cloud Search Index経由（RAG） |

---

## 3. Flow からの呼び出し

### 3-1. 基本: Activate済みテンプレートは自動的にFlowのInvocable Actionになる

```
Flow → Actions要素 → カテゴリ「Prompt Template」でフィルタ → テンプレートを選択
```

入力: テンプレートが期待するレコード（関連エンティティ）を渡す
出力: `promptResponse`（LLMの生成テキスト）

### 3-2. パターン例: ケースの自動分類・要約（Record-Triggered Flow）

```
[ケース作成/更新]
    ↓ Record-Triggered Flow
[Action: Prompt Template 呼び出し]
    → 入力: {!$Record}（トリガーレコード）
    → 出力: promptResponse（JSON文字列）
    ↓
[Action: Apex パーサー呼び出し]
    → JSON を構造化データにパース
    ↓
[Update Records]
    → パース結果でケースのType, Reason, Summary を更新
```

**Prompt Template 本文の例（JSON出力指定）:**
```
You are an API service. Categorize and summarize the case below.

Case Type options: Electrical, Mechanical, Electronic, Other
Case Reason options: Installation, Performance, Breakdown, Other

Priority: {!$Input:myCase.Priority}
Subject: {!$Input:myCase.Subject}
Description: {!$Input:myCase.Description}
Case Comments: {!$RelatedList:myCase.CaseComments.Records}

Output as valid JSON with keys "caseType", "reason", "summary".
```

**JSON パーサーの Apex 例:**
```apex
public class CaseSummarizationParser {
    @InvocableMethod(label='Parse Case Summary'
        description='Parses JSON case summary from prompt template')
    public static List<CaseSummary> parseCaseSummary(List<String> promptResponses) {
        List<CaseSummary> results = new List<CaseSummary>();
        for (String response : promptResponses) {
            Map<String, Object> parsed =
                (Map<String, Object>) JSON.deserializeUntyped(response);
            CaseSummary s = new CaseSummary();
            s.caseType = (String) parsed.get('caseType');
            s.reason = (String) parsed.get('reason');
            s.summary = (String) parsed.get('summary');
            results.add(s);
        }
        return results;
    }

    public class CaseSummary {
        @InvocableVariable public String caseType;
        @InvocableVariable public String reason;
        @InvocableVariable public String summary;
    }
}
```

### 3-3. パターン例: レコード変更の自動サマリー（Flex テンプレート）

```
[レコード更新]
    ↓ Record-Triggered Flow
[Apex: 変更前レコードをJSON変換]
[Apex: 変更後レコードをJSON変換]
    ↓
[Action: Flex Prompt Template 呼び出し]
    → 入力: Original Record JSON, Updated Record JSON, User
    → 出力: 変更サマリーテキスト
    ↓
[Update Records]
    → サマリーをカスタムフィールドに保存
```

### 3-4. Template-Triggered Prompt Flow

**Flow内で複雑なロジックを実行してからLLMに渡す**パターン。
テンプレートのグラウンディングソースとしてFlowを使い、Flow内で条件分岐・データ取得・加工を行った結果をプロンプトに注入する。

---

## 4. Apex からの呼び出し

### 4-1. Connect API を使用

```apex
// 入力の作成
ConnectApi.EinsteinPromptTemplateGenerationsInput input =
    new ConnectApi.EinsteinPromptTemplateGenerationsInput();
input.isPreview = false;  // true: 解決済みプロンプトのみ, false: LLM応答も取得
input.additionalConfig = new ConnectApi.EinsteinLlmAdditionalConfigInput();
input.additionalConfig.numGenerations = 1;
input.additionalConfig.temperature = 0;  // 0-1, 低いほど決定論的

// 入力変数の設定
Map<String, ConnectApi.WrappedValue> inputParams =
    new Map<String, ConnectApi.WrappedValue>();
inputParams.put('Input:myCase', // テンプレートの入力変数名
    ConnectApi.WrappedValue.toWrappedValue(caseRecord));
input.inputParams = inputParams;

// 呼び出し
ConnectApi.EinsteinPromptTemplateGenerationsRepresentation result =
    ConnectApi.EinsteinLLM.generateMessagesForPromptTemplate(
        'Summarize_Classify_Cases', // テンプレートAPI名
        input
    );

// 結果の取得
String responseText = result.generations[0].text;
```

### 4-2. パラメータ

| パラメータ | 説明 | デフォルト |
|---|---|---|
| `isPreview` | true: 解決済みプロンプトのみ返す / false: LLM応答も返す | false |
| `numGenerations` | 生成する応答の数 | 1 |
| `temperature` | 応答のランダム性（0=決定論的, 1=創造的） | 0 |
| `frequencyPenalty` | トークンの繰り返し抑制 | - |

### 4-3. ベストプラクティス

- **JSON出力を指示**してApexでパースしやすくする
- LWCから呼び出す場合もApexコントローラー経由で Connect API を使用
- バッチ処理には **Prompt Template Batch Processing** を使用（非同期・大量処理対応）

---

## 5. REST API からの呼び出し

```bash
# Connect REST API エンドポイント
sf api request rest \
  --method POST \
  --url "/services/data/v66.0/einstein/prompt-templates/<templateApiName>/generations" \
  --body '{
    "isPreview": false,
    "inputParams": {
      "Input:myCase": {
        "value": {"id": "500XXXXXXXXXXXX"},
        "valueType": "SOBJECT_ROW_INPUT"
      }
    },
    "additionalConfig": {
      "numGenerations": 1,
      "temperature": 0
    }
  }' \
  --target-org <username>
```

外部システムからSalesforceのPrompt Templateを呼び出す場合に使用。

---

## 6. Agentforce アクションとしての利用

Agentforce エージェントのアクションとしてPrompt Templateを使う方法は `metadata-agentforce.md` セクション9を参照。

要約:
1. Setup → **Agent Actions** → New Agent Action
2. Reference Action Type: **Prompt Template** → テンプレートを選択
3. Action Instructions（エージェントがこのアクションを使う条件）を記述
4. Agent Builder でトピックにアクションを追加

---

## 7. メタデータ構造

### 7-1. GenAiPromptTemplate

```xml
<?xml version="1.0" encoding="UTF-8"?>
<GenAiPromptTemplate xmlns="http://soap.sforce.com/2006/04/metadata">
    <activeVersionNumber>1</activeVersionNumber>
    <description>テンプレートの説明</description>
    <masterLabel>テンプレート表示名</masterLabel>
    <templateVersions>
        <content>プロンプト本文（マージフィールド含む）</content>
        <inputs>
            <apiName>myCase</apiName>
            <dataType>SObject</dataType>
            <definition>Case</definition>
        </inputs>
        <primaryModel>sfdc_ai__DefaultGPT4Omni</primaryModel>
        <status>Published</status>
        <versionNumber>1</versionNumber>
    </templateVersions>
    <type>flex</type>
</GenAiPromptTemplate>
```

### 7-2. Retrieve / Deploy

```bash
# Retrieve
sf project retrieve start \
  --metadata "GenAiPromptTemplate:<TemplateName>" \
  --target-org <username>

# Deploy
sf project deploy start \
  --metadata "GenAiPromptTemplate:<TemplateName>" \
  --target-org <username>
```

### 7-3. テンプレート一覧の確認

```bash
sf org list metadata \
  --metadata-type GenAiPromptTemplate \
  --target-org <username>
```

---

## 8. ユースケース一覧

| ユースケース | テンプレートタイプ | 呼び出し元 | 説明 |
|---|---|---|---|
| ケース自動分類・要約 | Flex | Record-Triggered Flow | ケース作成時にLLMで分類・要約しフィールド更新 |
| レコード変更サマリー | Flex | Record-Triggered Flow | 変更前後のJSON比較でサマリー自動生成 |
| フィールド自動生成 | Field Generation | レコードページ / Flow | 特定フィールドにLLM生成値を自動入力 |
| 営業メール生成 | Sales Email | メールコンポーザー / Flow | 商談情報をもとにパーソナライズされたメール生成 |
| ナレッジ検索回答（RAG） | Flex | Agentforce | Retrieverでグラウンディングして質問に回答 |
| リード自動スコアリング | Flex | Flow / Apex | リード情報をLLMで分析してスコア・分類 |
| 取引先サマリー生成 | Flex | Agentforce / Apex | 取引先の関連データを集約して要約 |

---

## 9. プロンプト設計のベストプラクティス

| 観点 | ガイドライン |
|---|---|
| **構造化出力** | JSON出力を指示し、キー名を明示する。パースが容易になる |
| **選択肢の制約** | 分類タスクでは有効な値を列挙してLLMの出力を制約する |
| **例示** | Few-shot（出力例を1-2個提示）で精度を向上させる |
| **言語指定** | 出力言語を明示する（「日本語で回答してください」等） |
| **ハルシネーション防止** | 「情報がない場合は『情報なし』と回答してください」と明記 |
| **トークン効率** | 不要なデータを渡さない。必要なフィールドのみマージフィールドで参照する |

---

## 10. よくあるエラーと対処

| エラー | 原因 | 対処 |
|---|---|---|
| テンプレートがFlowのAction一覧に表示されない | テンプレートが未Activate | Prompt Builder で Activate する |
| `Prompt template not found` | API名の誤り / 未デプロイ | `sf org list metadata --metadata-type GenAiPromptTemplate` で確認 |
| `Einstein Generative AI is not enabled` | Einstein未有効化 | Setup → Einstein Setup で有効化 |
| LLM応答が空 / エラー | Einstein Trust Layer でブロック | プロンプト内容を調整（有害コンテンツ判定を回避） |
| JSON出力が不正 | LLMが指示通りのJSONを返さない | プロンプトに出力例を追加、temperature を 0 に設定 |
| `Insufficient privileges` | 権限不足 | Prompt Template Manager 権限セットを割り当て |
| Flowでの依存デプロイエラー | Apex → Flow → Template の依存順序 | Apex → Flow → GenAiPromptTemplate の順でデプロイ |

# Metadata: Agentforce (GenAI Agent) Reference

## 概要
Agentforce AgentのSF CLIによる作成・設定・デプロイパターン。
APIバージョン60以上が必要。GenAiPlannerBundleはv64以上、AiAuthoringBundleはv65以上。

---

## 0. Agentforce エージェント種別（重要）

### 0-1. BotDefinition.Type の値と意味

| Type 値 | 現名称 | 用途 | 主なユーザー |
|---|---|---|---|
| `Bot` | Einstein Bot | 従来型チャットボット（ダイアログベース） | 外部顧客 |
| `ExternalCopilot` | Agentforce Service Agent | LLM推論ベースの顧客向けAIエージェント | 外部顧客 |
| `InternalCopilot` | Agentforce (Default) / Employee Agent | Salesforce CRM内で社内ユーザーが使うAIアシスタント | 社内ユーザー |

### 0-2. BotDefinition.AgentType の値

| AgentType 値 | 対応する Type | 説明 |
|---|---|---|
| null | `Bot` | 従来型Einstein Bot |
| `EinsteinServiceAgent` | `ExternalCopilot` | Agentforce Service Agent（顧客向け） |
| `EinsteinCopilot`（UI自動設定） | `InternalCopilot` | Employee Agent（社内向け） |

### 0-3. 種別ごとの利用シーン比較

| 種別 | 配置場所 | アクセス | ライセンス |
|---|---|---|---|
| ExternalCopilot | 外部サイト・メッセージング | 外部顧客 | Agentforce for Service |
| InternalCopilot | Salesforce Lightning UI（コパイロットパネル） | Salesforceログイン済みユーザー | Agentforce for Employees |
| Bot | Embedded Messaging等 | 外部顧客（従来型） | Einstein Bot |

### 0-4. CLI・API の制限

| 操作 | ExternalCopilot | InternalCopilot |
|---|---|---|
| `sf agent create` で作成 | OK | 不可（常にExternalCopilotが生成される） |
| Metadata API でBot作成 | OK | 不可 |
| `BotDefinition.Type` を変更 | 不可（書き込み不可フィールド） | 不可 |
| トピック/アクションをMetadata APIで追加 | OK（genAiPlugins/genAiFunctions） | 不可（デプロイ Succeeded でも無言で無視される） |
| トピック/アクションをUI Agent Builderで追加 | OK | OK |

### 0-5. Employee Agent（InternalCopilot）の作成方法

Employee Agent は CLI から作成できないため、**Salesforceセットアップ画面から作成**する：

1. Setup → Agents → **New** ボタン
2. エージェント種別で **"Employee Agent"** を選択
3. 名前・説明を入力して保存
4. Agent Builder で トピック・アクションを設定
5. 作成後は `sf project retrieve start --metadata "Bot:<APIName>"` で取得可能

作成後のメタデータ：
- `Bot.type = InternalCopilot`
- `Bot.agentType = EinsteinCopilot`（自動設定）
- `GenAiPlannerBundle.plannerSurfaces` に Employee 向けの surface（UI自動設定）

### 0-6. InternalCopilot の正しい実装フロー

```
1. Apex クラス作成（@InvocableMethod, @InvocableVariable）→ sf project deploy start
2. UIでEmployee Agentを新規作成（Setup → Agents → New → "Employee Agent"）
3. UIのAgent Builderでトピックを作成（Topics タブ → + New Topic）
4. UIでアクションを追加（+ Add from Asset Library → Apex アクション選択）
5. Activate
6. Metadata リトリーブでローカル保存
   sf project retrieve start --metadata "GenAiPlannerBundle:<APIName>"
```

---

## 1. メタデータ型と依存関係

### 1-1. 主要メタデータ型

| メタデータ型 | 説明 | API版 |
|---|---|---|
| **Bot** | Agentの最上位定義 | 60+ |
| **BotVersion** | Agentの特定バージョン設定 | 60+ |
| **GenAiPlannerBundle** | Agentの推論エンジン（LLM・推論戦略） | 64+ |
| **GenAiPlanner** | 推論エンジン（旧形式） | 60-63 |
| **GenAiPlugin** | Agentトピック（ジョブカテゴリ） | 60+ |
| **GenAiFunction** | Agentアクション（個別の実行可能処理） | 60+ |
| **GenAiPromptTemplate** | プロンプトテンプレート | 60+ |
| **AiAuthoringBundle** | Agentスクリプト言語定義 | 65+ |

### 1-2. 依存関係とデプロイ順序

```
GenAiPromptTemplate
    ↓
GenAiFunction（アクション）  ← Flow/Apex等の invocationTarget が先に必要
    ↓
GenAiPlugin（トピック）       ← 参照するGenAiFunctionが先に必要
    ↓
GenAiPlannerBundle            ← 参照するGenAiPlugin/GenAiFunctionが先に必要
    ↓
Bot + BotVersion              ← GenAiPlannerBundleが先に必要
```

**必ず依存先を先にデプロイするか、同時にデプロイすること。**

---

## 2. SF CLI 専用コマンド（sf agent）

### 2-1. Agent仕様ファイルの生成

```bash
sf agent generate agent-spec \
  --type customer \          # customer（外部向け） または internal（内部向け、LLMトピック生成用）
  --company-name "株式会社サンプル" \
  --role "役割の説明" \
  --target-org <username>
```

出力ファイル: `specs/agentSpec.yaml`

### 2-2. agentSpec.yaml の構造

```yaml
agentType: customer           # customer または internal（CLIでは常にExternalCopilotが作成される）
companyName: 会社名
companyDescription: |
  会社の説明文
role: |
  Agentの役割説明
maxNumOfTopics: 5
enrichLogs: false
tone: casual                  # casual / formal / neutral
topics:
  - name: トピック名（英語推奨）
    description: トピックの説明
```

### 2-3. Agentの作成（Service Agent / ExternalCopilot のみ）

```bash
# 注意: 常にExternalCopilot（Service Agent）が作成される
sf agent create \
  --spec specs/agentSpec.yaml \
  --name "Agent表示名" \
  --api-name AgentAPIName \
  --target-org <username>
```

### 2-4. Authoring Bundle 方式（推奨・新方式）

```bash
# Agent Script ファイル(.agent)を生成
sf agent generate authoring-bundle \
  --spec specs/agentSpec.yaml \
  --name "Agent表示名" \
  --api-name AgentAPIName \
  --target-org <username>

# Agentとして公開（注意: こちらも常にExternalCopilotが作成される）
sf agent publish authoring-bundle \
  --api-name AgentAPIName \
  --target-org <username>
```

### 2-5. Agentをブラウザで確認

```bash
sf org open agent \
  --api-name <AgentAPIName> \
  --target-org <username>
```

### 2-6. Agentの有効化・無効化

```bash
sf agent activate --api-name <AgentAPIName> --target-org <username>
sf agent deactivate --api-name <AgentAPIName> --target-org <username>
```

---

## 3. 既存Agentのメタデータ取得（Retrieve）

### 3-1. package.xml を使って全Agent関連メタデータを取得

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>*</members>
        <name>Bot</name>
    </types>
    <types>
        <members>*</members>
        <name>BotVersion</name>
    </types>
    <types>
        <members>*</members>
        <name>GenAiPlannerBundle</name>
    </types>
    <types>
        <members>*</members>
        <name>GenAiPlugin</name>
    </types>
    <types>
        <members>*</members>
        <name>GenAiFunction</name>
    </types>
    <version>65.0</version>
</Package>
```

```bash
sf project retrieve start \
  --manifest package.xml \
  --target-org <username>
```

---

## 4. ディレクトリ構造

```
force-app/main/default/
├── bots/
│   └── <BotAPIName>/
│       ├── <BotAPIName>.bot-meta.xml
│       └── botVersions/
│           └── v1.botVersion-meta.xml       # 取得時は <fullName>v1</fullName> のみ
├── genAiPlannerBundles/
│   └── <PlannerName>/
│       ├── <PlannerName>.genAiPlannerBundle  # 拡張子に -meta.xml は不要
│       ├── plannerActions/                   # InternalCopilot のトップレベルアクション
│       │   └── <actionFullName>/
│       │       ├── input/schema.json
│       │       └── output/schema.json
│       └── localActions/                     # InternalCopilot のトピック内アクション
│           └── <topicFullName>/
│               └── <actionFullName>/
│                   ├── input/schema.json
│                   └── output/schema.json
├── genAiPlugins/
│   └── <TopicAPIName>.genAiPlugin-meta.xml
├── genAiFunctions/
│   └── <ActionAPIName>/
│       ├── <ActionAPIName>.genAiFunction-meta.xml
│       ├── input/
│       │   └── schema.json
│       └── output/
│           └── schema.json
└── aiAuthoringBundles/                       # sf agent generate authoring-bundle で生成
    └── <BundleName>/
        ├── <BundleName>.bundle-meta.xml
        └── <BundleName>.agent               # Agent Script ファイル
```

---

## 5. Bot メタデータ構造

### 5-1. ExternalCopilot（Service Agent）の bot-meta.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Bot xmlns="http://soap.sforce.com/2006/04/metadata">
    <agentDSLEnabled>false</agentDSLEnabled>
    <agentTemplate>AiCopilot__AgentforceAgent</agentTemplate>
    <agentType>EinsteinServiceAgent</agentType>
    <botMlDomain>
        <label>エージェント名</label>
        <name>AgentAPIName</name>
    </botMlDomain>
    <botSource>None</botSource>
    <botUser>agentuser@orgid.ext</botUser>
    <description>エージェントの説明</description>
    <label>表示名</label>
    <logPrivateConversationData>false</logPrivateConversationData>
    <richContentEnabled>true</richContentEnabled>
    <sessionTimeout>0</sessionTimeout>
    <type>ExternalCopilot</type>
</Bot>
```

### 5-2. BotVersion の制限

- Agentforce エージェントの BotVersion をリトリーブすると `<fullName>v1</fullName>` のみ返る
- 実際のロジックはプラットフォーム内部で管理
- この最小XMLを再デプロイすると `Required field is missing: entryDialog` エラー
- **BotVersion は Metadata API で直接編集できない**

---

## 6. GenAiFunction（アクション）の作成

### 6-1. Apex @InvocableMethod をアクションとして登録

```xml
<?xml version="1.0" encoding="UTF-8"?>
<GenAiFunction xmlns="http://soap.sforce.com/2006/04/metadata">
    <description>アクションの説明（LLMへのヒント）</description>
    <invocationTarget>MyApexClassName</invocationTarget>
    <invocationTargetType>apex</invocationTargetType>
    <masterLabel>アクション表示名</masterLabel>
</GenAiFunction>
```

**重要**: `invocationTarget` には **Apex クラスの API名（DeveloperName）** を指定する。
~~クラスID（`01p...`）を指定するとデプロイ時に `InvocableTarget not found` エラーになる。~~

### 6-2. invocationTargetType の値一覧

| 値 | 対象 |
|---|---|
| `apex` | Apex @InvocableMethod |
| `flow` | 自動起動フロー（Auto-launched Flow） |
| `promptTemplate` | Prompt Builder テンプレート |
| `standardInvocableAction` | Salesforce 標準 Invocable Action |
| `externalService` | 外部サービス（External Credentials） |

### 6-3. Apex クラスが Agent Builder に表示されない場合

Agentのシステムユーザーが Apex クラスにアクセスできないため。対処法：

1. **権限セットに classAccesses を追加**してデプロイ：
   ```xml
   <classAccesses>
       <apexClass>MyApexClass</apexClass>
       <enabled>true</enabled>
   </classAccesses>
   ```
2. その権限セットを Agent のシステムユーザー（`agentuser@...ext`）に割り当て：
   ```bash
   sf data create record --sobject PermissionSetAssignment \
     --values "PermissionSetId=<権限セットID> AssigneeId=<AgentユーザーID>"
   ```

---

## 7. GenAiPlugin（トピック）の構造

```xml
<?xml version="1.0" encoding="UTF-8"?>
<GenAiPlugin xmlns="http://soap.sforce.com/2006/04/metadata" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <canEscalate>false</canEscalate>
    <description>このトピックが選ばれる条件の説明（英語推奨）</description>
    <developerName>PluginAPIName</developerName>
    <genAiFunctions>
        <functionName>ActionAPIName</functionName>
    </genAiFunctions>
    <genAiPluginInstructions>
        <description>トピック内の具体的な指示</description>
        <developerName>instruction_0</developerName>
        <language xsi:nil="true"/>
        <masterLabel>instruction_0</masterLabel>
        <sortOrder>0</sortOrder>
    </genAiPluginInstructions>
    <language>en_US</language>
    <localDeveloperName>PluginAPIName</localDeveloperName>
    <masterLabel>トピック表示名</masterLabel>
    <pluginType>Topic</pluginType>
    <scope>このAgentが対応できる範囲の説明</scope>
</GenAiPlugin>
```

**無効なフィールド名（エラーになる）：**
- `pluginLabel` → `masterLabel` を使う
- `pluginInstructions` → `genAiPluginInstructions` を使う
- `functions` → `genAiFunctions` を使う

### 7-1. トピック記述のベストプラクティス

トピックの各フィールドはLLMがトピック選択・アクション実行の判断に使う。曖昧な記述はトピック誤選択や期待外れの回答の原因になる。

#### Classification Description（`<description>`）— いつこのトピックを使うか

LLMが「このユーザーの質問はどのトピックに該当するか」を判断するための説明。

**書き方のポイント：**
- このトピックが**どのような質問・リクエストに対応するか**を明確に書く
- 他のトピックと区別できる具体性を持たせる
- 1〜3文で簡潔に

**良い例：**
```
Handles customer inquiries about order status, order modifications,
and return/refund requests for existing orders.
```

**悪い例：**
```
Manages orders.
```
→ 曖昧すぎてLLMが他のトピック（例: 新規注文作成）と区別できない

#### Scope（`<scope>`）— このトピックで何ができて何ができないか

エージェントの行動範囲を定義する。**できること（In Scope）** と **できないこと（Out of Scope）** の両方を明記する。

**書き方のポイント：**
- 「できること」と「できないこと」を箇条書きで明確に分ける
- 境界ケースを具体的に記述する（エージェントが迷う場面を減らす）
- エスカレーション条件を含める

**良い例：**
```
The agent can:
- Look up order status by order number or customer email
- Modify shipping address for orders not yet shipped
- Initiate return requests for orders delivered within the last 30 days
- Provide estimated delivery dates

The agent cannot:
- Cancel orders that have already been shipped
- Process refunds (escalate to human agent)
- Create new orders
- Access payment or credit card information

If the customer's request falls outside these boundaries,
escalate to a human agent.
```

**悪い例：**
```
Handle order inquiries.
```
→ 何ができて何ができないかが不明。エージェントが想定外の操作を試みるリスクがある

#### Instructions（`<genAiPluginInstructions>`）— どのように処理するか

エージェントが会話をどう進め、アクションをどう使うかの具体的な指示。

**書き方のポイント：**
- アクション名は **API名** で記述する（例: `Get_Order_Status` 。表示名ではなくAPI名を使うことでLLMが正確に識別できる）
- アクションの実行順序や前提条件がある場合は明示する
- `must` / `always` / `never` は慎重に使う（LLMが文字通りに従おうとして「スタック」するリスクがある）
- 否定形（`don't`）より肯定形（`always do X`）の方がLLMに明確
- 複数の指示は1つのinstructionボックスにまとめた方が効果的
- 長すぎる指示は応答速度を低下させる。簡潔に保つ
- 複雑な非決定論的ロジックはPrompt Templateアクションに移す

**良い例：**
```
1. First, identify the customer using the Get_Customer_Details action
   with their email address or order number.
2. Once the customer is identified, use the Get_Order_Status action
   to retrieve order information.
3. If the customer wants to modify their order, check the order status
   first. Only use the Modify_Order action if the status is "Processing".
4. For return requests, verify the delivery date is within 30 days
   before using the Create_Return_Request action.
5. Respond in the same language the customer uses.
6. If you cannot resolve the issue, offer to transfer to a human agent.
```

**悪い例：**
```
Help the customer with their order.
```
→ アクションの使い方、順序、条件が全く不明

#### User Input Examples — ユーザーの発話例（UI設定）

Agent Builder UIで設定する、ユーザーの典型的な発話パターン。LLMのトピック分類精度を向上させる。

**書き方のポイント：**
- 典型的な質問を5〜10個記述する
- バリエーションを持たせる（丁寧な言い方、カジュアルな言い方、キーワードだけ等）
- Out-of-scope の例も含めて境界を明確にする

**良い例：**
```
- 注文番号12345の配送状況を教えてください
- My order hasn't arrived yet, can you check?
- 配送先住所を変更したいです
- 返品したいのですが
- Where is my order?
- 注文をキャンセルできますか？（→ Out of scope として処理）
```

#### トピック設計の全体原則

1. **1トピック = 1目的**: 複数の目的を1つのトピックに詰め込まない
2. **最小限から始める**: 少ないトピック・少ない指示で開始し、テスト結果を見て追加
3. **既定トピック活用**: Salesforceが提供する事前構成済みトピックをベースにカスタマイズ
4. **アクション固有の指示はアクション側に**: トピックのInstructionsが複雑になったらアクションのInput/Output Instructionsに分散
5. **会話をスクリプト化しすぎない**: エージェントの自然な対話能力を活かす
6. **テスト駆動**: Agent Builderの Preview や `sf agent test run` で繰り返しテスト

---

## 8. GenAiPlannerBundle の構造

### 8-1. ExternalCopilot 用

```xml
<?xml version="1.0" encoding="UTF-8"?>
<GenAiPlannerBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <description>Agentの説明</description>
    <genAiPlugins>
        <genAiPluginName>TopicAPIName</genAiPluginName>
    </genAiPlugins>
    <masterLabel>表示名</masterLabel>
    <plannerSurfaces>
        <adaptiveResponseAllowed>true</adaptiveResponseAllowed>
        <callRecordingAllowed>false</callRecordingAllowed>
        <surface>SurfaceAction__CustomerWebClient</surface>
        <surfaceType>CustomerWebClient</surfaceType>
    </plannerSurfaces>
    <plannerSurfaces>
        <adaptiveResponseAllowed>true</adaptiveResponseAllowed>
        <callRecordingAllowed>false</callRecordingAllowed>
        <surface>SurfaceAction__Messaging</surface>
        <surfaceType>Messaging</surfaceType>
    </plannerSurfaces>
    <plannerType>AiCopilot__ReAct</plannerType>
</GenAiPlannerBundle>
```

**注意**: ファイル拡張子は `.genAiPlannerBundle`（`-meta.xml` なし）。ディレクトリ構造が必要。

### 8-2. InternalCopilot 用（UIで作成後にリトリーブした構造）

```xml
<GenAiPlannerBundle>
  <!-- トップレベルのアクションリンク -->
  <localActionLinks>
    <genAiFunctionName>AnswerQuestionsWithKnowledge_XXXXX</genAiFunctionName>
  </localActionLinks>

  <!-- トップレベルのトピックリンク -->
  <localTopicLinks>
    <genAiPluginName>TopicName_XXXXX</genAiPluginName>
  </localTopicLinks>

  <!-- トピック定義（インライン） -->
  <localTopics>
    <fullName>TopicName_XXXXX</fullName>
    <developerName>TopicName_XXXXX</developerName>
    <localDeveloperName>TopicName</localDeveloperName>
    <masterLabel>トピックラベル</masterLabel>
    <description>...</description>
    <scope>このトピックを使う条件...</scope>
    <pluginType>Topic</pluginType>
    <language>en_US</language>
    <canEscalate>false</canEscalate>
    <genAiPluginInstructions>
      <description>実行順序などの指示</description>
      <developerName>instruction_0</developerName>
      <language xsi:nil="true"/>
      <masterLabel>instruction_0</masterLabel>
      <sortOrder>0</sortOrder>
    </genAiPluginInstructions>

    <!-- トピック内アクションリンク -->
    <localActionLinks>
      <functionName>ActionFullName_XXXXX</functionName>
    </localActionLinks>

    <!-- アクション定義（インライン） -->
    <localActions>
      <fullName>ActionFullName_XXXXX</fullName>
      <developerName>ActionFullName_XXXXX</developerName>
      <localDeveloperName>ActionLocalName_UUID</localDeveloperName>
      <masterLabel>アクションラベル</masterLabel>
      <description>...</description>
      <invocationTarget>ApexClassName</invocationTarget>
      <invocationTargetType>apex</invocationTargetType>
      <isConfirmationRequired>false</isConfirmationRequired>
      <isIncludeInProgressIndicator>true</isIncludeInProgressIndicator>
      <progressIndicatorMessage>処理中メッセージ</progressIndicatorMessage>
    </localActions>
  </localTopics>

  <!-- トップレベルアクション定義 -->
  <plannerActions>
    <fullName>ActionFullName_XXXXX</fullName>
    ...
  </plannerActions>

  <plannerSurfaces>
    <surface>SurfaceAction__Messaging</surface>
    <surfaceType>Messaging</surfaceType>
  </plannerSurfaces>
  <plannerType>AiCopilot__ReAct</plannerType>
</GenAiPlannerBundle>
```

### 8-3. ExternalCopilot と InternalCopilot の構造比較

| 要素 | ExternalCopilot | InternalCopilot |
|---|---|---|
| トピック参照 | `genAiPlugins/genAiPluginName` | `localTopicLinks/genAiPluginName`（inline localTopics） |
| アクション参照 | `genAiFunctions/genAiFunctionName` | `localActionLinks/genAiFunctionName`（inline plannerActions） |
| Apex アクション追加 | GenAiFunction → GenAiPlugin → genAiPlugins | **UIのみ**（localActions に UUID形式で自動生成） |
| plannerSurfaces | CustomerWebClient, Messaging など | Messaging |

### 8-4. InternalCopilot の命名規則（UI自動生成パターン）

- `localTopics.fullName`: `{TopicAPIName}_{PlannerID}` 例: `AccountSummaryTopic_16jIe0000008Owc`
- `localActions.fullName`: `X{UUID}_{BotVersionKeyID}` 例: `X56c59cd0_f43d_56b5_ce55_8151629a97f3_179Ie000000wnDO`

**この命名規則を手動で再現するのは困難。UIで追加してからretrieveするのが正解。**

### 8-5. schema.json ファイル

各 `plannerActions` および `localActions` には対応するスキーマファイルが必要：

```
genAiPlannerBundles/{AgentName}/
├── plannerActions/
│   └── {actionFullName}/
│       ├── input/schema.json
│       └── output/schema.json
└── localActions/
    └── {topicFullName}/
        └── {actionFullName}/
            ├── input/schema.json
            └── output/schema.json
```

**スキーマフォーマット（Lightning型）：**
```json
{
  "required": ["paramName"],
  "unevaluatedProperties": false,
  "properties": {
    "paramName": {
      "title": "パラメータ名",
      "description": "説明",
      "const": "",
      "lightning:type": "lightning__textType",
      "lightning:isPII": false,
      "copilotAction:isUserInput": true
    }
  },
  "lightning:type": "lightning__objectType",
  "lightning:textIndexed": true
}
```

**lightning:type 値一覧：**
- `lightning__textType` - String/テキスト
- `lightning__booleanType` - Boolean
- `lightning__richTextType` - リッチテキスト（HTML）
- `lightning__objectType` - オブジェクト（プロパティのコンテナ）

**出力スキーマの追加属性：**
- `copilotAction:isDisplayable: true` - UIに表示する
- `copilotAction:isUsedByPlanner: true` - プランナーが使用する
- `maxLength: 100000` - テキスト最大長

---

## 9. Prompt Template をエージェントアクションとして使う

### 9-1. 概念

Agentforce エージェントが要約・分類・翻訳等を行う際、エージェント内部のLLMに任せるのではなく、**Prompt Builder で管理された構造化プロンプト**を呼び出せる。

**メリット：**
- プロンプトをバージョン管理・独立テストできる
- 入力データの構造・出力フォーマットを明示的に定義できる
- プロンプト変更時にエージェント本体を触らなくてよい

### 9-2. Prompt Builder でのテンプレート作成

1. Setup → Quick Find → **Prompt Builder**
2. **New Prompt Template**
   - Template Type: **Flex** が最も汎用的（任意の入力変数を定義可能）
   - Template Type: **Field Generation** は特定フィールドへの出力専用
3. 入力リソース（レコード・Apex・フロー経由のデータ）を設定
4. プロンプト本文を作成・テスト

### 9-3. エージェントアクションとして登録（UI）

1. Setup → **Agent Actions** → New Agent Action
   - **Reference Action Type**: `Prompt Template`
   - **Reference Action**: 作成したプロンプトテンプレートを選択
2. **Action Instructions**: エージェントがこのアクションを使う条件・目的を記述
3. **Input Instructions**: 前のアクションから渡されたデータをどう使うか記述
4. **Output Instructions**: 出力をどう表示するか（"Show in conversation" トグル）
5. 作成後、Agent Builder でトピックのアクションとして追加

### 9-4. データフローパターン（推奨）

```
Apex アクション（データ取得）
  → 出力: JSON文字列（取引先情報・商談・ケースなど）
  → Custom Variable に格納

Prompt Template アクション（要約生成）
  → 入力: Custom Variable の値（JSON）
  → プロンプト内で整形・要約
  → 出力: 要約テキスト → 会話に表示
```

### 9-5. GenAiPlannerBundle メタデータでの表現

UIで追加後にretrieveすると以下の形式で保存される：

```xml
<localActions>
  <fullName>XxxxxxxUUID_BotVersionID</fullName>
  <invocationTarget>YourPromptTemplateName</invocationTarget>
  <invocationTargetType>promptTemplate</invocationTargetType>
  <masterLabel>要約生成</masterLabel>
  <description>取引先サマリーを生成する</description>
  <isConfirmationRequired>false</isConfirmationRequired>
  <isIncludeInProgressIndicator>true</isIncludeInProgressIndicator>
  <progressIndicatorMessage>生成中...</progressIndicatorMessage>
</localActions>
```

### 9-6. デプロイ上の注意

- Prompt Template が Flow を参照し、そのFlowがApexアクションを含む場合、**Apex → Flow → Prompt Template の順で別パッケージでデプロイ**する必要あり
- GenAiPromptTemplate は API version 60 以上を使用
- 依存するカスタムオブジェクト・フィールドが対象orgに存在していることが前提

---

## 10. Context Variables と Agent Quick Action

### 10-1. Agentforce の変数種別

| 変数種別 | プレフィックス | 説明 | 編集可否 |
|---|---|---|---|
| **Context Variables** | `$Context` | システム生成。ユーザー情報・セッション情報を自動保持 | 読み取り専用（`$Context.EndUserLanguage` のみ変更可） |
| **Custom Variables** | なし（任意名） | ユーザー定義。アクションの入出力値を保持するメモリ | Agent Builder で設定可 |

### 10-2. 現在のレコード情報をエージェントに渡す方法

**Employee Agent は「現在開いているレコード」を自動的に知ることはできない。**
**Agent Quick Action** 機能を使うことで、レコードページから文脈付きでエージェントを起動できる。

#### Agent Quick Action の仕組み

1. **Lightning レコードページ**に「エージェントクイックアクション」コンポーネントを配置
2. ユーザーがボタンをクリックすると、**事前設定した Utterance（プロンプト文）** がエージェントパネルに自動送信される
3. Utterance にはレコードの**マージフィールド**（例: `{!Account.Name}`, `{!Account.Id}`）を埋め込み可能
4. エージェントはその Utterance を受け取り、通常の会話として処理する

**例：**
```
「{!Account.Name}（{!Account.Id}）の取引先サマリーを生成してください」
```
→ エージェントパネルには「Omega, Inc.（001Ie000001ABCDXXXX）の取引先サマリーを生成してください」と送信される

#### Agent Quick Action の設定手順

1. Setup → Quick Find → **Quick Actions** → New Quick Action
   - Action Type: **"Agent"**
   - Agent: 対象の Employee Agent を選択
   - Utterance Template: マージフィールド入りの文字列を設定
2. 対象オブジェクトの **Page Layout** にクイックアクションを追加
3. Lightning App Builder でレコードページに配置・保存

### 10-3. Custom Variables の活用（アクション間のデータ受け渡し）

Agent Builder でトピック変数を設定することで、アクション間でデータを引き継げる：

1. Agent Builder → 対象トピック → **Variables** タブ
2. 変数を作成（データ型・名前を設定）
3. 各アクションの **Input Mapping** / **Output Mapping** で変数をマッピング

**ユースケース例：**
- アクション1の出力 → 変数 `accountRecord` に格納
- アクション2の入力 → 変数 `accountRecord` から取得

#### Filter（実行条件）

変数に Filter を設定することで「この変数が埋まっていない限り実行しない」という依存関係を表現できる。
ただし、柔軟性が失われるため**本当に必要な場合のみ**使用すること。

---

## 11. Employee Agent のアクセス権設定

### 11-1. 権限セットまたはプロファイルに「Enabled Agent Access」を追加

Employee Agent を作成しただけではユーザーは使えない。

#### UI での設定

1. Setup → Quick Find → **Permission Sets** を選択
2. **Enabled Agent Access** を編集
3. アクセスを許可する Employee Agent を選択 → Save

#### CLI での設定（SetupEntityAccess レコード直接INSERT）

`enabledAgentAccesses` を PermissionSet メタデータ XML に追加してデプロイすると **Element invalid** エラーになる（API v65/66 未対応）。代わりに `SetupEntityAccess` レコードを REST API で直接INSERTする：

```bash
# 1. BotDefinition ID を取得
BOT_ID=$(sf data query --query "SELECT Id FROM BotDefinition WHERE DeveloperName = '<AgentAPIName>'" \
  -o <username> --json | python3 -c "import json,sys; print(json.load(sys.stdin)['result']['records'][0]['Id'])")

# 2. 対象の PermissionSet ID を取得
PS_ID=$(sf data query --query "SELECT Id FROM PermissionSet WHERE Name = '<PSName>'" \
  -o <username> --json | python3 -c "import json,sys; print(json.load(sys.stdin)['result']['records'][0]['Id'])")

# 3. アクセストークン取得 & SetupEntityAccess を INSERT
ACCESS_TOKEN=$(sf org display -o <username> --json | python3 -c "import json,sys; print(json.load(sys.stdin)['result']['accessToken'])")
INSTANCE_URL="https://<mydomain>.my.salesforce.com"

curl -s -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"ParentId\": \"$PS_ID\", \"SetupEntityId\": \"$BOT_ID\"}" \
  "$INSTANCE_URL/services/data/v66.0/sobjects/SetupEntityAccess"
```

---

## 12. デプロイ（Deploy）

```bash
# Validate 先行（推奨）
sf project deploy validate \
  --source-dir force-app \
  --ignore-conflicts \
  --target-org <username>

# Deploy
sf project deploy start \
  --source-dir force-app \
  --ignore-conflicts \
  --target-org <username>
```

---

## 13. 削除の制限

| メタデータ型 | 削除方法 |
|---|---|
| Bot | `sf agent deactivate` → `destructiveChanges.xml` で削除 |
| GenAiPlannerBundle | **Metadata API での削除不可** → UIから削除 |
| AiAuthoringBundle | **削除不可**（API/CLIサポートなし） |
| GenAiPlugin | `destructiveChanges.xml` で削除可能 |
| GenAiFunction | `destructiveChanges.xml` で削除可能 |

**アクティブなBotのバージョンを削除しようとすると `Cannot delete an active bot version` エラー。必ず先にdeactivateすること。**

---

## 14. よくあるエラーと対処

| エラー | 原因 | 対処 |
|---|---|---|
| `Invocable Target Type or Name not valid` | invocationTargetType が間違い | `apex`（小文字）を使用 |
| `InvocableTarget not found: 01pXXX...` | invocationTarget にクラスIDが入っている | Apexクラス名（DeveloperName）に修正 |
| `GenAiFunction dependency not found` | 依存先が未存在 | 依存先を先にデプロイ |
| `BotVersion required` | Bot デプロイ時にBotVersionがない | `sf agent create` で作成 |
| `Required field is missing: entryDialog` | BotVersion を手動デプロイしようとした | `sf agent create` で作成（手動不可） |
| `enableGenerativeActions invalid` | InternalCopilot BotVersionでは使用不可 | BotVersionは `<fullName>v1</fullName>` のみにする |
| `Cannot delete an active bot version` | アクティブなBotVersionは削除不可 | `sf agent deactivate` してから削除 |
| `GenAiPlannerBundle not available for delete` | Metadata API 削除不可 | UIから削除 |
| `Element systemDialogType invalid` | 旧形式のBotVersionファイル | 取得し直してクリーンなファイルを使う |
| `Apex class not visible in Agent Builder` | Agentユーザーにクラスアクセス権がない | 権限セットでclassAccessesを付与 |
| `Element XXX invalid at this location in type YYY` | そのフィールド名が存在しない | 自動生成されたメタデータを参照して正しい要素名を確認 |
| デプロイ "Succeeded" なのに反映されない | InternalCopilotへのMetadata API追加は無言で無視される | UIのAgent Builderで操作する |

---

## 15. Agent設定の前提条件（org設定）

1. **Setup → Einstein Setup** → Einstein を有効化
2. **Setup → Agents → Agentforce Agents** を有効化
3. **Setup → Agents → Enable the Agentforce (Default) Agent** をオン（Employee Agent用）
4. AgentforceライセンスがUserに割り当て済みであること

---

## 16. ベストプラクティス

- **Agent作成は `sf agent create` を使う**（手動のMetadata API Botデプロイは制限多い）
- **Employee Agent（InternalCopilot）はUIから作成する**（CLIでは不可）
- **GenAiFunctionの invocationTarget はApexクラス名を指定**（IDではない）
- **agentSpec.yaml のトピック名は英語で記述**（日本語だとLLM生成が失敗することがある）
- **BotVersionは手動デプロイしない**（`sf agent create`の内部APIを使う）
- **InternalCopilotのトピック/アクション追加はUIで行い、retrieveでローカル保存**
- Apex @InvocableMethod を追加したら Agent Builder の「呼び出し可能なメソッド」タブから手動登録が必要

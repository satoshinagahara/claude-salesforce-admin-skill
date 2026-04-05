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
| `InternalCopilot` | Agentforce 従業員エージェント | Salesforce CRM内で社内ユーザーが使うAIアシスタント | 社内ユーザー |

> **重要: Agentforce（デフォルト）は非推奨**
> 2025年6月17日以降、Agentforce（デフォルト）には新機能・改善が提供されない。新しいSalesforce環境では使用不可。
> **Agentforce 従業員エージェント（Employee Agent）への移行が推奨**される。
> 従業員エージェントは複数作成可能、テンプレート対応、Slack/Experience Cloud連携が可能。

### 0-2. BotDefinition.AgentType の値

| AgentType 値 | 対応する Type | 説明 |
|---|---|---|
| null | `Bot` | 従来型Einstein Bot |
| `EinsteinServiceAgent` | `ExternalCopilot` | Agentforce Service Agent（顧客向け） |
| `EinsteinCopilot` | `InternalCopilot` | Agentforce（デフォルト）— **非推奨**。従業員エージェントへの移行推奨 |
| `AgentforceEmployeeAgent` | `InternalCopilot` | Agentforce 従業員エージェント（後継。UI で作成） |

> **Agentforce（デフォルト）→ 従業員エージェントの移行**: Setup → Agents → Agentforce(デフォルト)のドロップダウン → 「従業員エージェントに移行」で自動移行可能。トピック・アクション・変数・設定が引き継がれる。

### 0-3. 種別ごとの利用シーン比較

| 種別 | 配置場所 | アクセス | ライセンス |
|---|---|---|---|
| ExternalCopilot (Service Agent) | 外部サイト・メッセージング | 外部顧客 | Agentforce for Service |
| InternalCopilot (従業員エージェント) | Lightning Experience・Salesforceモバイル・Slack・Experience Cloud Webメッセージング | Salesforceログイン済みユーザー | Agentforce for Employees |
| InternalCopilot (デフォルト・非推奨) | Lightning Experience・Salesforceモバイル | Salesforceログイン済みユーザー | Agentforce for Employees |
| Bot | Embedded Messaging等 | 外部顧客（従来型） | Einstein Bot |

**Agentforce（デフォルト）と従業員エージェントの主な違い:**
- デフォルト: orgに1つのみ。テンプレート非対応。Lightning/モバイルのみ
- 従業員エージェント: 複数作成可。テンプレート対応。Slack・Experience Cloud連携可。UIで権限管理が容易

### 0-4. CLI・API の制限

| 操作 | ExternalCopilot | InternalCopilot |
|---|---|---|
| `sf agent create` で作成 | OK | 不可（`--type internal` 指定でも常にExternalCopilotが生成される。2026-03再検証済み） |
| `sf agent publish authoring-bundle` で作成 | OK | 不可（同上） |
| Metadata API でBot作成 | OK | 不可 |
| `BotDefinition.Type` を変更 | 不可（書き込み不可フィールド） | 不可 |
| トピック/アクションをMetadata APIで追加 | OK（genAiPlugins/genAiFunctions） | 不可（デプロイ Succeeded でも無言で無視される） |
| トピック/アクションをUI Agent Builderで追加 | OK | OK |

> **注意**: `agentSpec.yaml` の `agentType: internal` はLLMのトピック生成コンテキストに影響するだけで、作成されるエージェントの種別（BotDefinition.Type）には一切影響しない。

### 0-5. Employee Agent（InternalCopilot）の作成方法

Employee Agent は CLI から作成できないため、**Salesforceセットアップ画面から作成**する：

#### 新規作成（Agentforce 従業員エージェント）
1. Setup → Agents → **New** ボタン（Agent Creator のガイド付き設定が起動）
2. **Agentforce 従業員エージェント** テンプレートを選択
3. **基本設定を入力**（以下3項目はAgent LLMの推論品質に直結する重要項目）：
   - **説明（Description）**: エージェントの目的・対象領域を簡潔に記述。Agent LLMがトピック選択の文脈として使用する
   - **ロール（Role）**: エージェントのペルソナ・専門性を定義。「あなたは〇〇の専門家です」形式で、対応言語の指示も含める
   - **会社（Company Description）**: どのようなビジネスを行っているか。業界・事業内容・重要課題を記述。Agent LLMが回答の文脈を理解するために使用する
4. 保存
5. Agent Builder で トピック・アクションを設定
6. 作成後は `sf project retrieve start --metadata "Bot:<APIName>"` で取得可能

**基本設定の記述例：**
```
説明: 製造業向けBOM（部品表）分析エージェント。製品のBOM構成分析と
      サプライヤーリスク分析を行い、コスト最適化や供給リスクの可視化を支援します。

ロール: あなたは製造業のBOM分析とサプライチェーンリスク管理の専門家です。
        製品のBOM階層構成、コスト分析、サプライヤー依存度の評価、および
        商談パイプラインへの影響分析を行います。日本語で回答してください。

会社: 製造業を営む企業で、複数の製品ラインを持ち、各製品は多層BOM（部品表）で
      構成されています。複数のサプライヤーから部品を調達しており、
      サプライチェーンリスクの管理と製品コストの最適化が重要な経営課題です。
```

#### Agentforce（デフォルト）からの移行
1. Setup → Agents → Agentforce(デフォルト)のドロップダウンメニュー
2. 「**従業員エージェントに移行（Migrate to an Employee Agent）**」を選択
3. 新しいエージェントの名前・API参照名を入力 → 作成
4. トピック・アクション・変数・設定が自動引き継ぎされる
5. 移行後、ユーザーにアクセス権を付与して有効化
6. 旧Agentforce（デフォルト）を無効化

> **注意**: カスタムプロンプト、名前を変更したアクション、権限セット/プロファイルの表示設定は新エージェントで再作成が必要。

作成後のメタデータ：
- `Bot.type = InternalCopilot`
- `Bot.agentType = AgentforceEmployeeAgent`（自動設定）
- `GenAiPlannerBundle.plannerSurfaces` に Employee 向けの surface（UI自動設定）

### 0-6. InternalCopilot の正しい実装フロー

```
1. Apex クラス作成（@InvocableMethod, @InvocableVariable）→ sf project deploy start
2. GenAiFunction 作成 → sf project deploy start
3. Prompt Template 作成（必要な場合）→ sf project deploy start
4. UIでEmployee Agentを新規作成（Setup → Agents → New → "Employee Agent"）
5. UIのAgent Builderでトピックを作成（Topics タブ → + New Topic）
6. UIでアクションを追加（+ Add from Asset Library → Apex/Prompt Template アクション選択）
7. トピック Instructions を記述（アクションの実行順序・条件を明確に指示）
8. Activate → テスト
9. Metadata リトリーブでローカル保存
   sf project retrieve start --metadata "GenAiPlannerBundle:<APIName>"
```

### 0-7. Agentforce のアーキテクチャ（重要な理解）

**Agent LLM と Prompt Template LLM の役割分担:**

```
[ユーザーの質問]
    ↓
[Agent LLM] ← トピック選択 + アクション推論（ルーティングのみ）
    ↓           ※ Agent LLM 自身はテキスト生成・要約・分析を行わない
[アクション実行]
    ├→ GenAiFunction (Apex) → データ取得のみ、LLM呼び出しなし
    └→ Prompt Template → 内部でLLMを呼び出してテキスト生成（要約・分析・示唆等）
    ↓
[結果をユーザーに返す]
```

- **Agent LLM**: どのトピック・アクションを使うかの判断（推論）のみ
- **GenAiFunction (Apex)**: データの取得・加工。LLMは関与しない
- **Prompt Template**: 実際のテキスト生成（要約、分析、示唆等）を担当するLLMを呼び出す

**設計上の示唆:**
- データ取得だけのApexアクションを直接ユーザーに返しても、生のJSONが返るだけで意味がない
- **要約・分析・示唆が必要な場合は、必ずPrompt Templateアクションを組み合わせる**
- Agent LLMはアクションのチェーン（複数アクションの順序実行）が可能。トピックInstructionsで実行順序を明示する

### 0-8. トピック設計のベストプラクティス（実証済み）

**アクション数の最適化:**
- トピック内のアクション数は**必要最小限**に絞る
- 重複するアクション（個別取得 vs 一括取得）があるとAgent LLMがどれを呼ぶか迷う
- 例: 個別5アクション + 統合1アクション → 統合1アクションに集約

**Instructions の書き方:**
- アクションの**実行順序を明示**する（「まずAを実行し、その結果をBに渡す」）
- アクション間の**データの受け渡し方法を明示**する
- **対応する質問の例**を含める（Agent LLMのトピック選択精度向上）

**⚠️ 同一セッション内でのAction再実行の強制（2026-03 実証済み・重要）:**

Agent LLM（ReActプランナー）は、同一セッション内で過去にActionを実行済みの場合、2回目以降の質問に対してActionを再実行せず、**1回目の出力テキストから直接回答を生成する**ことがある。これは以下のケースで問題になる：

- ユーザーの質問内容（userQuery）によって分析の焦点が変わるAction
- 同一データに対して異なる角度からの分析を提供するAction

**対策**: Instructionsに「毎回必ずActionを実行する」旨と「質問が異なれば結果が変わる」理由を明示する。

```
## NG（再実行されない場合がある）
1. 「分析アクション」を実行する
   - opportunityIdには現在表示中の商談のIDを渡す
2. アクションの出力をユーザーに返す

## OK（再実行が強制される — 実証済み）
1. ユーザーが質問をするたびに、必ず「分析アクション」を実行する
   - 過去の会話で既に結果を取得していても、質問が異なればアクションを再実行すること
   - opportunityIdには現在表示中の商談のIDを渡す
   - userQueryにはユーザーの質問内容をそのまま渡す（質問によって分析の焦点が変わるため）
2. アクションの出力（分析レポート）をそのままユーザーに返す
```

**背景**: Agent LLMは効率化のために「既に十分な情報がある」と判断するとActionをスキップする。Instructionsで「質問によって結果が変わる」と明示することで、Agent LLMに再実行の必要性を理解させる。userQueryパラメータの存在が再実行の合理的な理由となる。

**典型的なパターン: データ取得 → LLM生成:**

⚠️ **2アクション分離パターン（Apex → Prompt Template）はデータ量が少ない場合のみ有効。**
数KB以上のJSON受け渡しが必要な場合は、Apex内でConnectApi経由でPrompt Templateを呼び出す1アクション統合パターンを使用すること。
詳細は `prompt-templates.md` セクション 6-5 を参照。

```
## パターンA: 2アクション分離（データが小さい場合のみ）
1. まず「データ取得アクション」を実行し、データを取得する
2. 次に「Prompt Templateアクション」を実行する
   - 入力に手順1の結果を渡す
3. 手順2の結果をユーザーに返す

## パターンB: 1アクション統合（推奨・データ量を問わない）
1. 「分析アクション」を実行する（Apex内でデータ取得+Prompt Template呼び出しを一括実行）
2. アクションの出力（分析レポート）をそのままユーザーに返す
```

### 0-9. 複数Agent共存とUI挙動（2026-03 調査）

**1 orgでの複数Agent共存:**
- 1 orgあたり**最大20 Agent**まで作成・有効化可能
- 複数のEmployee Agent（InternalCopilot）を同時にアクティブにできる
- UIにはAgentforceアイコン横に**ドロップダウン**が表示され、ユーザーが手動で切替
- ドロップダウンに表示されるAgentは**パーミッションセット/プロファイル**で制御

**Topic/Actionの上限:**

| 項目 | 上限 |
|---|---|
| Agent数 / org | 20 |
| Topic数 / Agent | 15 |
| Action数 / Topic | 15 |

**設計判断: 1 Agent集約 vs 複数Agent分散:**

| 方式 | 適するケース |
|---|---|
| **1 Agent集約** | Topic 10-15件以下で、1人のユーザーが全機能を使う場合。管理がシンプル |
| **複数Agent分散** | 部門別に異なるAgent（営業用 / 製造用等）を提供する場合。パーミッションで出し分け |
| **Multi-Agent Orchestration** | プライマリAgentが意図を判断→専門Agentにルーティング。複雑なドメインが多い場合 |

**Topicの設計品質がルーティング精度を左右する:**
- Topic数が多いこと自体は問題ではない（15件上限まで実用的）
- Agent LLMのルーティング精度は**Topic名・分類記述（Description）・スコープ（Scope）の明確さ**に依存
- 曖昧なScope（「何でも対応します」）は他Topicとの判別を困難にする
- 各Topicの`aiPluginUtterances`（対応する質問例）を充実させるとルーティング精度が向上

### 0-10. レコードコンテキストの取得（2026-03 調査）

**Agent Topicはレコードページのコンテキストを自動的に認識しない。**
ただし以下の方法でレコードIDを取得可能：

**方法1: Apex Actionの入力変数（推奨）**
- `@InvocableVariable` に `recordId` を定義
- レコードページから起動された場合、Salesforceが**自動的にIDを注入**
- Agent LLMが「現在表示中のレコード」の文脈を理解し、適切にIDを渡す

**方法2: Context Variable**
- Agent設定でContext Variableを定義（Text, Id等のデータ型を指定）
- Topic Instructions 内で「Context Variable のレコードIDを使用する」と明示
- 各ActionのInput MappingでContext Variableの値をパラメータに割り当て

**方法3: Record-Triggered Flow**
- レコード更新時にFlowからAgentを非同期呼び出し
- FlowのトリガーレコードIDがそのまま渡される

### 0-11. Custom Lightning Types（CLT）によるチャット内LWCレンダリング（2026-03 調査・実装中）

**概要**: Agentforceのチャット内にカスタムLWCコンポーネントをインラインでレンダリングできる仕組み。API v64.0+。

**前提条件:**
- Apex Actionの出力が構造化クラス（global, @AuraEnabled, @JsonAccess）であること
- LWCが `lightning__AgentforceOutput` ターゲットを持つこと
- Setup → Lightning 種別 で管理される

**ファイル構成:**
```
lightningTypes/
  MyTypeName/
    schema.json                      ← データ構造定義（title, lightning:type必須）
    lightningDesktopGenAi/
      renderer.json                  ← LWCコンポーネントの指定
```

**schema.jsonの2つのフォーマット:**

**パターンA: Apexクラス参照（推奨・公式サンプル準拠）:**
```json
{
  "title": "My Type Name",
  "description": "説明",
  "lightning:type": "@apexClassType/c__MyApexClassName"
}
```
- propertiesの手動定義不要 — Apexクラスのフィールドが自動的にスキーマになる
- チャネルに「Agentforce (Lightning Experience)」が表示される
- レンダラータブが出現する

**パターンB: 手動プロパティ定義（非推奨）:**
- 各プロパティに `title` と `lightning:type` が必須
- `lightning:type` の値: `lightning__textType`, `lightning__numberType`, `lightning__booleanType`, `lightning__objectType`
- チャネルが「エクスペリエンスビルダー」のみになり、レンダラータブが出ない
- `$schema` キーワードは使用不可
- 配列型に `lightning__collectionType` は使用不可

**renderer.jsonの正しいフォーマット（実証済み）:**
```json
{
  "renderer": {
    "componentOverrides": {
      "$": {
        "definition": "c/myComponentName"
      }
    }
  }
}
```

> ⚠️ 以下のフォーマットは全て**失敗する**:
> - `{"component": "c/myComponent"}` → additionalProperties error
> - `{"lwc": {"name": "c/myComponent"}}` → additionalProperties error
> - `{"definition": "c:myComponent"}` → Internal Server Error
> - `{"namespace": "c", "name": "myComponent"}` → additionalProperties error
> - `{}` → 空ファイルエラー

**Apex出力クラスの要件:**
```apex
@JsonAccess(serializable='always' deserializable='always')
global class MyResult {
    @AuraEnabled
    global String reportText;
    // 全フィールドに @AuraEnabled が必要
    // クラスは global（publicは不可）
    // トップレベルクラスのみ（内部クラス不可）
}
```

**現状の課題（2026-03時点）:**
- schema.json + renderer.json のデプロイは成功する
- Agent LLMは `show` 関数で構造化データを返す
- しかしLWCがチャット内でレンダリングされない（JSONが生テキスト表示）
- renderer → LWCの紐付けが効いていない可能性。調査継続中

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

> **推奨**: ExternalCopilot（Service Agent）の構築には **Agent Script（Authoring Bundle）方式** を使う（セクション17参照）。以下の `sf agent create` は旧方式だが引き続き使用可能。

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

#### Instructions（`<genAiPluginInstructions>`）— どのように処理するか（重要）

エージェントが会話をどう進め、アクションをどう使うかの**ステップバイステップの具体的な指示**。
**トピック設定で最も重要な項目。** Description（分類）やScope（範囲）が正しくても、Instructionsが不十分だとエージェントはアクションを正しく実行できない。

**書き方のポイント：**
- **必ずステップバイステップで記述する**（「## 実行手順」→ 番号付きリスト形式）
- アクション名は **API名** で記述する（例: `Get_Order_Status` 。表示名ではなくAPI名を使うことでLLMが正確に識別できる）
- **各ステップでどのアクションを使うか、入力に何を渡すか、出力をどうするかを明示する**
- アクション間の**データの受け渡し方法を明示**する（「手順2の出力を手順3の入力に渡す」）
- `must` / `always` / `never` は慎重に使う（LLMが文字通りに従おうとして「スタック」するリスクがある）
- 否定形（`don't`）より肯定形（`always do X`）の方がLLMに明確
- 複数の指示は1つのinstructionボックスにまとめた方が効果的
- 長すぎる指示は応答速度を低下させる。簡潔に保つ
- 複雑な非決定論的ロジックはPrompt Templateアクションに移す

**良い例（データ取得→LLM生成パターン）：**
```
## 実行手順
ユーザーが製品のBOM分析を依頼した場合、必ず以下の順序で実行してください：

1. ユーザーの質問から対象の製品（Product2）を特定する
   - 製品名やProductCodeが含まれている場合はそれを使う
   - 不明な場合はユーザーに確認する

2. 「製品BOM構成取得」アクション（BOMAnalysisGetProductBOM）を実行する
   - 入力: 特定した製品のレコードID（productId）
   - このアクションはBOM全階層データをJSON形式で返す

3. 手順2の出力（analysisData）を「製品BOM分析レポート」アクション（BOMProductAnalysis）の入力に渡す
   - 入力: analysisDataに手順2で取得したJSON文字列をそのまま渡す

4. 手順3の出力（分析レポート）をそのままユーザーに返す
```

**良い例（汎用パターン）：**
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
→ アクションの使い方、順序、条件が全く不明。Agent LLMはどのアクションをどの順で使うか判断できない

#### User Input Examples — ユーザーの発話例（UI設定・重要）

Agent Builder UIで設定する、ユーザーの典型的な発話パターン。**Agent LLMのトピック分類精度に直接影響する重要な設定項目。**
Description・Scope・Instructionsと並び、トピック設定時に必ず入力すること。

**書き方のポイント：**
- 典型的な質問を**5〜10個**記述する
- **バリエーションを持たせる**（丁寧な言い方、カジュアルな言い方、キーワードだけ等）
- **具体的な固有名詞を含む例**を入れる（Agent LLMがパターンマッチしやすくなる）
- **ユーザーが実際に使う言語**で記述する（日本語環境なら日本語の例を含める）
- Out-of-scope の例も含めて境界を明確にする

**良い例（製造業BOM分析エージェント）：**
```
トピック「製品BOM分析」の入力例:
- Battery, High CapacityのBOM構成を分析して
- B-1000のBOMコスト内訳を教えて
- この製品のサプライヤー依存度はどうなっていますか？
- 製品のBOM階層を見せて
- BOMのコスト削減余地はありますか？

トピック「サプライヤーインパクト分析」の入力例:
- 東亜電子工業が供給停止したらどうなる？
- このサプライヤーのリスクを分析して
- サプライヤーの影響度を調べてほしい
- この取引先が納品できなくなった場合の影響範囲は？
- 供給リスクのある商談を教えて
```

**良い例（汎用）：**
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

### 9-3a. エージェントアクション指示の書き方（重要）

Agent Builderでアクションをトピックに追加すると、アクションごとに3つの指示フィールドを設定できる。
**トピックのInstructionsがフロー全体の手順を示すのに対し、アクション指示は個々のアクションの使い方をAgent LLMに伝える。**

| フィールド | 目的 | 記述すべき内容 |
|---|---|---|
| **エージェントアクション指示（Agent Action Instructions）** | このアクションを使う条件・目的 | いつ・なぜこのアクションを呼ぶか。前提条件（先に実行すべきアクション）があれば明記 |
| **入力の指示（Input Instructions）** | 入力パラメータに何を渡すか | 前のアクションの出力をどう渡すか。パラメータごとに記述 |
| **出力の指示（Output Instructions）** | 出力をどう扱うか | ユーザーにそのまま表示するか、次のアクションに渡すか |

**記述例（データ取得→Prompt Template生成パターン）：**

```
■ データ取得アクション（Apex GenAiFunction）
  エージェントアクション指示: ユーザーが指定した製品のBOMデータを取得します。
                              製品のレコードIDが必要です。
  入力の指示（productId）: ユーザーが指定した製品のSalesforceレコードIDを渡してください。
  出力の指示: この出力は直接ユーザーに表示せず、次のアクション（分析レポート生成）の入力として使用します。

■ Prompt Templateアクション
  エージェントアクション指示: データ取得アクションで取得したJSON形式の分析データをもとに、
                              分析レポートを生成します。
                              必ず先にデータ取得アクションを実行してからこのアクションを呼び出してください。
  入力の指示（analysisData）: データ取得アクションの出力であるJSON文字列をそのまま渡してください。
  出力の指示: 生成されたレポートをそのままユーザーに表示してください。
```

**ポイント：**
- **前提条件を明記**: 「必ず先に〇〇を実行してから」— Agent LLMがアクション実行順序を守るために重要
- **データの受け渡しを明示**: 「〇〇の出力をそのまま渡す」— Agent LLMが中間データを正しくルーティングするために必要
- **出力の用途を明示**: 「ユーザーに表示」or「次のアクションに渡す」— Agent LLMが不要な中間出力をユーザーに見せないようにする

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

#### レコードページコンテキストの3箇所明示パターン（重要・実証済み）

Agent Quick Actionなしでも、ユーザーが「このサプライヤーの分析をして」とレコードページ上で発話した場合にIDを聞き返さないようにするには、**以下の3箇所すべて**にレコードページコンテキストの指示を入れる必要がある：

1. **トピック Instructions**: 「ユーザーがレコードページにいる場合は、現在のレコードIDをそのまま使用する。IDを聞き返さないこと。」
2. **GenAiFunction description（XML）**: 「ユーザーがXXXのレコードページにいる場合は、現在のレコードIDをそのままxxxIdとして渡してください。」
3. **Apex InvocableVariable description**: 「レコードページにいる場合は現在のレコードIDをそのまま渡す」

1箇所だけでは不十分。Agent LLMはトピック→GenAiFunction→パラメータの順に参照するため、各段階で明示する必要がある。

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
| Bot | `sf agent deactivate` → `destructiveChanges.xml` で削除。ただしAiAuthoringBundleが参照するBotはAgentGraphReference相互参照により削除不可（UIから削除が必要） |
| GenAiPlannerBundle | **Metadata API での削除不可** → UIから削除 |
| AiAuthoringBundle | **削除不可**（API/CLI/Tooling APIいずれもサポートなし） |
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
3. AgentforceライセンスとFlex Creditsがプロビジョニング済みであること
4. **Employee Agent の場合**: Setup → Agents → New → **Agentforce 従業員エージェント** テンプレートで作成
   - ~~旧方式: 「Enable the Agentforce (Default) Agent」をオン~~ → **非推奨（2025年6月17日以降、新機能なし・新環境で使用不可）**
   - 既にAgentforce（デフォルト）を使用中の場合は「従業員エージェントに移行」で移行可能

---

## 16. アセットライブラリと再利用可能アセットの活用

### 16-1. アセットライブラリとは

Agent Builder のUIで「**+ New → Add from Asset Library**」から利用できる、Salesforceが事前構成した**標準トピック**と**標準アクション**のコレクション。最小限のカスタマイズで再利用でき、新規エージェント構築の出発点として推奨される。

標準トピック・アクションはSalesforceにより継続的に追加・更新されるため、**静的なリストではなく、実装タイミングで都度orgを調査する**のが正しいアプローチ。

### 16-2. エージェント作成時の推奨フロー（重要）

**新しいエージェントを作成する際は、まず既存アセットの再利用を検討し、不足分のみ新規実装する。**

```
1. 要件整理
   - エージェントの目的・対象ユーザー・必要な機能を明確にする

2. org内の既存アセットを調査（標準 + カスタム両方）
   - 標準トピック・アクション: アセットライブラリで確認
   - 既存カスタムトピック・アクション: 他のエージェント用に作成済みのものも再利用可能
   - 既存Apex @InvocableMethod / Flow: アクションとして登録可能なものがないか確認

3. 判断
   - 標準アセットで対応可能 → そのまま追加（必要に応じてカスタマイズ）
   - 既存カスタムアセットで対応可能 → 再利用
   - 不足がある → カスタムトピック/アクションを新規作成

4. 新規実装（必要な場合のみ）
   - アクション種別を選択: Apex / Flow / Prompt Template
   - トピック設計: Classification Description / Scope / Instructions
   - テスト → 有効化
```

### 16-3. org内アセットの調査方法（CLI）

#### 既存トピック一覧の取得

```bash
# org内の全トピック（標準 + カスタム）を一覧表示
sf data query \
  --query "SELECT Id, DeveloperName, MasterLabel, PluginType, Language FROM GenAiPluginDefinition ORDER BY MasterLabel" \
  --use-tooling-api \
  --target-org <username>
```

#### 既存アクション一覧の取得

```bash
# org内の全アクション（標準 + カスタム）を一覧表示
sf data query \
  --query "SELECT Id, DeveloperName, MasterLabel FROM GenAiFunctionDefinition ORDER BY MasterLabel" \
  --use-tooling-api \
  --target-org <username>
```

#### 既存の再利用可能な Apex @InvocableMethod の確認

```bash
# @InvocableMethod を持つApexクラスを検索
sf data query \
  --query "SELECT Id, Name FROM ApexClass WHERE Name LIKE '%Controller%' OR Name LIKE '%Action%'" \
  --use-tooling-api \
  --target-org <username>
```

#### 既存エージェントの構成確認

```bash
# エージェント一覧とタイプ
sf data query \
  --query "SELECT Id, DeveloperName, MasterLabel, Type, AgentType FROM BotDefinition ORDER BY CreatedDate DESC" \
  --target-org <username>
```

### 16-4. 標準アクションの代表例（参考）

以下はEmployee Agentで利用可能な標準アクションの代表例。orgのエディション・設定により利用可能なアクションは異なる。

| アクション名 | 説明 |
|---|---|
| Draft or Revise Email | 会話コンテキストを使ってメール作成・修正 |
| Get Record Details | 特定レコードの詳細情報を取得 |
| Identify Object by Name | オブジェクトを名前で特定 |
| Identify Record by Name | レコードを名前で特定 |
| Query Records | 条件に合う複数レコードを検索 |
| Query Records with Aggregate | 集計クエリ（COUNT等） |
| Summarize Record | レコードの要約を生成 |
| Update Record | レコードを更新 |
| Answer Questions with Knowledge | ナレッジ記事/Data LibraryベースのQ&A |

### 16-5. Apex Actionのベストプラクティス

| 観点 | ガイドライン |
|---|---|
| **ラベル・説明** | Atlas推論エンジンがアクション選択に使う。明確かつ具体的に記述する |
| **再利用性** | 汎用的に設計し、スキーマに密結合しない。複数エージェント/トピックで再利用可能にする |
| **セキュリティ** | `with sharing` を使用。必要最小限のフィールドのみ返す。パブリックアクションではSOQLを制限 |
| **エラーハンドリング** | try-catchでユーザーフレンドリーなメッセージを返す。Database クラスで部分的処理に対応 |
| **パフォーマンス** | バルク化、SOQLフィルタリング、非同期処理（Queueable）を活用。大規模アクションは分割 |
| **依存関係** | 複数アクションの実行順序はtopic instructionsと変数で制御。決定論的処理が必要なら複合アクション（1つのApex/Flowに統合） |

---

## 17. Agent Script（.agent ファイル）によるエージェント構築

### 17-0. 概要と推奨方針

Agent Script は Agentforce エージェントを宣言的に定義する DSL。`.agent` 拡張子のファイルを直接編集し、`sf agent publish authoring-bundle` でデプロイする。

**Agent Script を第一選択として使う。** 従来の GenAiPlugin/GenAiFunction XML メタデータ手動構築よりも、Agent Script の方が：
- 1ファイルでエージェント全体を見通せる
- トピック・アクション・変数・推論ロジックを一元管理できる
- `validate` → `publish` のワークフローで安全にデプロイできる
- 自然言語プロンプト（`|`）と決定論的ロジック（`->`）を混在できる

**制限**: CLI/Agent Script から作成されるのは常に **ExternalCopilot（Service Agent）**。Employee Agent（InternalCopilot）は引き続きUIで作成し、retrieveでローカル管理する。

> 完全な構文リファレンスは `docs/concepts/agent-script-syntax-reference.md` を参照。

### 17-1. ワークフロー

```
# 新規作成
sf agent generate agent-spec → specs/agentSpec.yaml を編集
sf agent generate authoring-bundle → .agent ファイル生成
.agent ファイルを編集（トピック・アクション・指示を追加）
sf agent validate authoring-bundle → コンパイル検証
sf agent publish authoring-bundle → org にパブリッシュ

# 既存エージェントの更新
.agent ファイルを直接編集
sf agent validate authoring-bundle → 検証
sf agent publish authoring-bundle → パブリッシュ
```

**依存リソース（Apex/Flow）を変更した場合は、`publish` の前に `sf project deploy start` で先にデプロイする。**

### 17-2. ファイル構造

```
force-app/main/default/aiAuthoringBundles/<BundleName>/
├── <BundleName>.bundle-meta.xml    # bundleType: AGENT
└── <BundleName>.agent              # Agent Script 本体
```

`bundle-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<AiAuthoringBundle xmlns="http://soap.sforce.com/2006/04/metadata">
  <bundleType>AGENT</bundleType>
</AiAuthoringBundle>
```

### 17-3. .agent ファイルの基本構造

```
config:                         # エージェント設定（必須）
variables:                      # 変数定義
system:                         # システム指示・メッセージ
language:                       # ロケール
connections:                    # Omni-Channel接続（@utils.escalate 用）
start_agent topic_selector:     # エントリポイント（必須）
topic <name>:                   # トピック定義（複数可）
```

インデント: **スペースのみ**（タブ不可）、3スペース推奨。

### 17-4. config ブロック

```yaml
config:
    developer_name: "MyAgent"
    agent_label: "エージェント表示名"
    agent_type: "AgentforceEmployeeAgent"   # または未指定（Service Agent）
    description: "エージェントの説明"
    default_agent_user: "NEW AGENT USER"    # Service Agent向け
```

### 17-5. variables ブロック

```yaml
variables:
    # mutable: 読み書き可能（アクション結果の保持等）
    account_id: mutable string = ""
        description: "対象取引先ID"
    result_data: mutable object = {}
        description: "取得結果"
    is_verified: mutable boolean = False
        description: "認証済みフラグ"

    # linked: 読み取り専用（セッション情報等の外部ソース紐付け）
    EndUserId: linked string
        source: @MessagingSession.MessagingEndUserId
        description: "MessagingEndUser Id"
```

**型一覧**: `string`, `number`, `integer`, `long`, `boolean`, `object`, `id`, `date`, `datetime`, `time`, `currency`, `list[<type>]`

**参照方法**:
- ロジック内: `@variables.name`
- プロンプト内: `{!@variables.name}`
- 代入（`->` 内）: `set @variables.name = "value"`

### 17-6. アクション定義（重要）

トピック内の `actions:` ブロックで定義。`target:` でApex/Flow/Prompt Templateを指定。

```yaml
topic order_management:
    description: "注文管理"

    actions:
        get_order_status:
            description: "注文ステータスを取得"
            inputs:
                order_id: string
                    description: "注文ID"
                    is_required: True
            outputs:
                status: string
                    description: "注文ステータス"
                items: list[object]
                    description: "商品リスト"
                    complex_data_type_name: "lightning__recordInfoType"
            target: "flow://GetOrderStatus"

        analyze_data:
            description: "データ分析レポートを生成"
            inputs:
                data_json: string
                    description: "分析対象データ（JSON）"
            outputs:
                report: string
                    description: "分析レポート"
            target: "apex://DataAnalysisService"

        generate_summary:
            description: "サマリーを生成"
            inputs:
                "Input:email": string
                    description: "メールアドレス"
                    is_required: True
            outputs:
                promptResponse: string
                    description: "生成結果"
                    is_used_by_planner: True
            target: "generatePromptResponse://GenerateSummaryTemplate"
```

**ターゲットタイプ**:

| タイプ | 構文 | 対象 |
|--------|------|------|
| Flow | `"flow://FlowApiName"` | Autolaunched Flow |
| Apex | `"apex://ClassName"` | `@InvocableMethod` を持つApexクラス |
| Prompt Template | `"generatePromptResponse://TemplateName"` | Prompt Builder テンプレート |

**アクションオプション**:
- `require_user_confirmation: True` — 実行前にユーザー確認
- `include_in_progress_indicator: True` — 処理中インジケーター表示
- `progress_indicator_message: "検索中..."` — インジケーターメッセージ

### 17-7. reasoning ブロック（推論ロジック）

#### プロンプトモード（`|`）— LLMに渡す自然言語
```yaml
reasoning:
    instructions: |
        ユーザーの注文に関する質問に回答してください。
```

#### ロジックモード（`->`）— 決定論的処理
```yaml
reasoning:
    instructions: ->
        if @variables.account_id == "":
            | 取引先IDを教えてください。
        else:
            run @actions.get_data
                with account_id = @variables.account_id
                set @variables.result_data = @outputs.result

            | データを取得しました: {!@variables.result_data}
```

#### reasoning.actions（LLMが状況に応じて選択するツール）
```yaml
reasoning:
    instructions: |
        ユーザーの要望に応じてツールを使ってください。
    actions:
        # LLMがスロットフィル（... = 会話から値を抽出）
        search: @actions.search_records
            with keyword = ...
            set @variables.result = @outputs.records

        # 固定値バインド
        get_japanese: @actions.get_records
            with language = "ja"

        # 変数バインド
        analyze: @actions.analyze_data
            available when @variables.result != ""
            with data = @variables.result
            set @variables.analysis = @outputs.report

        # アクションチェーン（コールバック）
        create_and_notify: @actions.create_record
            with name = ...
            set @variables.record_id = @outputs.id
            run @actions.send_notification
                with record_id = @variables.record_id

        # トピック遷移
        go_to_support: @utils.transition to @topic.support
            description: "サポートトピックへ遷移"
            available when @variables.needs_support == True

        # 変数収集
        collect_info: @utils.setVariables
            description: "ユーザー情報を収集"
            with user_name = ...
            with email = ...

        # エスカレーション
        escalate_to_human: @utils.escalate
            description: "人間のエージェントに転送"
```

**入力バインディングパターン**:

| パターン | 構文 | 説明 |
|----------|------|------|
| LLMスロットフィル | `with param = ...` | LLMが会話から値を抽出 |
| 固定値 | `with param = "value"` | 常に固定値 |
| 変数バインド | `with param = @variables.name` | 変数の現在値を渡す |

### 17-8. `available when` ガード

reasoning.actions 内のアクションに条件を付けて、LLMからの可視性を制御:

```yaml
actions:
    execute_transfer: @actions.execute_transfer
        available when @variables.validation_passed and @variables.amount > 0

    view_details: @utils.transition to @topic.details
        available when @variables.record_id != ""
```

条件が満たされない場合、そのアクションはLLMのツール一覧に表示されない。

### 17-9. before_reasoning / after_reasoning

推論の前後に毎リクエスト実行される決定論的処理ブロック。`|`（パイプ）は使用不可。

```yaml
topic main:
    description: "メイントピック"

    before_reasoning:
        run @actions.check_session
            set @variables.session_valid = @outputs.is_valid

    reasoning:
        instructions: ...

    after_reasoning:
        set @variables.turn_count = @variables.turn_count + 1
        if @variables.should_redirect:
            transition to @topic.next_step
```

### 17-10. @utils ビルトイン一覧

| ユーティリティ | 用途 | 使用箇所 |
|---|---|---|
| `@utils.transition to @topic.<name>` | トピック遷移（一方向） | reasoning.actions / `->` 内の `transition to` |
| `@utils.setVariables` | LLMに会話から変数値を抽出させる | reasoning.actions |
| `@utils.escalate` | 人間エージェントへのエスカレーション | reasoning.actions（`connections` 必須） |

### 17-11. CLIコマンド一覧

```bash
# バンドル生成（agentSpecから）
sf agent generate authoring-bundle \
  --spec specs/agentSpec.yaml \
  --name "表示名" \
  --api-name ApiName \
  --target-org <username>

# バンドル生成（specなし — 空テンプレート）
sf agent generate authoring-bundle \
  --no-spec \
  --name "表示名" \
  --api-name ApiName \
  --target-org <username>

# コンパイル検証
sf agent validate authoring-bundle \
  --api-name ApiName \
  --target-org <username>

# パブリッシュ（Bot + BotVersion + GenAiXX 自動生成）
sf agent publish authoring-bundle \
  --api-name ApiName \
  --target-org <username>

# 対話プレビュー
sf agent preview \
  --api-name ApiName \
  --target-org <username>

# 有効化・無効化
sf agent activate --api-name ApiName --target-org <username>
sf agent deactivate --api-name ApiName --target-org <username>

# ブラウザで確認
sf org open agent --api-name ApiName --target-org <username>
```

### 17-12. 実装パターン例

#### パターン1: データ取得 → 分析（Apex + Prompt Template）

```
config:
    developer_name: "AccountAnalyst"
    agent_label: "取引先分析エージェント"
    description: "取引先データを収集し分析レポートを生成"

variables:
    account_id: mutable id = ""
        description: "対象取引先ID"
    raw_data: mutable string = ""
        description: "取得した生データ（JSON）"

system:
    instructions: "あなたは取引先データの分析を支援するエージェントです。日本語で応答してください。"
    messages:
        welcome: "取引先分析エージェントです。どの取引先を分析しますか？"
        error: "エラーが発生しました。再度お試しください。"

language:
    default_locale: "ja"

start_agent topic_selector:
    description: "ユーザーリクエストを適切なトピックにルーティング"

    reasoning:
        instructions: |
            ユーザーのメッセージに最も合うトピックを選択してください。
        actions:
            go_to_analysis: @utils.transition to @topic.account_analysis
                description: "取引先の分析・サマリー生成"

topic account_analysis:
    description: "取引先データの収集と分析レポート生成"

    actions:
        get_account_data:
            description: "取引先の商談・ケース・コンタクト情報を一括取得"
            inputs:
                accountId: id
                    description: "取引先ID"
                    is_required: True
            outputs:
                analysisData: string
                    description: "取得データ（JSON）"
            target: "apex://AccountDataCollector"

        generate_report:
            description: "取得データからAI分析レポートを生成"
            inputs:
                "Input:analysisData": string
                    description: "分析対象データ（JSON）"
                    is_required: True
            outputs:
                promptResponse: string
                    description: "生成されたレポート"
                    is_used_by_planner: True
                    is_displayable: True
            target: "generatePromptResponse://AccountAnalysisReport"

    reasoning:
        instructions: ->
            if @variables.account_id == "":
                | 分析対象の取引先名またはIDを教えてください。
            else:
                run @actions.get_account_data
                    with accountId = @variables.account_id
                    set @variables.raw_data = @outputs.analysisData

                | データを取得しました。レポートを生成します。

                run @actions.generate_report
                    with "Input:analysisData" = @variables.raw_data

                | {!@outputs.promptResponse}

        actions:
            set_account: @utils.setVariables
                description: "取引先IDを設定"
                with account_id = ...
```

---

## 18. その他のベストプラクティス

- **ExternalCopilot（Service Agent）は Agent Script（Authoring Bundle）方式で構築する**（推奨。GenAiPlugin/GenAiFunction XML の手動構築は代替手段）
- **Employee Agent（InternalCopilot）はUIから作成し、retrieveでローカル管理する**（CLIでは作成不可）
- **Agent Script の `.agent` ファイルを直接編集してOK**（UIのAgent Builderと等価）
- **依存リソース（Apex/Flow）変更時は `sf project deploy start` を先に実行**してから `sf agent publish authoring-bundle`
- **`sf agent validate authoring-bundle` でコンパイル検証してから publish**
- **GenAiFunctionの invocationTarget はApexクラス名を指定**（IDではない）
- **agentSpec.yaml のトピック名は英語で記述**（日本語だとLLM生成が失敗することがある）
- **BotVersionは手動デプロイしない**（`sf agent create` または `sf agent publish` の内部APIを使う）
- **InternalCopilotのトピック/アクション追加はUIで行い、retrieveでローカル保存**
- Apex @InvocableMethod を追加したら Agent Builder の「呼び出し可能なメソッド」タブから手動登録が必要（Agent Script方式では `target: "apex://ClassName"` で直接指定可能）

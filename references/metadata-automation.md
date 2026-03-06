# Metadata: Apex, Flow & Automation Reference

## 概要
Apexクラス・トリガー、フロー、ワークフロー・プロセスビルダーのRetrieve・変更・Deployパターン。

---

## 1. Apex クラス

### 1-1. 既存ApexクラスのRetrieveとDeploy

```bash
# 既存クラスを取得
sf project retrieve start \
  --metadata "ApexClass:<ClassName>" \
  --target-org <username>

# 全Apexクラスを一覧
sf org list metadata \
  --metadata-type ApexClass \
  --target-org <username>
```

ファイルパス: `force-app/main/default/classes/<ClassName>.cls` と `<ClassName>.cls-meta.xml`

### 1-2. Apexクラスのメタファイル

```xml
<!-- <ClassName>.cls-meta.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<ApexClass xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>63.0</apiVersion>
    <status>Active</status>
</ApexClass>
```

### 1-3. Apexクラスのサンプル

```apex
// <ClassName>.cls
public class ClassName {
    public static void doSomething() {
        // 処理内容
    }
}
```

### 1-4. DeployとApexテスト実行

```bash
# デプロイ時にテストを実行（本番デプロイでは必須）
sf project deploy start \
  --metadata "ApexClass:<ClassName>" \
  --test-level RunLocalTests \
  --target-org <username>

# テストレベルの選択肢
# NoTestRun: テストを実行しない（Sandbox向け）
# RunLocalTests: ローカルのテストを実行（本番向け）
# RunAllTestsInOrg: 全テストを実行
```

**⚠️ 本番へのApexデプロイは75%以上のコードカバレッジが必要。**

### 1-5. 匿名Apex実行（即時実行・デプロイ不要）

```bash
# ファイルから匿名Apexを実行
sf apex run \
  --file ~/my-apex-script.apex \
  --target-org <username>

# インラインで実行（単純な処理向け）
echo "System.debug('Hello World');" | sf apex run --target-org <username>
```

---

## 2. Apex トリガー

### 2-1. Retrieve

```bash
sf project retrieve start \
  --metadata "ApexTrigger:<TriggerName>" \
  --target-org <username>
```

ファイルパス: `force-app/main/default/triggers/<TriggerName>.trigger` と `<TriggerName>.trigger-meta.xml`

### 2-2. トリガーのメタファイルとサンプル

```xml
<!-- <TriggerName>.trigger-meta.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<ApexTrigger xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>63.0</apiVersion>
    <status>Active</status>
</ApexTrigger>
```

```apex
// <TriggerName>.trigger
trigger TriggerName on ObjectName__c (before insert, before update, after insert) {
    if (Trigger.isBefore) {
        // 処理
    }
}
```

### 2-3. トリガーの無効化（メタデータで管理）

```xml
<!-- status を Inactive に変更 -->
<ApexTrigger xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>63.0</apiVersion>
    <status>Inactive</status>
</ApexTrigger>
```

---

## 3. フロー（Flow）

### 3-1. 既存フローのRetrieve

```bash
# 特定フローを取得
sf project retrieve start \
  --metadata "Flow:<FlowAPIName>" \
  --target-org <username>

# フロー一覧
sf org list metadata \
  --metadata-type Flow \
  --target-org <username>
```

ファイルパス: `force-app/main/default/flows/<FlowAPIName>.flow-meta.xml`

### 3-2. フローの有効化・無効化

フローの`status`フィールドを変更してDeploy：

```xml
<!-- 有効化 -->
<status>Active</status>

<!-- 無効化 -->
<status>Inactive</status>
```

**注意**: フローのバージョン管理は複雑。既存フローを修正する場合は必ずRetrieveしてから編集。

### 3-3. フローのDeploy

```bash
sf project deploy start \
  --metadata "Flow:<FlowAPIName>" \
  --target-org <username>
```

### 3-4. フローの実行確認（Apex経由）

```apex
// 匿名Apexでフローをテスト実行
Map<String, Object> inputs = new Map<String, Object>();
inputs.put('inputVar', 'value');
Flow.Interview myFlow = Flow.Interview.createInterview('FlowAPIName', inputs);
myFlow.start();
```

---

## 4. ワークフロー（Workflow / Process Builder）

**推奨**: 新規作成はフロー（Flow）を使用。ワークフローとプロセスビルダーはSalesforceの推奨から外れている。

### 4-1. ワークフローのRetrieve

```bash
sf project retrieve start \
  --metadata "Workflow:<ObjectName>" \
  --target-org <username>
```

ファイルパス: `force-app/main/default/workflows/<ObjectName>.workflow-meta.xml`

### 4-2. ワークフロールールの無効化

```xml
<WorkflowRule>
    <fullName>RuleName</fullName>
    <active>false</active>
    <!-- その他の設定 -->
</WorkflowRule>
```

---

## 5. Validate → Deploy（共通）

```bash
# 必ずValidate先行
sf project deploy validate \
  --source-dir force-app \
  --test-level RunLocalTests \
  --target-org <username>

# Deploy
sf project deploy start \
  --source-dir force-app \
  --test-level RunLocalTests \
  --target-org <username>
```

---

## 6. よくあるエラーと対処

| エラー | 原因 | 対処 |
|---|---|---|
| `Code coverage below 75%` | テストカバレッジ不足 | テストクラスを追加・修正してカバレッジを上げる |
| `Compile error` | Apexコンパイルエラー | エラー行を確認してコードを修正 |
| `Flow version conflict` | フローバージョン不一致 | 最新バージョンをRetrieveして再編集 |
| `Cannot activate flow` | フロー設定に問題 | フロー定義のXMLを検証 |
| `SOQL/SOSL in loops` | Apexのガバナ制限違反 | ループ外でクエリを実行するよう修正 |
| `System.LimitException` | ガバナ制限超過 | バルク処理に対応したコードに修正 |

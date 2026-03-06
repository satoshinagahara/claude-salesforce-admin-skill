# Production Safety Protocol

## 概要
本番（Production）環境での操作前に必ず実行する安全確認プロトコル。
このファイルの手順をスキップして本番操作を行ってはならない。

---

## ⚠️ 本番操作の前提ルール

1. **Validate必須**: メタデータ変更は必ずDeployの前にValidateを実行し、結果をユーザーに提示してから承認を得る
2. **バックアップ必須**: 破壊的変更（削除・大量更新）の前は必ずRetrieve/SOQLで現状を保存
3. **影響範囲の提示必須**: 変更前に影響を受けるユーザー・オブジェクト・プロセスをユーザーに提示
4. **承認の取得必須**: ユーザーが「実行してください」「proceed」など明示的な承認を出すまで実行しない

---

## 1. データ操作（DML）前の確認チェックリスト

### 1-1. 単件更新・削除

実行前にユーザーへ提示する内容：

```
【確認事項】
対象オブジェクト: <ObjectName>
対象レコードID: <RecordId>
操作: <Create/Update/Delete>
変更内容: <フィールド名: 変更前 → 変更後>
ロールバック: <可能（削除のみ不可）>

上記の操作を本番環境に実行してよろしいですか？
```

### 1-2. バルク操作前

```
【確認事項】
対象オブジェクト: <ObjectName>
操作: <Insert/Update/Upsert/Delete>
対象件数: <N件>
データソース: <CSVファイルパス>
ロールバック: <Delete操作は不可。Update/Insertは手動で戻し可>

バルク操作の前にデータのバックアップSOQLを実行します。
完了後、本番環境への実行をご承認ください。
```

バックアップSOQL例：
```bash
# 対象レコードの現状を保存
sf data query \
  --query "SELECT Id, <変更対象フィールド> FROM <ObjectName> WHERE <条件>" \
  --result-format csv \
  --target-org <username> > backup_$(date +%Y%m%d_%H%M%S).csv
```

---

## 2. メタデータ操作前の確認チェックリスト

### 2-1. 追加・変更操作（Create/Update）

```
【確認事項】
操作タイプ: <メタデータの追加/変更>
対象メタデータ: <タイプ: 名前>
変更内容:
  - <変更点1>
  - <変更点2>
影響範囲: <影響を受けるユーザー/プロファイル/プロセス>
ロールバック: <可能（Deployで元に戻せる）>

Validateを実行してから結果をご確認ください。
```

### 2-2. 削除操作（Delete）

**削除は特に慎重に。フィールド削除後のデータは復元不可。**

```
【⚠️ 警告】本番環境での削除操作
対象: <削除するメタデータの名前>
影響: <削除により失われるデータ・機能>
依存関係: <このメタデータを参照しているもの一覧>
ロールバック: 【不可】削除したデータ・定義は復元できません

本当に削除を実行しますか？
削除の前に依存関係の解消が必要な場合は先に対処します。
```

依存関係の確認：
```bash
# Salesforce UIのMetadata Dependency APIで確認
# または、取得したXMLファイルでgrepを使用
grep -r "<ObjectName>.<FieldName>" force-app/
```

---

## 3. 権限変更前の確認チェックリスト

```
【確認事項】
対象: <プロファイル名 / 権限セット名>
変更内容:
  - <追加する権限>
  - <削除する権限>
影響するユーザー数: <N名>（SOQLで確認済み）
ロールバック: 可能（元のメタデータをDeployで戻せる）

影響ユーザーの確認SOQLを実行しますか？
```

影響ユーザー確認：
```bash
# 権限セットが割り当てられているユーザー数
sf data query \
  --query "SELECT COUNT(Id), Assignee.Profile.Name FROM PermissionSetAssignment WHERE PermissionSet.Name = '<PermSetName>' GROUP BY Assignee.Profile.Name" \
  --target-org <username>

# プロファイルユーザー数
sf data query \
  --query "SELECT COUNT(Id) FROM User WHERE Profile.Name = '<ProfileName>' AND IsActive = true" \
  --target-org <username>
```

---

## 4. Apex/フロー変更前の確認チェックリスト

```
【確認事項】
対象: <Apexクラス名 / フロー名>
変更内容: <変更の概要>
テストカバレッジ: <変更後のカバレッジ（本番デプロイは75%以上必須）>
副作用: <自動起動フロー・トリガーの場合、起動条件の確認>
ロールバック: 可能（前バージョンのコードを再Deployで戻せる）

Validateとテスト実行を行ってから結果をご確認ください。
```

---

## 5. 緊急時のロールバック手順

### メタデータのロールバック

```bash
# バックアップとして保存していたXMLをDeployして元に戻す
sf project deploy start \
  --metadata "<MetadataType>:<Name>" \
  --target-org <username>
```

### データ変更のロールバック（バルク更新の場合）

```bash
# バックアップCSVを使って元の値に戻す
sf data upsert bulk \
  --sobject <ObjectName> \
  --file backup_<timestamp>.csv \
  --external-id Id \
  --target-org <username>
```

### 削除レコードの復元（30日以内）

```bash
# 削除済みレコードの確認
sf data query \
  --query "SELECT Id, Name FROM <ObjectName> WHERE IsDeleted = true ALL ROWS" \
  --target-org <username>

# ゴミ箱から復元（Salesforce UIを使用するか、undelete SOQLを匿名Apexで実行）
echo "Database.undelete(new List<Id>{'<RecordId>'})" | sf apex run --target-org <username>
```

---

## 6. 禁止事項（本番環境）

- ユーザーの明示的承認なしにDeployを実行すること
- Validateをスキップしてデプロイすること
- バックアップなしにバルク削除・バルク更新を実行すること
- システム管理者プロファイルを直接変更すること
- Apexトリガー・フローを無効化せずに大量データ変更を行うこと

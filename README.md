# Claude Code — Salesforce Admin Skill


Claude Code 用の Salesforce 管理スキルです。Salesforce CLI (`sf`) を使って、データ操作・メタデータ操作・LWC作成などを自然言語で指示するだけで実行できます。


## ⚠️ 免責事項 / Disclaimer

> **本スキルの使用により生じたいかなる損害についても、作者は一切の責任を負いません。**
>
> **The author assumes no responsibility for any damages arising from the use of this skill.**

本スキルは Salesforce CLI を通じて Salesforce 組織に対して実際の操作を実行します。使用方法の誤りやスキルの不備によって生じた、データ損失・設定ミス・意図しないメタデータ変更・本番環境への影響等、いかなる損害についても作者は責任を負いません。特に本番環境での使用においては、操作内容と影響範囲をご自身で十分に確認の上、**自己責任**でご利用ください。

本スキルは MIT ライセンスのもとで無償提供されますが、**サポートは提供しません**。Issue・PR への対応、動作保証、特定環境での互換性保証はいたしません。本スキルは「現状のまま（as-is）」で提供されます。

This skill executes actual operations against your Salesforce organization via Salesforce CLI. The author is not liable for any damages — including data loss, misconfiguration, unintended metadata changes, or impacts to production environments — whether caused by user error or defects in this skill. Use in production environments is entirely **at your own risk**.

This skill is provided free of charge under the MIT License, but **no support is provided**. There is no guarantee of response to Issues or PRs, no warranty of operation, and no guarantee of compatibility with specific environments. This skill is provided **"as-is"**.

---




## ⚠️ 利用環境について（重要）


> **このスキルは Developer Edition または Sandbox 環境での利用を推奨します。**


本スキルは Salesforce CLI を通じて org に直接操作を行います。**本番環境（Production）での使用は十分な理解と慎重な運用が必要です。**


| 環境 | 推奨度 | 備考 |
|---|---|---|
| **Developer Edition** | ✅ 推奨 | 無料で取得可能。本番データへの影響なし |
| **Sandbox** | ✅ 推奨 | 本番の設定を引き継いだテスト環境 |
| **本番 (Production)** | ⚠️ 上級者向け | 誤操作がビジネスに直結。自動安全確認が適用されるが要注意 |


Developer Edition org は [Salesforce 開発者登録](https://developer.salesforce.com/signup) から無料で取得できます。


---


## 主な機能


| 操作カテゴリ | できること |
|---|---|
| **データ操作 (DML)** | レコードの登録・更新・削除（単件／バルクCSV）|
| **カスタムオブジェクト・フィールド** | 作成・変更・削除・リレーション設定 |
| **ページレイアウト・レコードタイプ** | レイアウト変更・セクション追加・フィールド配置 |
| **権限セット・プロファイル** | FLS設定・オブジェクト権限・ユーザー割り当て |
| **Apex・フロー・自動化** | クラス作成・トリガー・フロー有効化/無効化 |
| **Lightning Web Component** | コンポーネント作成・デプロイ |


---


## 前提条件


1. [Salesforce CLI](https://developer.salesforce.com/tools/salesforcecli) (`sf`) がインストール済みであること
2. Salesforce org に **接続アプリケーション（Connected App）** が作成済みであること
3. JWT認証用の設定ファイル (`sf-config.json`) が準備済みであること


---


## Salesforce 側のセットアップ


このスキルは **JWT Bearer Flow** を使って Salesforce に接続します。以下の手順で一度だけセットアップが必要です。


### Step 1: RSA 鍵ペアの生成


ターミナルで以下を実行します：


```bash
# 作業ディレクトリを作成
mkdir -p ~/sf-jwt-keys && cd ~/sf-jwt-keys


# 秘密鍵と自己署名証明書を生成（有効期間365日）
openssl req -x509 -newkey rsa:2048 \
  -keyout server.key \

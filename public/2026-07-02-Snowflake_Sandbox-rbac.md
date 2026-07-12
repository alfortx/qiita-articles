---
title: SnowflakeのRBACを2層設計にした話 ― データベースロールとアカウントロールを使い分ける
tags:
  - Security
  - Terraform
  - RBAC
  - Snowflake
  - アクセス制御
private: false
updated_at: '2026-07-12T20:08:32+09:00'
id: f13900fe9d4f84653a2d
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## TL;DR

- Snowflake の RBAC は**データベースロール**と**アカウントロール**の2種類がある
- DB スコープの権限はデータベースロールに集約し、ウェアハウス等の DB 外権限はアカウントロールに持たせる**2層設計**がベストプラクティス
- 役割ロール（DEVELOPER / VIEWER など）は両方を継承するだけ ― 権限定義の重複がなくなってスッキリする

---

## 背景

Snowflake のサンドボックス環境を Terraform で構築していて、RBAC 設計でかなり悩んだ。

最初はすべての権限をアカウントロールに集約しようとしていたが、DB が増えるにつれて権限定義が爆発的に増えた。DB ごとに `GRANT SELECT ON ALL TABLES IN SCHEMA ... TO ROLE DEVELOPER_ROLE` を書き続ける羽目になり、「これは絶対スケールしない」と感じて設計を見直した。

調べると Snowflake には**データベースロール**という仕組みがあり、これを使うと DB スコープの権限を DB 側でカプセル化できる。

---

## 環境

- Snowflake Enterprise Edition
- Terraform + Snowflake Provider v2.14.0

---

## データベースロールとは

アカウントロール（`CREATE ROLE`）は皆さんご存知のロール。一方、データベースロール（`CREATE DATABASE ROLE`）は以下の特徴を持つ。

| 項目 | データベースロール | アカウントロール |
|---|---|---|
| スコープ | 自 DB 内のオブジェクトのみ | アカウント全体 |
| ユーザーへの直接付与 | ❌ 不可 | ✅ 可 |
| セッションでアクティブ化 | ❌ 不可 | ✅ 可 |
| アカウントロールへの継承 | ✅ 可 | — |
| Terraform リソース | `snowflake_database_role` | `snowflake_role` |

重要なのは「**ユーザーへの直接付与ができない**」点。データベースロールは必ずアカウントロールに継承させてから使う。

---

## 2層設計の考え方

```
【第1層: データベースロール（DB スコープの権限を集約）】
SANDBOX_DB.WORK_READ  ← SELECT on tables/views in SANDBOX_DB.WORK
SANDBOX_DB.WORK_WRITE ← INSERT/UPDATE/DELETE/SELECT on SANDBOX_DB.WORK
RAW_DB.COVID19_READ   ← SELECT on tables in RAW_DB.COVID19
...（DB ごとに定義）

【第2層: アカウントロール（役割ロールが DB ロール + WH 権限を継承）】
DEVELOPER_ROLE
 ├── SANDBOX_DB.WORK_WRITE（DB ロールを継承）
 ├── RAW_DB.COVID19_READ（DB ロールを継承）
 ├── FR_WH_SANDBOX_OPERATE（WH 操作権限ロール）
 └── ...

VIEWER_ROLE
 ├── SANDBOX_DB.WORK_READ（DB ロールを継承）
 ├── RAW_DB.COVID19_READ（DB ロールを継承）
 ├── FR_WH_SANDBOX_USE（WH 使用権限ロール）
 └── ...
```

ポイントは「**DB 外の権限（ウェアハウス・インテグレーション等）はアカウントロールが持つ**」こと。これで権限の責任範囲が明確になる。

---

## Terraform での実装

### データベースロールの定義

```hcl
# SANDBOX_DB 内の読み取りロール
resource "snowflake_database_role" "sandbox_work_read" {
  provider = snowflake.sysadmin
  database = "SANDBOX_DB"
  name     = "WORK_READ"
  comment  = "SANDBOX_DB.WORK の読み取り専用"
}

# SANDBOX_DB.WORK 内のテーブル・ビューに SELECT を付与
resource "snowflake_grant_privileges_to_database_role" "sandbox_work_read_select" {
  provider          = snowflake.securityadmin
  database_role_name = "SANDBOX_DB.WORK_READ"
  privileges        = ["SELECT"]

  on_schema_object {
    future {
      object_type_plural = "TABLES"
      in_schema          = "SANDBOX_DB.WORK"
    }
  }
}
```

### アカウントロール（役割ロール）への継承

```hcl
# VIEWER_ROLE に DB ロールを継承させる
resource "snowflake_grant_database_role" "viewer_sandbox_work_read" {
  provider           = snowflake.securityadmin
  database_role_name = "SANDBOX_DB.WORK_READ"
  parent_role_name   = "VIEWER_ROLE"
}
```

### ウェアハウス権限は機能ロール（FR_*）として分離

```hcl
# WH の OPERATE 権限を持つ機能ロール
resource "snowflake_role" "fr_wh_sandbox_operate" {
  provider = snowflake.useradmin
  name     = "FR_WH_SANDBOX_OPERATE"
}

resource "snowflake_grant_privileges_to_role" "fr_wh_sandbox_operate" {
  provider   = snowflake.securityadmin
  role_name  = "FR_WH_SANDBOX_OPERATE"
  privileges = ["OPERATE", "USAGE"]

  on_account_object {
    object_type = "WAREHOUSE"
    object_name = "SANDBOX_WH"
  }
}

# DEVELOPER_ROLE が WH 機能ロールを継承
resource "snowflake_role_grants" "developer_wh" {
  provider  = snowflake.securityadmin
  role_name = "FR_WH_SANDBOX_OPERATE"
  roles     = ["DEVELOPER_ROLE"]
}
```

---

## ハマったポイント

### CORTEX_USER などの組み込みロールはデータベースロールに付与できない

`SNOWFLAKE.CORTEX_USER` は `SNOWFLAKE` という別 DB の組み込みロールなので、**別 DB のデータベースロールには付与できない**（スコープ外）。

最初 `CORTEX_DB.CORTEX_USE`（自作のデータベースロール）に付与しようとしてエラーになった。

```
SQL access control error: Cannot grant a DATABASE ROLE to a different DB's DATABASE ROLE.
```

解決策は、`SNOWFLAKE.CORTEX_USER` を役割ロール（アカウントロール）に直接付与すること。

```hcl
# NG: データベースロールへの付与 → スコープ外でエラー
# GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO DATABASE ROLE CORTEX_DB.CORTEX_USE;

# OK: アカウントロールへ直接付与
resource "snowflake_grant_database_role" "cortex_user_to_developer" {
  provider           = snowflake.accountadmin
  database_role_name = "SNOWFLAKE.CORTEX_USER"
  parent_role_name   = "DEVELOPER_ROLE"
}
```

### `OR REPLACE` でデータベースロールを再作成するとシェアへの付与が外れる

Terraform で `CREATE OR REPLACE DATABASE ROLE` を実行すると DROP → CREATE になる。SnowflakeのShareにデータベースロールを付与していた場合、その付与が外れる。

本プロジェクトでは Share を使っていないので問題なかったが、共有環境では注意が必要。

---

## 設計の結果

新しい DB を追加するたびに **DB ロールだけ追加すればよく**、役割ロール（DEVELOPER/VIEWER）の定義には一切触れない。権限追加の影響範囲がDBに閉じるので、Terraform の diff が見通しやすくなった。

| Before（アカウントロール一本） | After（2層設計） |
|---|---|
| DB 追加のたびに DEVELOPER_ROLE / VIEWER_ROLE への GRANT を追記 | DB ロールを定義して GRANT DATABASE ROLE するだけ |
| 役割ロールの定義が際限なく膨らむ | 役割ロールは継承関係のみ（権限定義ゼロ） |
| どの DB の権限かが名前から分からない | `SANDBOX_DB.WORK_READ` と DB 名が権限名に含まれる |

---

## まとめ

Snowflake の RBAC は「アカウントロールだけで管理する」より、**DB スコープの権限はデータベースロールに集約 → 役割ロールが継承する 2 層設計**にするとスケールする。

最初は「ロールが 2 種類あってめんどくさい」と感じたが、慣れると「DB 内の話は DB ロールで閉じる」という境界が明確になって逆に楽になった。

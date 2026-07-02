---
title: Snowflakeビューの参照元テーブルの権限
tags:
  - SQL
  - RBAC
  - Snowflake
private: false
updated_at: '2026-07-02T21:26:15+09:00'
id: 6fafc0f0aa317b58df29
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## TL;DR

- Snowflake のビューは**所有者権限モデル**で動作する
- ビューに `SELECT` 権限があれば、**参照先テーブルに権限がなくてもクエリは成功する**


---

## 背景・課題

先日業務で、「ビューの参照しているテーブルの権限がなければ、クエリは失敗するのか」という問いにぶち当たった。あなたは即答できますか？私はできませんでした。ので、実験しました。

公式ドキュメント（[CREATE VIEW](https://docs.snowflake.com/en/sql-reference/sql/create-view)）には明確にこう書いてある。

> When you create a view and then grant privileges on that view to a role, the role can use the view even if the role does not have privileges on the underlying table(s) that the view accesses.

つまり、**ビュー経由ならテーブル権限は不要**。

実際にやってみた。

---

## 環境

- Snowflake Enterprise Edition

---

## 実証するシナリオ

次の2ロールを用意して、権限を意図的に非対称にする。

| ロール | 保持する権限 |
|---|---|
| `VIEW_OWNER` | `customer_table`・`customer_view` の所有者。テーブルに `SELECT` あり |
| `VIEW_CONSUMER` | `customer_view` の `SELECT` **のみ**。テーブルには権限なし |

`VIEW_CONSUMER` でビュー経由クエリ → 成功するか？
`VIEW_CONSUMER` でテーブル直接クエリ → 失敗するか？

---

## 実装手順

### 1. テスト専用のリソースを作成（SYSADMIN）

```sql
USE ROLE SYSADMIN;

CREATE DATABASE IF NOT EXISTS VIEW_TEST_DB;
CREATE SCHEMA  IF NOT EXISTS VIEW_TEST_DB.DEMO;

CREATE WAREHOUSE IF NOT EXISTS VIEW_TEST_WH
    WAREHOUSE_SIZE               = 'XSMALL'
    AUTO_SUSPEND                 = 60
    AUTO_RESUME                  = TRUE
    INITIALLY_SUSPENDED          = TRUE
    STATEMENT_TIMEOUT_IN_SECONDS = 180;
```

### 2. テスト用ロールを作成（SECURITYADMIN）

```sql
USE ROLE SECURITYADMIN;

CREATE ROLE IF NOT EXISTS VIEW_OWNER
    COMMENT = 'customer_table と customer_view の所有者';

CREATE ROLE IF NOT EXISTS VIEW_CONSUMER
    COMMENT = 'customer_view の SELECT のみ保持（テーブルへの権限なし）';

-- ロールを SYSADMIN 配下に配置
GRANT ROLE VIEW_OWNER    TO ROLE SYSADMIN;
GRANT ROLE VIEW_CONSUMER TO ROLE SYSADMIN;

-- 現ユーザーに付与（CURRENT_USER() は変数経由で渡す）
SET my_user = CURRENT_USER();
GRANT ROLE VIEW_OWNER    TO USER IDENTIFIER($my_user);
GRANT ROLE VIEW_CONSUMER TO USER IDENTIFIER($my_user);
```

### 3. 権限を付与（SYSADMIN）

```sql
USE ROLE SYSADMIN;

-- VIEW_OWNER：テーブル・ビューを作成できる最小セット
GRANT USAGE ON DATABASE  VIEW_TEST_DB          TO ROLE VIEW_OWNER;
GRANT USAGE ON SCHEMA    VIEW_TEST_DB.DEMO     TO ROLE VIEW_OWNER;
GRANT USAGE ON WAREHOUSE VIEW_TEST_WH          TO ROLE VIEW_OWNER;
GRANT CREATE TABLE ON SCHEMA VIEW_TEST_DB.DEMO TO ROLE VIEW_OWNER;
GRANT CREATE VIEW  ON SCHEMA VIEW_TEST_DB.DEMO TO ROLE VIEW_OWNER;

-- VIEW_CONSUMER：USAGE のみ。テーブルには何も付与しない
GRANT USAGE ON DATABASE  VIEW_TEST_DB          TO ROLE VIEW_CONSUMER;
GRANT USAGE ON SCHEMA    VIEW_TEST_DB.DEMO     TO ROLE VIEW_CONSUMER;
GRANT USAGE ON WAREHOUSE VIEW_TEST_WH          TO ROLE VIEW_CONSUMER;
```

### 4. テーブルとビューを作成（VIEW_OWNER）

```sql
USE ROLE VIEW_OWNER;
USE WAREHOUSE VIEW_TEST_WH;

CREATE OR REPLACE TABLE VIEW_TEST_DB.DEMO.customer_table (
    customer_id  INT,
    name         STRING,
    credit_limit NUMBER    -- 機微列（VIEW_CONSUMER に直接は見せない）
);

INSERT INTO VIEW_TEST_DB.DEMO.customer_table VALUES
    (1, '田中太郎', 1000000),
    (2, '佐藤花子', 2000000),
    (3, '鈴木一郎',  500000),
    (4, '高橋美咲', 1500000),
    (5, '伊藤健太', 3000000);

CREATE OR REPLACE VIEW VIEW_TEST_DB.DEMO.customer_view AS
    SELECT customer_id, name, credit_limit
    FROM VIEW_TEST_DB.DEMO.customer_table;

-- ビューの SELECT だけを VIEW_CONSUMER に付与（テーブルへの GRANT はしない）
GRANT SELECT ON VIEW VIEW_TEST_DB.DEMO.customer_view TO ROLE VIEW_CONSUMER;
```

### 5. 検証（VIEW_CONSUMER）

```sql
-- セカンダリーロールを無効にして、VIEW_CONSUMER の権限だけで評価する
USE SECONDARY ROLES NONE;
USE ROLE VIEW_CONSUMER;
USE WAREHOUSE VIEW_TEST_WH;

-- テスト1：ビュー経由（期待値：成功）
SELECT * FROM VIEW_TEST_DB.DEMO.customer_view;

-- テスト2：テーブル直接参照（期待値：権限エラー）
SELECT * FROM VIEW_TEST_DB.DEMO.customer_table;
```

---

## 実行結果

**テスト1（ビュー経由）→ 成功**

```
 CUSTOMER_ID | NAME     | CREDIT_LIMIT
-------------+----------+--------------
           1 | 田中太郎 |      1000000
           2 | 佐藤花子 |      2000000
           3 | 鈴木一郎 |       500000
           4 | 高橋美咲 |      1500000
           5 | 伊藤健太 |      3000000
```

**テスト2（テーブル直接参照）→ 権限エラー**

```
002003 (42S02): SQL compilation error:
Object 'VIEW_TEST_DB.DEMO.CUSTOMER_TABLE' does not exist or not authorized.
```

想定通りの結果。

---

## 注意点

### USE SECONDARY ROLES NONE を忘れると検証にならない

Snowflake はデフォルトでセカンダリーロールが有効で、ユーザーに付与された全ロールの権限が合わさって評価される。

これを忘れると `VIEW_CONSUMER` に切り替えたつもりでも SYSADMIN 権限が効いてしまい、「テーブルも見えてしまう」という誤った結果になる。

```sql
-- 必ずセカンダリーロールを無効化してから検証する
USE SECONDARY ROLES NONE;
USE ROLE VIEW_CONSUMER;
```

---

## まとめ

Snowflake のビューは **所有者権限モデル**で動作する。

**ビューにSELECT権限があれば、参照先テーブルへの権限がなくてもクエリは成功する**

「テーブルを隠してビューだけ公開する」という設計は Snowflake では正しく機能する。

Snowflakeではビューの所有者＝作成者であり、「作成時にビューの処理が成功する必要がある＝所有者に参照先テーブルの権限がある」が成り立つため、この仕様は自然。
逆に、ビューの所有権を別のロールに移したりしたら事故が起きそう。

「オブジェクトの所有権は作成者が持つ」の原理を守ることの大切さを学んだ。

なお、対象のビューがアカウント間のデータシェア対象であった場合、別の権限が必要（実務ではここでこけた）。この件はまた記事にしたいです。
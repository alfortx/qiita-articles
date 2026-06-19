---
title: 'Row Access Policyを試してみた'
tags:
  - Snowflake
  - セキュリティ
  - SQL
  - データ基盤
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## TL;DR

- 先行記事は山ほどありますが改めて調べてみたので投稿します
- Snowflake の **Row Access Policy（行アクセスポリシー）** を使うと、同じテーブルでも参照ロールによって見える行を動的に絞り込める
- ポリシーの判定には `CURRENT_ROLE()` ではなく **`IS_ROLE_IN_SESSION()`** を使う（ロール階層を考慮するため）
- **テーブル所有者とポリシー管理者を分離する**のがベストプラクティス

---

## 環境

- Snowflake Enterprise Edition（Row Access Policy は Enterprise 以上が必要）
- SnowCLI / Snowsight での SQL 実行

---

## 背景・課題

RBAC（Role-Based Access Control）でテーブルへの SELECT 権限を制御する場合、通常は「テーブルごとにアクセス可否」しか制御できません。しかし「HR 部門は全社員データを見られるが、SALES 部門は自部門の社員データしか見られない」といった **行レベルの制御** は、ビューを大量に作ったりする必要があります。

Row Access Policy を使えば、1 つのテーブルに対してロール別の行フィルタを宣言的に設定できます。

---

## 実装手順

全体の流れは以下のとおりです。

```
SECURITYADMIN: ロール作成・権限付与
      ↓
DATA_ENGINEER: テーブル作成・データ投入
      ↓
RAP_ADMIN: マッピングテーブル作成 → ポリシー作成 → テーブルに適用
```

### 0. ロールの作成と権限付与（SECURITYADMIN）

ポリシー管理者（RAP_ADMIN）とテーブル所有者（DATA_ENGINEER）を分離します。テーブル所有者がポリシーを勝手に外せない構成にするのがポイントです。

これは Snowflake 公式が推奨する **集中管理（Centralized）アプローチ** に基づいています。公式ドキュメントでは、集中管理・ハイブリッド・分散の3モデルが紹介されており、ガバナンスチームがポリシーの作成と適用を一元管理する集中管理モデルが職務分離の観点で最も推奨されています（[Choosing a centralized, hybrid, or decentralized management approach](https://docs.snowflake.com/en/user-guide/security-row-intro#choosing-a-centralized-hybrid-or-decentralized-management-approach)）。

```sql
USE ROLE SECURITYADMIN;

CREATE ROLE IF NOT EXISTS DATA_ENGINEER
    COMMENT = 'テーブル作成・データ投入用ロール';

CREATE ROLE IF NOT EXISTS RAP_ADMIN
    COMMENT = 'Row Access Policy の作成・管理用ロール';

-- 両ロールを SYSADMIN 配下に配置
GRANT ROLE DATA_ENGINEER TO ROLE SYSADMIN;
GRANT ROLE RAP_ADMIN TO ROLE SYSADMIN;

-- 権限付与
GRANT USAGE ON DATABASE sandbox_db TO ROLE DATA_ENGINEER;
GRANT USAGE ON SCHEMA sandbox_db.work TO ROLE DATA_ENGINEER;
GRANT CREATE TABLE ON SCHEMA sandbox_db.work TO ROLE DATA_ENGINEER;

GRANT USAGE ON DATABASE sandbox_db TO ROLE RAP_ADMIN;
GRANT USAGE ON SCHEMA sandbox_db.work TO ROLE RAP_ADMIN;
GRANT CREATE ROW ACCESS POLICY ON SCHEMA sandbox_db.work TO ROLE RAP_ADMIN;
GRANT CREATE TABLE ON SCHEMA sandbox_db.work TO ROLE RAP_ADMIN;  -- マッピングテーブル用

GRANT USAGE ON WAREHOUSE sandbox_wh TO ROLE DATA_ENGINEER;
GRANT USAGE ON WAREHOUSE sandbox_wh TO ROLE RAP_ADMIN;
```

### 1. サンプルテーブルの作成（DATA_ENGINEER）

部門（department）列がポリシーのフィルタ対象になります。

```sql
USE ROLE DATA_ENGINEER;
USE DATABASE sandbox_db;
USE SCHEMA work;

CREATE OR REPLACE TABLE work.employees (
    employee_id   INT,
    name          STRING,
    department    STRING,   -- ポリシーの判定対象列
    salary        NUMBER,
    hire_date     DATE
);

INSERT INTO work.employees VALUES
    (1, '田中太郎',   'SALES',       5000000, '2020-04-01'),
    (2, '佐藤花子',   'ENGINEERING', 7000000, '2019-07-15'),
    (3, '鈴木一郎',   'HR',          4500000, '2021-01-10'),
    (4, '高橋美咲',   'SALES',       5500000, '2022-03-20'),
    (5, '伊藤健太',   'ENGINEERING', 8000000, '2018-11-01'),
    (6, '渡辺優子',   'HR',          4800000, '2023-06-01');

-- RAP_ADMIN がマッピングテーブルから参照できるよう権限付与
GRANT SELECT ON TABLE work.employees TO ROLE RAP_ADMIN;
```

### 2. マッピングテーブルの作成（RAP_ADMIN）

「どのロールがどの部門の行を見られるか」をテーブルで管理します。ポリシー本体に条件をハードコードせず、テーブルで管理することで柔軟に変更できます。これは公式ドキュメントの [Example: Mapping table lookup](https://docs.snowflake.com/en/user-guide/security-row-using#example-mapping-table-lookup) で紹介されているパターンです。

```sql
USE ROLE RAP_ADMIN;
USE DATABASE sandbox_db;
USE SCHEMA work;

CREATE OR REPLACE TABLE work.rap_mapping (
    role_name    STRING,
    department   STRING
);

INSERT INTO work.rap_mapping VALUES
    ('SALES_ROLE',       'SALES'),        -- SALES 部門のみ
    ('ENGINEERING_ROLE', 'ENGINEERING'),  -- ENGINEERING 部門のみ
    ('HR_ROLE',          'SALES'),        -- HR は全部門を閲覧可
    ('HR_ROLE',          'ENGINEERING'),
    ('HR_ROLE',          'HR');
```

### 3. Row Access Policy の作成（RAP_ADMIN）

`IS_ROLE_IN_SESSION()` でロール階層を考慮しながら判定します。SYSADMIN 系ロールは全行見えるバイパス条件を入れておくのが一般的です。

```sql
CREATE OR REPLACE ROW ACCESS POLICY work.rap_department_access
AS (department_val STRING)
RETURNS BOOLEAN ->
    IS_ROLE_IN_SESSION('SYSADMIN')
    OR EXISTS (
        SELECT 1
        FROM work.rap_mapping m
        WHERE m.department = department_val
          AND IS_ROLE_IN_SESSION(m.role_name)
    )
;
```

**`CURRENT_ROLE()` を使ってはいけない理由**: `CURRENT_ROLE()` はセッションのアクティブロール（1 つ）しか返さないため、継承ロールが無視されます。`IS_ROLE_IN_SESSION()` はロール階層全体を評価するため、これを使うのが推奨です。

:::note info
**`IS_ROLE_IN_SESSION('SYSADMIN')` でバイパスされるのはどのロール？**

「SYSADMIN 系ロール」と書きましたが、具体的には **SYSADMIN と ACCOUNTADMIN の 2 つ**です。

`IS_ROLE_IN_SESSION('X')` は「セッションのアクティブロールの**先祖に X がいるか**」を評価します。Snowflake のデフォルト階層では ACCOUNTADMIN が SYSADMIN を継承している（親の関係になっている）ため、ACCOUNTADMIN でも TRUE になります。

一方、チュートリアルで作成した DATA_ENGINEER・RAP_ADMIN は `GRANT ROLE ... TO ROLE SYSADMIN` で **SYSADMIN の子** として配置しています。子から親方向は継承されないため、これらのロールでは FALSE になります。

以下のクエリで実際に確認できます（セカンダリーロールを切ってから試すこと）。

```sql
USE SECONDARY ROLES NONE;

USE ROLE SYSADMIN;
SELECT IS_ROLE_IN_SESSION('SYSADMIN');   -- TRUE（SYSADMINで実行してリウので当然）

USE ROLE ACCOUNTADMIN;
SELECT IS_ROLE_IN_SESSION('SYSADMIN');   -- TRUE（ACCOUNTADMIN は SYSADMIN を継承・親の関係）

USE ROLE SALES_ROLE;
SELECT IS_ROLE_IN_SESSION('SYSADMIN');   -- FALSE（SALES_ROLEは SYSADMIN を継承していない・子の関係）
```
:::

### 3-1. APPLY 権限の付与（SECURITYADMIN）

ポリシーが作成された後に、RAP_ADMIN にテーブルへの適用権限を付与します。

```sql
USE ROLE SECURITYADMIN;
GRANT APPLY ON ROW ACCESS POLICY sandbox_db.work.rap_department_access TO ROLE RAP_ADMIN;
```

### 4. テーブルへの適用（RAP_ADMIN）

```sql
USE ROLE RAP_ADMIN;
ALTER TABLE sandbox_db.work.employees
    ADD ROW ACCESS POLICY work.rap_department_access
    ON (department);
```

---

## 動作確認

ポリシーが正しく適用されているかを `information_schema.policy_references` で確認できます。

```sql
SELECT
    policy_name,
    policy_kind,
    ref_entity_name,
    ref_column_name
FROM TABLE(information_schema.policy_references(
    ref_entity_name   => 'sandbox_db.work.employees',
    ref_entity_domain => 'TABLE'
));
```

各ロールで SELECT を実行すると、見える行が異なることを確認できます。

```sql
-- SALES_ROLE → SALES 部門の 2 行のみ
-- ENGINEERING_ROLE → ENGINEERING 部門の 2 行のみ
-- HR_ROLE → 全 6 行
-- SYSADMIN → 全 6 行（バイパス）
```


---

## ハマったポイントと解決策

### セカンダリーロールでポリシーがバイパスされる

セカンダリーロールは、ログイン中のユーザーが使用できるロールを同時に使用できる機能で、デフォルトだとオンになっています（[セカンダリーロールの概要](https://docs.snowflake.com/en/user-guide/security-access-control-overview#authorization-through-primary-role-and-secondary-roles)）。`IS_ROLE_IN_SESSION('SYSADMIN')` でバイパス条件を設定しているとき、テスト用の制限ロールに切り替えても SYSADMIN がセカンダリーロールとして残っていると全行見えてしまいます。ロール切替の効果を正確に確認したい場合は `USE SECONDARY ROLES NONE;` で無効化してからテストします。

```sql
-- テスト前に必ず実行
USE SECONDARY ROLES NONE;
```

### テーブル所有者はポリシーを外せる

`ALTER TABLE ... DROP ROW ACCESS POLICY` はテーブルの OWNERSHIP を持つロールなら `APPLY` 権限なしでも実行できます。本番環境では、重要テーブルの OWNERSHIP を DATA_ENGINEER から剥奪し、SYSADMIN や専用の管理ロールだけが持つ設計にすることを検討します。

---

## まとめ

| 項目 | 内容 |
|---|---|
| 適用単位 | テーブル/ビュー 1 つにつき 1 ポリシー |
| ロール判定 | `IS_ROLE_IN_SESSION()` 推奨（階層考慮） |
| 管理方法 | マッピングテーブルで動的管理が柔軟 |
| ロール設計 | テーブル所有者とポリシー管理者を分離 |
| 注意点 | セカンダリーロールの影響に注意 |

Row Access Policy を使うことで、ビューを乱立させることなく行レベルのアクセス制御を一元管理できます。特にマルチテナントや部門別データ分離が必要な場面で威力を発揮します。

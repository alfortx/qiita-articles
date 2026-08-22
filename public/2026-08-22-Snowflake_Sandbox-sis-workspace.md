---
title: Streamlit in Snowflake(SiS) をCoCoで開発するために必要なこと
tags:
  - Snowflake
  - Streamlit
  - coco
  - CortexCode
  - SIS
private: false
updated_at: '2026-08-22T16:04:26+09:00'
id: 71ed5b6a1fe8d93eca6a
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## TL;DR

- Snowsight のワークスペース＋CoCo（Cortex Code in Snowsight）で Streamlit in Snowflake（SiS）を対話的に開発する手順を、実機で一通り検証しました
- SiSにはワークスペース方式とステージ方式がありますが、CoCoで対話開発をしたいならワークスペース方式一択です
- 実運用でありがちな、作成者（OWNER）と利用者（VIEWER）でロールを分ける場合の注意点もまとめました

## 環境

- Snowflake（有償アカウント、Standard Edition）

## 背景・課題

SiS には「ワークスペース方式」と「ステージ方式」の2つのデプロイ経路があります。片方がもう片方を包含する関係ではなく、完全に独立した経路で、アプリの起動ファイルと設定ファイルの配置先自体が異なります。

| | ワークスペース方式 | ステージ方式 |
|---|---|---|
| CoCo による編集 | ✅ | ❌（ワークスペース外は編集対象外） |
| ソースの保管場所 | ワークスペース内のファイル | 内部ステージ |
| デプロイ操作 | Workspace UI の Deploy ボタン | `CREATE STREAMLIT FROM '@stage'` |
| 対応ランタイム | container のみ | container / warehouse 両方 |


CoCo in Snowsight は、ワークスペース内のファイルしか読めず、ステージ上のファイルは編集対象外です。つまり、CoCo で Streamlit アプリを自然言語で開発したいなら、**ワークスペース方式一択になります**。

そしてワークスペース方式は**container ランタイム専用**となるため、軽量な warehouse ランタイムは選べません。開発中も含めて**コンピュートプールが必須**になります

## 権限設計（owner's rights とロール分離）

SiS は既定で **owner's rights** モデルで動作します。アプリ内のクエリは常にアプリのオーナーロールの権限で実行され、閲覧者自身の権限は使われません。ストアドプロシージャの `EXECUTE AS OWNER` と同じ考え方です。

この記事の検証では、実運用を想定して**作成・デプロイ用の OWNER ロール**と**閲覧専用の VIEWER ロール**を分けています。必要な権限をマトリクスにすると次のとおりです。

| 権限 | OWNER | VIEWER |
|---|:---:|:---:|
| USAGE ON DATABASE | ✅ | ✅ |
| USAGE ON SCHEMA（アプリ置き場） | ✅ | ✅ |
| USAGE ON SCHEMA（データ置き場） | ✅ | ❌ |
| CREATE STREAMLIT ON SCHEMA | ✅ | ❌ |
| SELECT ON TABLE（データ置き場） | ✅ | ❌ |
| USAGE ON WAREHOUSE | ✅ | ❌ |
| USAGE, OPERATE ON COMPUTE POOL | ✅ | ❌ |
| USAGE ON INTEGRATION（EAI） | ✅ | ❌ |
| **USAGE ON STREAMLIT（アプリ本体）** | オーナーなので不要 | ✅ |



## 検証手順

> 注意：今回の検証では課金が発生するウェアハウスやコンピュートプール、AI機能を利用します。

### 1. 専用リソースをSQLで新設

学習用に DB・スキーマ・WH・コンピュートプール・EAI・ロールを新設します。ポイントは、**アプリ置き場のスキーマ（`APP`）とデータ置き場のスキーマ（`DATA`）を分離**し、VIEWER には `DATA` への権限を一切与えないことです。

```sql
USE ROLE SYSADMIN;

CREATE DATABASE IF NOT EXISTS SISWS_DB;
CREATE SCHEMA IF NOT EXISTS SISWS_DB.APP;   -- Streamlit オブジェクト置き場
CREATE SCHEMA IF NOT EXISTS SISWS_DB.DATA;  -- 参照テーブル置き場

CREATE WAREHOUSE IF NOT EXISTS SISWS_WH
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 60
  INITIALLY_SUSPENDED = TRUE;

-- データは予め用意したコロナの感染者数データ
CREATE OR REPLACE TABLE SISWS_DB.DATA.JHU_TIMESERIES AS
  SELECT * FROM RAW_DB.COVID19.V_JHU_TIMESERIES;
CREATE OR REPLACE TABLE SISWS_DB.DATA.WORLD_TESTING AS
  SELECT * FROM RAW_DB.COVID19.V_COVID19_WORLD_TESTING;
```

コンピュートプールと、Pythonパッケージを PyPI から取得するための External Access Integration（EAI）も作成します。
コンピュートプールはアカウントレベルオブジェクトです。
Snowflake 管理のネットワークルール `SNOWFLAKE.EXTERNAL_ACCESS.PYPI_RULE` を使います。

```sql
USE ROLE ACCOUNTADMIN;

CREATE COMPUTE POOL IF NOT EXISTS SISWS_COMPUTE_POOL
  MIN_NODES = 1
  MAX_NODES = 1
  INSTANCE_FAMILY = CPU_X64_XS
  AUTO_SUSPEND_SECS = 60
  INITIALLY_SUSPENDED = TRUE;

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION SISWS_PYPI_EAI
  ALLOWED_NETWORK_RULES = (SNOWFLAKE.EXTERNAL_ACCESS.PYPI_RULE)
  ENABLED = TRUE;
```

ロール2つとGRANT、FUTURE GRANTをまとめて設定します。

```sql
USE ROLE SECURITYADMIN;

CREATE ROLE IF NOT EXISTS SISWS_OWNER_ROLE; -- アプリ作成者
CREATE ROLE IF NOT EXISTS SISWS_VIEWER_ROLE; -- アプリ利用者

-- OWNER: アプリ作成とDATAの利用権限を与える
GRANT USAGE ON DATABASE SISWS_DB TO ROLE SISWS_OWNER_ROLE;
GRANT USAGE ON SCHEMA SISWS_DB.APP TO ROLE SISWS_OWNER_ROLE;
GRANT USAGE ON SCHEMA SISWS_DB.DATA TO ROLE SISWS_OWNER_ROLE;
GRANT CREATE STREAMLIT ON SCHEMA SISWS_DB.APP TO ROLE SISWS_OWNER_ROLE;
GRANT SELECT ON ALL TABLES IN SCHEMA SISWS_DB.DATA TO ROLE SISWS_OWNER_ROLE;

-- OWNER: WH / コンピュートプール / EAI
GRANT USAGE ON WAREHOUSE SISWS_WH TO ROLE SISWS_OWNER_ROLE;
GRANT USAGE, OPERATE ON COMPUTE POOL SISWS_COMPUTE_POOL TO ROLE SISWS_OWNER_ROLE;
GRANT USAGE ON INTEGRATION SISWS_PYPI_EAI TO ROLE SISWS_OWNER_ROLE;

-- VIEWER: DB とアプリ置き場スキーマの USAGE のみ。DATA スキーマには一切権限を与えない
GRANT USAGE ON DATABASE SISWS_DB TO ROLE SISWS_VIEWER_ROLE;
GRANT USAGE ON SCHEMA SISWS_DB.APP TO ROLE SISWS_VIEWER_ROLE;

-- APP スキーマに今後作られる STREAMLIT へ USAGE を自動付与する（FUTURE GRANT）
GRANT USAGE ON FUTURE STREAMLITS IN SCHEMA SISWS_DB.APP TO ROLE SISWS_VIEWER_ROLE;

-- <ログイン中ユーザー名> は SELECT CURRENT_USER(); で確認する
GRANT ROLE SISWS_OWNER_ROLE  TO USER <ログイン中ユーザー名>;
GRANT ROLE SISWS_VIEWER_ROLE TO USER <ログイン中ユーザー名>;
```

### 2. ワークスペースでアプリを作成

Snowsight で `SISWS_OWNER_ROLE` に切り替え、ワークスペースを開いて **+ Add new » Streamlit app** を選択すると、アプリ名の入力とコンピュートを先ほと作成したものを選択します
![alt text](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/image-1.png)
「ネットワーク」で外部アクセス統合を選択します
![alt text](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/image-3.png)

自分のワークスペースに、`streamlit_app.py` / `pyproject.toml` / `snowflake.yml` / `.streamlit/config.toml` の4ファイルが自動生成されます。
![alt text](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/image-2.png)
このファイルはダッシュボードのソースコードであり、ワークスペースにあるため**CoCoから編集可能です**。




### 3. Run → Deploy

アプリコードが編集できたら、**Run(実行)**で自分だけが見える開発アプリとして起動できます。
![alt text](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/image-4.png)

**Deploy(展開)** を押すと、App title / Location / Execution（compute pool・warehouse）/ Network（EAI）/ Sharing（USAGE 付与ロール）の5項目を確認するダイアログが出ます。
開発状態との違いはスキーマを指定することです。ここからも分かる通り、デプロイするとスキーマオブジェクトとして配置され他の人からも見えるようになります
![alt text](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/image-5.png)
ネットワークタブで、開発状態と同じく外部アクセス統合を有効化
![alt text](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/image-6.png)
共有タブで、ダッシュボード共有先のロールを選択できるようです
![alt text](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/image-7.png)



### 4. owner's rights の検証

VIEWER ロールはテーブルに1行もアクセスできないのに、デプロイ済みアプリ経由ではデータが見られることを確認します。

ただし同一ユーザーに OWNER と VIEWER 両方のロールを付与しているため、セッションでセカンダリロールが有効なままだと `USE ROLE` を切り替えても OWNER の権限が同時に有効化されるため、「VIEWER はテーブルに直接アクセスできない」という検証が成立しなくなる可能性があります。検証の前に `USE SECONDARY ROLES NONE` で無効化しておきます。

```sql
-- VIEWER でテーブルへの直接アクセスは失敗する
USE ROLE SISWS_VIEWER_ROLE;
USE SECONDARY ROLES NONE;
SELECT * FROM SISWS_DB.DATA.JHU_TIMESERIES LIMIT 1;
-- → SQL compilation error: Schema 'SISWS_DB.DATA' does not exist or not authorized.
```

一方、Snowsight で VIEWER ロールに切り替えてアプリを開くと、全チャートが正常に表示され、OWNER権限でデータにアクセスできていることがわかります！
![alt text](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/image-8.png)

### 5. クリーンナップ

学習が終わったら、コンピュートプール停止 → DB削除 → ロール削除の順で全リソースを消します。

```sql
USE ROLE ACCOUNTADMIN;
ALTER COMPUTE POOL SISWS_COMPUTE_POOL STOP ALL;
DROP COMPUTE POOL IF EXISTS SISWS_COMPUTE_POOL;
DROP EXTERNAL ACCESS INTEGRATION IF EXISTS SISWS_PYPI_EAI;

USE ROLE SYSADMIN;
DROP DATABASE IF EXISTS SISWS_DB;
DROP WAREHOUSE IF EXISTS SISWS_WH;

USE ROLE SECURITYADMIN;
DROP ROLE IF EXISTS SISWS_VIEWER_ROLE;
DROP ROLE IF EXISTS SISWS_OWNER_ROLE;
```

**Snowsight 上のワークスペース自体は SQL では削除できません。** Workspaces 画面から手動で消す必要があり、これは見落としやすいので注意してください。

## ハマったポイントと解決策

### 1. `pyproject.toml` に streamlit 自体を書かないと起動エラー

アプリコードの中にパッケージの依存関係を記述する `pyproject.toml` があります。container ランタイムには Python / Streamlit / Snowpark がプリインストールされているはずなので、「pandas と altair だけ追加すればいい」と思って以下のように書いたところ、アプリを開いた瞬間に落ちました。

```toml
# これだとエラーになる
dependencies = ["pandas>=2.0.0", "altair>=5.0.0"]
```

```
Failed to get the version of the Streamlit library. Please check if the Streamlit
library is installed and fulfills the following version constraints: ">=1.48.0".
```

`pyproject.toml`（依存ファイル）を1つでも作成した時点で `uv sync` が環境をその定義どおりに再構築し、明記していないプリインストール済みパッケージ（streamlit 本体を含む）はインストールされなくなります（[公式ドキュメント: Dependency files](https://docs.snowflake.com/en/developer-guide/streamlit/app-development/dependency-management#dependency-files)）。streamlit 自体も `streamlit[snowflake]>=1.48.0` の形で明記する必要があります。

### 2. VIEWER への STREAMLIT 権限が漏れやすい

viewerにはstreamlitアプリに対する`USAGE`権限が必要となるため作成者が都度権限を与える必要が発生してしまいます。またアプリを再デプロイすると STREAMLIT オブジェクトはスキーマに作り直されるため、この GRANT は毎回失われます。このためスキーマに対して`GRANT USAGE ON FUTURE STREAMLITS`を許可しておく運用が良さそうです。

```sql
GRANT USAGE ON FUTURE STREAMLITS IN SCHEMA SISWS_DB.APP TO ROLE SISWS_VIEWER_ROLE;
```

### 3. streamlitを止めたはずなのに、COMPUTE POOLが停止しない

COMPUTE POOLは、サービスを停止すれば止まるはず・・・ですが、sisの場合、**アプリを止めてもブラウザワークスペースを開いていると、自動で再起動するようです。**`Takahir_O`さんが痛い経験をまとめて頂いてました。（感謝）
https://zenn.dev/fusic/articles/0010-snowflake-cost-streamlit


## まとめ

ファイル配置方式が複数あったり、権限周りが特殊だったりと、自分の中でモヤがかかっていたSIS周りの設定を理解することができました。

CoCoの登場により、自然言語だけで楽にダッシュボード作ろうぜ、という声が社内で多く上がっていますが、綺麗に使うには色々と気にしなければならいことがわかりました。

次はAPP RUNTIMEでアプリ作ります！

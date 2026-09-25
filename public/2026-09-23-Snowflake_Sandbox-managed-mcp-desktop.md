---
title: '【Snowflake】MCP Server で Claude Desktop からアクセスする経路を作るまでの試行錯誤'
tags:
  - Snowflake
  - MCP
  - Cortex
  - Claude
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## TL;DR

- Snowflake 側で必要なのは **OAuth Security Integration**・**MCP Server**・**GRANT 一式**の3つです。Claude Desktop 側は公式「Snowflake」コネクタに URL と Client ID / Secret を入れるだけです
- ハマりどころは2つでした。① MCP SERVER の USAGE だけでなく**スキーマの USAGE** も必要 ② Agent spec で固定していたモデル `claude-4-sonnet` が**廃止されていた**（`auto` で解決）
- つながらないときは `LOGIN_HISTORY` → 接続元 IP が Anthropic のレンジ内か → `QUERY_HISTORY` の順に切り分けると、どの段階で止まっているかが分かります

## 環境

| 項目 | 内容 |
|---|---|
| Snowflake | Standard エディション（AWS） |
| クライアント | Claude Desktop（2026年9月時点）、Claude Code 2.1.280 |
| 公開する Agent | Cortex Agent 2つ（Cortex Analyst / Cortex Search を使う既存の Agent） |

## MCP を使う意義

Cortex Agent は Snowsight や REST API から呼べますが、分析する人が普段使っている AI クライアントから使えると便利です。

Snowflake Managed MCP Server は、Snowflake 自身が MCP サーバーをホストする機能です。サーバーを自前で立てる必要がなく、アクセスは OAuth ＋ロールで制御されます。**データアクセス統制を Snowflake で一元管理しつつ、入口だけを AI クライアントに開ける**のが最大の利点です。

## 構成

```text
Claude Desktop（公式 Snowflake コネクタ）
   │  OAuth（Client ID / Secret、redirect: https://claude.ai/api/mcp/auth_callback）
   ▼
Security Integration: MCP_OAUTH_CLAUDE（ALLOWED_ROLES_LIST = ANALYST_ROLE）
   ▼
MCP Server: ANALYTICS_DB.MCP.ANALYSIS_MCP_SERVER
   ├─ tool: sales_agent    → AGENT_DB.AGENTS.SALES_AGENT
   └─ tool: support_agent  → AGENT_DB.AGENTS.SUPPORT_AGENT
```

公開するツールは `CORTEX_AGENT_RUN` だけにして、SQL を直接実行する `SYSTEM_EXECUTE_SQL` は入れていません。[Snowflake-managed MCP server](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp) に記載があるように、

> Exposing `SYSTEM_EXECUTE_SQL` on the same server allows the MCP client to bypass the agent's semantic views, verified queries, and orchestration; if direct SQL is required, expose it through a separate MCP server with a dedicated least-privileged role.

同じサーバーに SQL ツールを置くと、クライアントは **Agent のセマンティックビューや検証済みクエリを通らずにデータを読めてしまい**ます。分析の窓口としてアクセス導線をCortexツールに絞りたいなら、Agent だけに絞るのが安全です。

## 実装手順

### 1. OAuth Security Integration を作る

前提：Integration 名は `MCP_OAUTH_CLAUDE`、MCP 経由で使わせるロールは `ANALYST_ROLE` とします。

```sql
CREATE SECURITY INTEGRATION "MCP_OAUTH_CLAUDE"
  TYPE                         = OAUTH
  OAUTH_CLIENT                 = CUSTOM
  OAUTH_CLIENT_TYPE            = 'CONFIDENTIAL'
  OAUTH_REDIRECT_URI           = 'https://claude.ai/api/mcp/auth_callback'
  ENABLED                      = true
  OAUTH_USE_SECONDARY_ROLES    = NONE
  ALLOWED_ROLES_LIST           = ("ANALYST_ROLE")
  OAUTH_ISSUE_REFRESH_TOKENS   = true
  OAUTH_REFRESH_TOKEN_VALIDITY = 7776000;  -- 90日
```

- `OAUTH_REDIRECT_URI` は Claude Desktop / claude.ai 共通の `https://claude.ai/api/mcp/auth_callback` です（[Anthropic 公式](https://claude.com/docs/connectors/building/authentication)）
- `ALLOWED_ROLES_LIST` で、MCP 経由で使えるロールを分析用ロールに限定しています。セッションのロールは、接続ユーザーの `DEFAULT_ROLE` になります

Client ID / Secret は次のクエリで取得します（ACCOUNTADMIN で実行）。

```sql
DESC INTEGRATION MCP_OAUTH_CLAUDE;                       -- OAUTH_CLIENT_ID
SELECT SYSTEM$SHOW_OAUTH_CLIENT_SECRETS('MCP_OAUTH_CLAUDE'); -- OAUTH_CLIENT_SECRET
```

### 2. MCP Server を作る

前提：MCP Server は `ANALYTICS_DB.MCP` スキーマに作り、作成済みの Cortex Agent `AGENT_DB.AGENTS.SALES_AGENT` と `AGENT_DB.AGENTS.SUPPORT_AGENT` をツールとして公開します。

```sql
CREATE MCP SERVER "ANALYTICS_DB"."MCP"."ANALYSIS_MCP_SERVER"
  COMMENT = '分析ユーザーがCortex Agent経由でデータ分析を行うための窓口'
  FROM SPECIFICATION $$
tools:
  - name: "support_agent"
    type: "CORTEX_AGENT_RUN"
    identifier: "AGENT_DB.AGENTS.SUPPORT_AGENT"
    title: "Support Agent"
    description: "問い合わせ履歴に関する分析を行うエージェント"
  - name: "sales_agent"
    type: "CORTEX_AGENT_RUN"
    identifier: "AGENT_DB.AGENTS.SALES_AGENT"
    title: "Sales Agent"
    description: "売上データに関する分析を行うエージェント"
$$;
```

`description` は、Claude がどのツールを呼ぶか判断する材料になります。Agent の守備範囲が分かるように書いておくと、呼び分けが安定します。

### 3. GRANT する

前提：接続ユーザー `analyst_user` の `DEFAULT_ROLE` は `ANALYST_ROLE` です。スキーマの USAGE は、データベースロール `ANALYTICS_DB.MCP_USE` 経由で付与します（理由は罠1で説明します）。

まず、MCP Server に到達するための権限です。

```sql
-- Cortex Agent を実行するためのデータベースロール
GRANT DATABASE ROLE "SNOWFLAKE"."CORTEX_AGENT_USER" TO ROLE "ANALYST_ROLE";

-- MCP Server の USAGE
GRANT USAGE ON MCP SERVER "ANALYTICS_DB"."MCP"."ANALYSIS_MCP_SERVER" TO ROLE "ANALYST_ROLE";

-- MCP Server が置かれたスキーマの USAGE
CREATE DATABASE ROLE "ANALYTICS_DB"."MCP_USE";
GRANT USAGE ON SCHEMA "ANALYTICS_DB"."MCP" TO DATABASE ROLE "ANALYTICS_DB"."MCP_USE";
GRANT DATABASE ROLE "ANALYTICS_DB"."MCP_USE" TO ROLE "ANALYST_ROLE";
```

次に、Agent を実行するための権限です。MCP Server の USAGE だけではツール（Agent）は使えず、**Agent 本体と、Agent が内部で使うリソースの権限が別途必要**です。公式ドキュメントにも "Access to the MCP Server does not give access to the tools." とあります。

前提：`SALES_AGENT` は Semantic View `AGENT_DB.SEMANTIC_MODELS.SALES_SV` と、Cortex Search Service `AGENT_DB.SEARCH_SERVICES.SALES_SEARCH` をツールに持ち、ウェアハウス `ANALYST_WH` で動くとします。

```sql
-- Agent 本体
GRANT USAGE ON DATABASE AGENT_DB                         TO ROLE ANALYST_ROLE;
GRANT USAGE ON SCHEMA   AGENT_DB.AGENTS                  TO ROLE ANALYST_ROLE;
GRANT USAGE ON AGENT    AGENT_DB.AGENTS.SALES_AGENT      TO ROLE ANALYST_ROLE;

-- Cortex Analyst が使う Semantic View
GRANT USAGE  ON SCHEMA        AGENT_DB.SEMANTIC_MODELS          TO ROLE ANALYST_ROLE;
GRANT SELECT ON SEMANTIC VIEW AGENT_DB.SEMANTIC_MODELS.SALES_SV TO ROLE ANALYST_ROLE;

-- Cortex Search Service
GRANT USAGE ON SCHEMA                AGENT_DB.SEARCH_SERVICES              TO ROLE ANALYST_ROLE;
GRANT USAGE ON CORTEX SEARCH SERVICE AGENT_DB.SEARCH_SERVICES.SALES_SEARCH TO ROLE ANALYST_ROLE;

-- Agent を動かすウェアハウス
GRANT USAGE ON WAREHOUSE ANALYST_WH TO ROLE ANALYST_ROLE;
```

`SUPPORT_AGENT` も同じ要領で付与します。公式ドキュメントでは、接続ユーザーに `DEFAULT_WAREHOUSE` を設定しておくことも求められています（未設定だとセッションの初期化に失敗します）。

### 4. Claude Desktop でコネクタを登録する

1. 設定 → コネクタ → **探索** を開き、検索欄に `snow` と入力します
2. **提供元が Snowflake の公式「Snowflake」コネクタ**を選んで追加します
3. MCP Server の URL と、手順1で取得した Client ID / Secret を入力します
4. ブラウザで Snowflake のログイン画面が開くので、接続ユーザーでログインします

![Claude Desktop のコネクタ探索画面で snow を検索した結果](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/mcp-desktop-connector-search-annotated.png)

URL は次の形式です。`<org>-<account>` は**ハイフン区切り・小文字**で書きます。

```text
https://<org>-<account>.snowflakecomputing.com/api/v2/databases/ANALYTICS_DB/schemas/MCP/mcp-servers/ANALYSIS_MCP_SERVER
```

## 接続できないときの調査方法

「認証は通るのにつながらない」ときは、Snowflake 側の履歴から、どの段階で止まっているかを切り分けます。

| 段階 | 見るもの | 分かること |
|---|---|---|
| ① 通信・認証 | `LOGIN_HISTORY_BY_USER` | Snowflake まで届いたか、認証が成功したか |
| ② 接続元 | CLIENT_IP と Anthropic の IP レンジ | リクエストが Claude（Anthropic のインフラ）から来ているか |
| ③ クエリ実行 | `QUERY_HISTORY` | Agent がそのユーザー・ロールで SQL を実行したか |

どれも `INFORMATION_SCHEMA` のテーブル関数を使います。`ACCOUNT_USAGE` のビューは反映に遅延があるので、切り分けの最中には向きません。

### ① LOGIN_HISTORY で認証の成否を見る

前提：接続ユーザーは `analyst_user`（小文字で作成）とします。テーブル関数を呼ぶデータベースはどれでも構いません（ここでは `ANALYTICS_DB`）。

```sql
SELECT EVENT_TIMESTAMP, CLIENT_IP, REPORTED_CLIENT_TYPE,
       FIRST_AUTHENTICATION_FACTOR, IS_SUCCESS, ERROR_MESSAGE
FROM TABLE(ANALYTICS_DB.INFORMATION_SCHEMA.LOGIN_HISTORY_BY_USER(
       USER_NAME        => 'analyst_user',
       TIME_RANGE_START => DATEADD('minute', -15, CURRENT_TIMESTAMP())))
ORDER BY EVENT_TIMESTAMP DESC;
```

Claude Desktop から接続すると、次のように2種類のログインが記録されます。

```text
| EVENT_TIMESTAMP               | CLIENT_IP      | REPORTED_CLIENT_TYPE | FIRST_AUTHENTICATION_FACTOR | IS_SUCCESS |
|-------------------------------+----------------+----------------------+-----------------------------+------------|
| 2026-09-23 15:20:57.715 -0700 | 160.79.106.169 | SQL_API              | OAUTH_ACCESS_TOKEN          | YES        |
| 2026-09-23 15:20:52.611 -0700 | 160.79.106.168 | SQL_API              | OAUTH_ACCESS_TOKEN          | YES        |
| 2026-09-23 15:20:48.240 -0700 | xxx.xxx.xx.xx  | SNOWFLAKE_UI         | PASSWORD                    | YES        |
```

- `SNOWFLAKE_UI` / `PASSWORD`：自分のブラウザで Snowflake にログインした記録（OAuth の認可画面）
- `SQL_API` / `OAUTH_ACCESS_TOKEN`：発行されたトークンで、Claude 側が MCP Server にアクセスした記録

`OAUTH_ACCESS_TOKEN` の行が `IS_SUCCESS = YES` で並んでいれば、通信も認証も問題ありません。

ユーザー名を小文字で作っている場合、`USER_NAME => 'ANALYST_USER'` と大文字で書くと `User 'ANALYST_USER' does not exist or not authorized.` になります。

### ② 接続元 IP が Anthropic のレンジ内か確かめる

Claude Desktop・claude.ai のコネクタは、手元の PC からではなく **Anthropic のクラウドから** MCP Server に接続します。送信元の IPv4 レンジは `160.79.104.0/21` です。

- [Claude Connectors: Authentication](https://claude.com/docs/connectors/building/authentication) の「Network reference」
- [Anthropic IP addresses](https://platform.claude.com/docs/en/api/ip-addresses) の「Outbound」

同じ IP addresses のページには `160.79.104.0/23` も載っていますが、こちらは **Inbound**（Anthropic の API が受ける側）です。ネットワークポリシーで許可するのは `/21` のほうなので、取り違えないよう注意してください。

レンジ内かどうかは `PARSE_IP` で判定できます（仕組みは後述のコラムで説明します）。

```sql
SELECT CLIENT_IP,
       PARSE_IP(CLIENT_IP, 'INET'):ipv4
         BETWEEN PARSE_IP('160.79.104.0/21', 'INET'):ipv4_range_start
             AND PARSE_IP('160.79.104.0/21', 'INET'):ipv4_range_end AS IN_ANTHROPIC_RANGE,
       COUNT(*) AS CNT
FROM TABLE(ANALYTICS_DB.INFORMATION_SCHEMA.LOGIN_HISTORY_BY_USER(
       USER_NAME        => 'analyst_user',
       TIME_RANGE_START => DATEADD('hour', -3, CURRENT_TIMESTAMP())))
GROUP BY 1, 2
ORDER BY 3 DESC;
```

```text
| CLIENT_IP      | IN_ANTHROPIC_RANGE | CNT |
|----------------+--------------------+-----|
| 160.79.106.161 | True               | 4   |
| xxx.xxx.xx.xx  | False              | 2   |
| 160.79.106.169 | True               | 2   |
```

OAuth トークンでのアクセスが `True` になっていれば、Claude からのリクエストは Snowflake まで届いています。ネットワークポリシーやファイアウォールを疑う必要はありません。逆に `LOGIN_HISTORY` に何も残っていなければ、手前で遮断されている可能性が高いです。

### コラム：PARSE_IP で IP アドレスを数値として比較する

`PARSE_IP` は、IP アドレスや CIDR の文字列を分解して OBJECT で返す関数です。単一の IP と CIDR を渡すと、それぞれ次のように返ります（主要なフィールドのみ抜粋）。

```sql
SELECT PARSE_IP('160.79.106.169', 'INET') AS host,
       PARSE_IP('160.79.104.0/21', 'INET') AS cidr;
```

```text
HOST: { "family": 4, "host": "160.79.106.169", "ipv4": 2689559209, "netmask_prefix_length": null }
CIDR: { "family": 4, "host": "160.79.104.0",   "ipv4": 2689558528,
        "ipv4_range_start": 2689558528, "ipv4_range_end": 2689560575, "netmask_prefix_length": 21 }
```

IPv4 は `ipv4` に**整数**で入り、CIDR を渡すと範囲の始点と終点も `ipv4_range_start` / `ipv4_range_end` に整数で入ります。そのため「IP の整数値が範囲の始点と終点の間にあるか」を `BETWEEN` で書くだけで、レンジ内かどうかを判定できます。文字列の前方一致（`LIKE '160.79.10%'` など）では `/21` のような境界を正しく扱えないので、こちらを使うのが確実です。

IPv6 を渡すと、`ipv4` の代わりに `hex_ipv6`・`hex_ipv6_range_start`・`hex_ipv6_range_end` が16進文字列で返ります。IPv6 のレンジを判定したいときは、こちらを文字列として比較します。

### ③ QUERY_HISTORY でクエリ実行を見る

```sql
SELECT START_TIME, ROLE_NAME, WAREHOUSE_NAME, EXECUTION_STATUS, ERROR_MESSAGE
FROM TABLE(ANALYTICS_DB.INFORMATION_SCHEMA.QUERY_HISTORY(
       END_TIME_RANGE_START => DATEADD('minute', -30, CURRENT_TIMESTAMP()),
       RESULT_LIMIT         => 1000))
WHERE USER_NAME = 'analyst_user'
ORDER BY START_TIME;
```

ここで2つ注意があります。

**1つ目：接続しただけではクエリは記録されません。** コネクタを接続した直後（ツール一覧を取得しただけ）の時点では、`QUERY_HISTORY` は0件です。Claude に質問して Agent が呼ばれて、初めて記録されます。

**2つ目：`QUERY_HISTORY_BY_USER` は小文字のユーザー名だとエラーにならずに0件を返します。**
①の`LOGIN_HISTORY_BY_USER` は `'analyst_user'` で取れるのに、`QUERY_HISTORY_BY_USER` は `'"analyst_user"'` とダブルクォートで囲まないとヒットしません。同じ条件で比べた結果です。

```text
| 取得方法                                           | 件数 |
|----------------------------------------------------+------|
| QUERY_HISTORY_BY_USER(USER_NAME=>'analyst_user')   | 0    |
| QUERY_HISTORY_BY_USER(USER_NAME=>'"analyst_user"') | 6    |
| QUERY_HISTORY() + WHERE USER_NAME='analyst_user'   | 6    |
```

エラーにならないので、「クエリが来ていない」と誤読しやすいです。上の SQL のように `QUERY_HISTORY()` を `WHERE` で絞るほうが確実です。

## ハマったポイント

### 罠1：`MCP server ... does not exist or not authorized`

`LOGIN_HISTORY` では認証が成功しているのに、Claude Desktop 側では次のエラーになりました。

```text
MCP server ANALYTICS_DB.MCP.ANALYSIS_MCP_SERVER does not exist or not authorized.
```

MCP Server 自体の権限は `GRANT USAGE ON MCP SERVER` で付与済みでした。原因は、**MCP Server が置かれたスキーマ `ANALYTICS_DB.MCP` の USAGE が接続ロールに無かった**ことです。

MCP Server はスキーマレベルのオブジェクトなので、通常のテーブルと同じく、上位のデータベースとスキーマの USAGE も必要です。Snowflake は権限が足りない場合も「存在しない」と同じメッセージを返すので、MCP SERVER の USAGE を付けていると、スキーマ側の抜けに気づきにくいです。

今回はデータベースロール `ANALYTICS_DB.MCP_USE` を作って、そこにスキーマの USAGE を付け、接続ロールに継承させました。データベースロールは、自分が属するデータベースの USAGE を暗黙に持ちます。そのため、スキーマの USAGE だけ付ければデータベース側の USAGE は不要です。

```sql
GRANT USAGE ON SCHEMA "ANALYTICS_DB"."MCP" TO DATABASE ROLE "ANALYTICS_DB"."MCP_USE";
GRANT DATABASE ROLE "ANALYTICS_DB"."MCP_USE" TO ROLE "ANALYST_ROLE";
```

これで MCP Server に到達し、Agent の呼び出しまで進みました。

### 罠2：Agent エラー 399504（固定モデルの廃止）

MCP 経由で Agent は呼べたものの、今度は Agent 自体がエラーを返しました。

```text
MCP error calling tool sales_agent: Agent error (code 399504): claude-4-sonnet is not an allowed model for Agent requests. Please switch your agent configuration to use 'auto' for automatic model selection, or choose a different model.
```

Agent spec は MCP を試すよりだいぶ前に書いたもので、オーケストレーションのモデルを `claude-4-sonnet` に固定していました。その間にこのモデルが Agent で使えなくなっていた、というのが原因です。Snowsight から Agent を触っていなかったので、MCP 経由で初めて気づきました。

```yaml
models:
  orchestration: auto   # 変更前: claude-4-sonnet
```

エラーメッセージの推奨どおり `auto` に変更して、`CREATE OR REPLACE AGENT` で作り直したところ解消しました。モデルを固定すると、廃止されるたびに同じ壊れ方をします。特定のモデルにこだわる理由がなければ、`auto` にしておくのが無難です。

## 動作確認

Claude Desktop で質問すると、Claude が `sales_agent` ツールを選んで呼び出します。③の `QUERY_HISTORY` で確認すると、Agent が生成した SQL が接続ユーザー・ロールで実行されていました。

```text
| START_TIME                    | USER_NAME    | ROLE_NAME    | WAREHOUSE_NAME | EXECUTION_STATUS  |
|-------------------------------+--------------+--------------+----------------+-------------------|
| 2026-09-23 15:23:50.702 -0700 | analyst_user | ANALYST_ROLE | ANALYST_WH     | SUCCESS           |
| 2026-09-23 15:23:55.126 -0700 | analyst_user | ANALYST_ROLE | ANALYST_WH     | SUCCESS           |
```

`ALLOWED_ROLES_LIST` で許可した `ANALYST_ROLE` のまま Agent が動いていることが確認できました。

## おまけ：Claude Code からも同じコネクタが使える

claude.ai（Claude Desktop）で追加したコネクタは、**同じ claude.ai アカウントでログインしている Claude Code でも自動で使えます**。Claude Code の公式ドキュメントの「[claude.ai から MCP サーバーを使用する](https://code.claude.com/docs/ja/mcp#use-mcp-servers-from-claude-ai)」に記載があります。

実際に Claude Code で `claude mcp list` を実行すると、`claude.ai` の接頭辞付きで表示されます。

```bash
$ claude mcp list
claude.ai Snowflake: https://<org>-<account>.snowflakecomputing.com/api/v2/databases/ANALYTICS_DB/schemas/MCP/mcp-servers/ANALYSIS_MCP_SERVER - ✔ Connected
```

Claude Code 用に Integration を別に作って `claude mcp add` する必要はありません。Claude Code の OAuth コールバックは localhost なので、自前で登録すると redirect URI の異なる Integration がもう1つ必要になります。コネクタを共有すれば、Integration は1つで済みます。

ほかの Snowflake 用ツール（Cortex Code CLI など）とツールが競合する場合は、Claude Code の `/mcp` からこのコネクタだけを無効化できます。

## まとめ

- MCP Server の GRANT は `USAGE ON MCP SERVER` だけでは足りません。**スキーマ（とデータベース）の USAGE** も必要で、抜けていてもエラー文は「存在しない」になります
- **`models.orchestration` の固定モデルは使えなくなる**ことがあります。迷ったら `auto` が無難です
- つながらないときは `LOGIN_HISTORY` → IP レンジ→ `QUERY_HISTORY` の順に切り分けます。

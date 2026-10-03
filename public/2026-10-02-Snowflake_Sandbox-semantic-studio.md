---
title: 【Snowflake】Semantic Studio でのセマンティックビューの自動生成と内部挙動について調べた
tags:
  - Snowflake
  - Snowsight
  - SemanticView
  - coco
private: false
updated_at: '2026-10-03T09:01:06+09:00'
id: c35be2fb3b740824de8e
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## TL;DR

- Workspaces で `.sv.yaml` を作って「CoCoで作成する」を選ぶと、CoCo がテーブルを調べてセマンティックビューの YAML を生成します
- 「Publish」を押すと、スキーマにセマンティックビューが作られます。公開先は「ターゲット」として名前を付けて管理でき、dev / staging のように複数持てます
- 定義が有効かどうかは画面下部で自動検証され、エラーと警告の件数が表示されます

## 環境

| 項目 | 内容 |
|---|---|
| 検証日 | 2026-10-02（GA の 2 日後） |
| Snowflake | 有料アカウント・Standardエディション |
| 画面 | Snowsight（日本語 UI） |

## Semantic Studio とは

Semantic Studio は、Workspaces の中でセマンティックビューを作るための環境です。[GA のリリースノート](https://docs.snowflake.com/en/release-notes/2026/other/2026-09-30-semantic-studio-ga)では次のように説明されています。

> Semantic Studio is the authoring environment for semantic views in Workspaces, combining conversational authoring with CoCo, direct YAML editing, and deploying YAML files to Snowflake objects.

CoCoとの対話による作成、YAML の直接編集、Snowflake オブジェクトへのデプロイを 1 か所でこなせます。2026-09-30 に GA になりました。

[公式ドキュメント](https://docs.snowflake.com/en/user-guide/views-semantic/semantic-studio)によると、作成に使うロールには次の権限が必要です。

- 作成先スキーマへの `CREATE SEMANTIC VIEW`
- 作成先データベースとスキーマへの `USAGE`
- セマンティックビューで使うテーブル・ビューへの `SELECT`


なお、GA の 2 日後に試した時点でも、私の環境ではワークスペースの「新規追加」メニューに「プレビュー」の表示が残っていました。

![新規追加メニュー。セマンティックビューに「プレビュー」の表示が付いている](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-01-annotated.png)

## 手順1: .sv.yaml を作る

ワークスペースの「新規追加」から「セマンティックビュー」を選び、ファイル名を付けます。今回は `my_semantic` としたので `my_semantic.sv.yaml` が作られ、作成方法を選ぶ画面が開きました。

![作成方法を選ぶ画面。CoCoで作成する・ガイド付きウィザード・空白で開始の3つが並ぶ](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-02-annotated-v2.png)

選択肢は次の 3 つです。

| 選択肢 | 画面の説明 |
|---|---|
| CoCoで作成する（推奨） | CoCo がスキーマを検出し、セマンティックビューを生成する |
| ガイド付きウィザード | テーブル・列・コンテキストを段階的に設定する。YAML は不要 |
| 空白で開始 | 空のセマンティックビューを開き、YAML を直接書くか貼り付ける |

今回は「CoCoで作成する」を選びました。

## 手順2: CoCo に生成させる

### 質問に答える

画面右側に CoCo のパネルが開き、`/agent-studio` スキルが `Help me create a semantic view` というプロンプト付きで自動的に呼ばれます。

![CoCoのパネルが開き、/agent-studio スキルが呼ばれて最初の質問が表示された](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-03-annotated.png)

CoCo からは次の 3 つを質問されました。

1. Source tables: セマンティックビューに含めるテーブル
2. Target: 作成先のデータベース・スキーマ
3. Name: セマンティックビューの名前（ファイル名から `MY_SEMANTIC` が提案される）

セッションにデフォルトのデータベース・スキーマが設定されていなかったので、入力するよう求められました。事前に用意しておいたものを指定します。作成するロールがテーブルを読めることが前提です。

![完全修飾名でテーブルと作成先スキーマを入力した](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-04-annotated.png)

入力した内容は次のとおりです（データはコロナウイルス感染者の公開データをロードしたもの）。

```text
1. RAW_DB.COVID19.V_JHU_TIMESERIES
RAW_DB.COVID19.V_COVID19_WORLD_TESTING
2. CORTEX_DB.SEMANTIC_MODELS
```

### YAML を生成する

回答すると、CoCo は生成リクエスト（json_proto）を `/tmp/my_semantic_proto.json` に書き出します。作成先・テーブル・列名・説明文をまとめた JSON です。後述の QUERY_HISTORY に全文が残っていたので、列名を一部省略して載せます。

```json
{
  "json_proto": {
    "name": "MY_SEMANTIC",
    "database": "CORTEX_DB",
    "schema": "SEMANTIC_MODELS",
    "tables": [
      {
        "database": "RAW_DB", "schema": "COVID19", "table": "V_JHU_TIMESERIES",
        "columnNames": ["UID", "FIPS", "ISO2", "ISO3", "...", "CONFIRMED", "DEATHS", "RECOVERED"]
      },
      {
        "database": "RAW_DB", "schema": "COVID19", "table": "V_COVID19_WORLD_TESTING",
        "columnNames": ["ISO_CODE", "CONTINENT", "LOCATION", "DATE", "..."]
      }
    ],
    "semanticDescription": "COVID-19 analytics model combining JHU time series data (confirmed cases, deaths, recoveries by region) with world testing, vaccination, and demographic data.",
    "metadata": {"warehouse": "SANDBOX_WH"}
  }
}
```

続いて CoCo は、テーブルのメタデータと利用状況を集めます。過去 30 日の利用状況から、どの列がフィルターやメジャーとして使われているかを推測していました。`V_JHU_TIMESERIES` には「2 queries, 2 users (30d)」と出ているので、自分以外のユーザーの利用状況も参照しているようです。

![テーブルごとの利用状況。フィルターやメジャーに使われている列が推測されている](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-06-annotated.png)

ここから生成コマンドを実行します。CoCo が実行するコマンドは、毎回「Run this command?」と聞かれ、Skip か Allow を選びます。

```bash
cortex agent-studio sv-generate --file-path /tmp/my_semantic_proto.json --out-path /tmp/my_semantic_response.json 2>&1
```

生成結果は JSON で返り、YAML は `json_proto.semanticYaml` に文字列として入っています。これを取り出すスクリプトも CoCo が書いて実行します。

```bash
python3 -c "
import json
r = json.load(open('/tmp/my_semantic_response.json'))
yaml_content = r['json_proto']['semanticYaml']
with open('/tmp/my_semantic.sv.yaml', 'w') as f:
    f.write(yaml_content)
print(f'YAML extracted: {len(yaml_content)} chars')
# Show summary
lines = yaml_content.split('\n')
print(f'Lines: {len(lines)}')
# Count tables and VQRs
import re
tables = len(re.findall(r'^  - name:', yaml_content, re.MULTILINE))
vqrs = len(re.findall(r'verified_queries:', yaml_content))
print(f'Tables found in YAML: {tables}')
" 2>&1
```

最後に、取り出した YAML を `sv-write` でワークスペースのファイルに書き込みます。

```bash
cortex agent-studio sv-write --yaml-content "$(cat /tmp/my_semantic.sv.yaml)" --file-path my_semantic.sv.yaml 2>&1
```

:::note info
`cortex agent-studio`はCortexCliが実行するコマンドのようで、セマンティックビューとエージェントの操作を行います。[Cortex Code の changelog](https://docs.snowflake.com/en/user-guide/cortex-code/changelog)  `sv-generate`・`sv-write`・`sv-read` は、執筆時点で公式ドキュメントに記載がありません。今後追加されると思われます。
:::

## 手順3: 生成結果を確認して確定

書き込みが終わると、ワークスペースに `cortex_project` フォルダが作られ、セマンティック定義が入った `my_semantic.sv.yaml` と、`cortex-project.yaml` が置かれます。CoCo による変更は差分として表示され、画面下部のバーで「元に戻す」か「ファイル内のすべてを保持」を選べます。

![cortex_projectフォルダが作られ、セマンティック定義が書き込まれた](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-07-annotated.png)

`.sv.yaml` は「ビジュアル」と「YAML」を切り替えて表示でき、ビジュアル側からも編集できます（以前からあったAI Studioの機能）。カスタム手順、変数、論理テーブル、ディメンションなどがフォームで並びます。

![ビジュアル表示。カスタム手順・変数・論理テーブルがフォームで並ぶ](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-08-annotated.png)

論理テーブルの説明文には、元のビューに付けていたコメントがそのまま入っていました。コメントを整えておくと、生成される説明文にも反映されるようです。

CoCo のパネルに戻ると、書き込んだ定義を `sv-read` で読み直して確認していました。

```bash
cortex agent-studio sv-read --source workspace --file-path my_semantic.sv.yaml 2>&1 | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(data['yaml_content'][:3000])
"
```

確認が終わると、ビューのサマリと次のアクションの候補が出ます。「The file is now populated but not yet deployed to Snowflake.」とあるとおり、この時点ではまだ Snowflake にオブジェクトはありません。

![ビューのサマリと次のアクションの候補](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-09-annotated.png)

| 候補 | 内容 |
|---|---|
| Deploy | CORTEX_DB.SEMANTIC_MODELS にセマンティックビューを作る |
| Generate descriptions | テーブルと列の説明を AI で生成する |
| Suggest relationships | 2 つのテーブルの結合を検出する |
| Validate | デプロイ前に YAML を検証する |
| Audit | 品質スコアとベストプラクティス違反を確認する |

最後に、CoCo のパネルで「Keep all」を押して変更を確定します。

![Keep allでCoCoの変更を確定する](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-10-annotated.png)

## 手順4: Publish

ここまでの内容は、まだ自分のワークスペース内の下書きです。Snowflake にセマンティックビューを作るには、エディタ上部の「Publish」を押します。

![エディタ上部のPublishボタン](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-11-annotated.png)

ダイアログでターゲット名（初期値は `dev`）を確認し、公開先のデータベースとスキーマを選んで「公開」を押します。

![Publishダイアログ。ターゲット名と公開先スキーマを選ぶ](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-12-annotated.png)

選んだスキーマに、セマンティックビュー `MY_SEMANTIC` が作られました。

![選んだスキーマにMY_SEMANTICが作られた](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-13-annotated.png)

公開後は、エディタ上部のボタンが「プル」と「変更を公開」に変わります。定義を変更したら、「変更を公開」で差分を公開します。

![公開後はプルと変更を公開のボタンに変わる](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-14-annotated.png)

便利なのが、画面下部の検証表示です。エラーと警告の件数と、「有効なセマンティックビュー」という判定が表示されます。[ドキュメント](https://docs.snowflake.com/en/user-guide/views-semantic/validation-rules)を見るとセマンティックビューの記載ルールに従って検証されるようですが、単なるyml構文エラーもきちんと警告してくれるようなので、ちょっとだけ直した時に非常に助かります。

![画面下部の検証表示。エラー0・警告0・有効なセマンティックビュー](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-15-annotated.png)


## ターゲットで公開先を管理する

ターゲットは、公開先のスキーマ等の情報に名前を付けたものです。最初の Publish で付けた `dev` がそれにあたります。

定義を変更したあと「変更を公開」の ▼ を開くと、公開先を選ぶメニューが出ます。既存のターゲットには同期状態が表示され、「Publish to new target...」から新しいターゲットを追加できます。チェックボックスで、複数のターゲットを選べます。

下の画像は、`staging` を新しいターゲットとして追加して公開したあと、変更を加えた状態です。dev・staging とも「Out of sync」になっています。

![変更の公開先メニュー。新しいターゲットも追加できる](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-16-annotated.png)

ターゲットは、`cortex_project` フォルダの `cortex-project.yaml` に保存されます。`targets` の下に、ターゲット名と公開先オブジェクトの組が並びます。

![cortex-project.yaml](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/semantic-studio-17-annotated.png)

公式ドキュメントでは、`cortex-project.yaml` は次のように説明されています。

> A `cortex-project.yaml` file appears automatically in your workspace. This is a project manifest that tracks which files belong to the project and where they deploy to (target database and schema). You can ignore this file. It is auto-managed by Semantic Studio and CoCo.

Semantic Studio と CoCo が自動で管理するファイルのようです。中を見れば、どのファイルがどこに公開されるのかを一覧で確認できます。dev → staging → 本番のように、デプロイ先スキーマを変えたい時に使用できます。

## 裏側で実行されていたクエリ

画面からは、裏でどんな SQL が実行されたのかは見えません。そこで、CoCo を操作した時のクエリを `INFORMATION_SCHEMA.QUERY_HISTORY` で確認しました。

```sql
SELECT TO_CHAR(CONVERT_TIMEZONE('Asia/Tokyo', start_time), 'HH24:MI:SS') AS jst,
       query_type,
       LEFT(REGEXP_REPLACE(query_text, '\\s+', ' '), 120) AS head,
       query_tag
FROM TABLE(CORTEX_DB.INFORMATION_SCHEMA.QUERY_HISTORY(
       END_TIME_RANGE_START => DATEADD('hour', -6, CURRENT_TIMESTAMP()),
       RESULT_LIMIT => 10000))
WHERE TO_CHAR(CONVERT_TIMEZONE('Asia/Tokyo', start_time), 'YYYY-MM-DD HH24:MI')
      BETWEEN '2026-10-02 21:05' AND '2026-10-02 21:55'
ORDER BY start_time;
```

結果の抜粋です。2 つのビューに同じクエリが実行されていたものは、`V_JHU_TIMESERIES` の分だけ載せています。

| 操作内容 | クエリ（抜粋） | QUERY_TAG の app |
|---|---|---|
| テーブル指定 | `ALTER SESSION SET QUERY_TAG = '{"app":"cortex_code_sandbox", ...}'` | （ここで設定） |
| テーブル指定 | `DESCRIBE TABLE RAW_DB.COVID19.V_JHU_TIMESERIES` | cortex_code_sandbox |
| sv-generate 承認 | `SELECT SYSTEM$CORTEX_ANALYST_FAST_GENERATION('{ "json_proto": ... }')` | cortex_code_cli |
| sv-generate 承認 | `SHOW PRIMARY KEYS` / `SHOW UNIQUE KEYS` / `SHOW IMPORTED KEYS IN ...` | cortex_code_cli |
| sv-generate 承認 | `SELECT COUNT(*) FROM RAW_DB.COVID19.V_JHU_TIMESERIES` | cortex_code_cli |
| sv-generate 承認 | `SELECT COUNT(DATE), APPROX_COUNT_DISTINCT(DATE) FROM ...` | cortex_code_cli |
| Publish | `select SYSTEM$WRITE_SEMANTIC_MODEL_YAML('CORTEX_DB.SEMANTIC_MODELS', $$...$$, true)` | なし |

### QUERY_TAG で発行元を見分けられる

CoCo のクエリには `QUERY_TAG` が付いていました。CoCo が直接実行したクエリは `cortex_code_sandbox`、パネルで承認した `cortex agent-studio` コマンドから実行されたクエリは `cortex_code_cli` です。Snowsight 上の CoCo も、コマンドの実行には Cortex Code CLI を使っているようです。Publish 時のクエリにはタグがありませんでした。

### sv-generate の実体は `SYSTEM$CORTEX_ANALYST_FAST_GENERATION`

`sv-generate` を実行した時に、`SYSTEM$CORTEX_ANALYST_FAST_GENERATION` が呼ばれているようです。引数は、手順2 で書き出した json_proto そのものです。

その直後に、同じ `cortex_code_cli` のタグで元のビューを調べるクエリが続きます。調べていたのは、主キー・一意キー・外部キーの定義、件数、`DATE` 列の値の種類数です。キーや時間軸の候補を決める材料にしていると思われます。作成するロールに `SELECT` 権限が必要なのは、こうしたクエリを実行するためでもありそうです。

### Publish 直後の検証らしき呼び出し

Publish した直後には、`SYSTEM$WRITE_SEMANTIC_MODEL_YAML` が 3 回呼ばれていました。第 3 引数はいずれも `true` です。

```sql
select SYSTEM$WRITE_SEMANTIC_MODEL_YAML(
  'CORTEX_DB.SEMANTIC_MODELS',
  $$name: my_semantic
description: COVID-19 analytics model combining JHU time series data ...
tables:
  - name: v_covid19_world_testing
    ...$$,
  true);
```

公式に記載のあるストアドプロシージャ [`SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML`](https://docs.snowflake.com/en/sql-reference/stored-procedures/system_create_semantic_view_from_yaml) は、第 3 引数 `verify_only` に `TRUE` を渡すと、ビューを作らずに YAML の検証だけを行います。

引数の形が同じなので、この 3 回も検証だけを行う呼び出しと考えられます。同じタイミングで、このセマンティックビューに対する Cortex Analyst のリクエストログを取得するクエリ（[`SNOWFLAKE.LOCAL.CORTEX_ANALYST_REQUESTS`](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst/admin-observability)）や、`(date, iso3)` などの列の組み合わせが一意かどうかを数えるクエリも実行されていました。

:::note warn
`SYSTEM$CORTEX_ANALYST_FAST_GENERATION` と `SYSTEM$WRITE_SEMANTIC_MODEL_YAML` は、執筆時点では公式ドキュメントに記載が見つかりませんでした。
:::

## まとめ

- テーブルと作成先を答えるだけで、CoCo がセマンティックビューの定義を作ってくれます。
- ワークスペースでの編集が可能になったことで、CoCoでの編集に加え、手元での編集→デプロイの流れをバージョン管理しながら行えるようになりました。
- ただし、公開操作を行うと単純に上書きされてしまうため、チームでレビューを挟みながら使うなら、Git 連携したワークスペースを使用することになります。
- とはいえAI機能をとりま使いたい！という方は多いのでそういったニーズを満たせる機能だと思いました！
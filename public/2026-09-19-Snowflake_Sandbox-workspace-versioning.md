---
title: 【Snowflake】共有ワークスペースが裏側で何をしているのか調べてみた
tags:
  - Snowflake
  - Snowsight
  - ワークスペース
private: true
updated_at: '2026-09-20T18:15:56+09:00'
id: c652ff825e60da7f975a
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## TL;DR

- 共有ワークスペースの「公開」は、**自分の下書きを確定させて他の人に公開する**操作です
- **UI の公開はファイル単位に見えますが、裏側で管理されている版はワークスペース単位**です。1ファイルだけ公開しても、触っていないファイルを含む新しい版が1つできます。

## 環境

| 項目 | 値 |
|---|---|
| Snowflake | Standardエディション |
| Snowflake CLI | 3.25.0 |
| ワークスペース | Snowsight の Projects » Workspaces |
| 検証対象 | 共有ワークスペース（`CREATE WORKSPACE` で作成したもの） |

## 調べたきっかけ

Snowsight の共有ワークスペースで SQL ファイルを編集していると、右上に `Publish changes` というボタンが出てきます。公開できるっぽいことは分かるのですが、**具体的に何をしているのか知りたくなりました。**

- どこまでが公開されるのか。開いているファイルだけなのか、ワークスペース全体なのか
- 公開する前の状態は、どこに保存されているのか
- 他の人には、いつ何が見えるようになるのか

公式ドキュメントの共有ワークスペースの項には、こう書かれています。

> To make your changes visible to all other collaborators, you must publish the file.
> This is a per-file action that updates the shared version.

📎 https://docs.snowflake.com/en/user-guide/ui-snowsight/workspaces-shared

「per-file action 」ファイル単位の操作。
「the shared version」バージョン単位で公開される。
ファイル単位の操作なのに、ワークスペース全体のバージョン管理があるかのような記載で、モヤりました。

そこで、UI を触りながら、SQL で裏側を覗いて確かめてみました。

## UI から見た公開の仕組み

まず Snowsight 側だけで、ファイルを作って公開するまでを追ってみます。

右上のボタンの動きに注目してください。

| 状態 | 右上のボタン | `Discard changes` | ファイル名の横の青い点 |
|---|---|---|---|
| ① ファイル新規作成直後 | `Publish file`（有効） | 無効 | ● |
| ② そのまま編集 | `Publish file`（有効） | 無効 | ● |
| ③ 公開直後 | `Publish changes`（無効） | 無効 | 消える |
| ④ 公開後に編集 | `Publish changes`（有効） | 有効 | ● |

ポイントは2つあります。

**1つ目は、ボタンのラベルが `Publish file` と `Publish changes` で出し分けられること。** まだ一度も公開していないファイルは `Publish file`（ファイルそのものを公開する）、一度公開済みなら `Publish changes`（変更を公開する）になります。

**2つ目は、`Discard changes` の有効／無効が「戻す先があるか」を示していること。** 一度も公開していないファイルには戻す先の公開版が存在しないので、①②では押せません。

![ファイル新規作成直後は Discard changes が無効](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/ws-publish-new-file-annotated.png)

③で初めて公開版ができ、④でそこからの差分が生まれて、ようやく `Discard changes` が有効になります。

![公開後に編集すると Discard changes が有効になる](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/ws-publish-after-edit-annotated.png)

ファイル名の横の青い点は「未公開の変更がある」印です。公開すると消えます。

## Version history に2つのタブがある

ファイルの `⋯` メニューから `Version history` を開くと、右側に **`Published` と `My drafts` という2つのタブ**があります。

`Published` タブには公開済みの版が並びます。版を選んで `Restore this version` を押せば、過去の状態に戻せます。

![Version history の Published タブ](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/ws-version-history-published-annotated.png)

`My drafts` タブには、まだ公開していない下書きが入っています。未公開の状態にも日時が記録されているのが分かります。

![Version history の My drafts タブ](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/ws-version-history-drafts-annotated.png)

ここまでで、ワークスペースが**「公開済み」と「下書き」の2階層**を持っていることが見えてきました。タブ名が `My drafts`（＝私の下書き）であることから、下書きがユーザー単位で管理されていることを示しています。

ただし、UI から見えるのは、**あくまで開いているファイル1つ分の履歴**です。ワークスペース全体としてどうなっているかは、この画面からは分かりません。

## SQL で補足すると版の実体が見える

ここから SQL で裏側を確認します。ワークスペースの版は `SHOW VERSIONS` で一覧できます。

```sql
SHOW VERSIONS IN WORKSPACE WSSCOPE_DB.WS.T;
```

出力がこちらです（列は抜粋しています）。

```text
| created_on                       | name      | is_live | is_last |
| 2026-09-17 15:15:03.369000-07:00 | None      | true    | false   |
| 2026-09-17 15:15:09.100000-07:00 | VERSION$4 | false   | true    |
| 2026-09-17 15:14:16.103000-07:00 | VERSION$3 | false   | false   |
| 2026-09-17 15:12:42.114000-07:00 | VERSION$2 | false   | false   |
| 2026-09-17 15:11:52.476000-07:00 | VERSION$1 | false   | false   |
```

UI の2つのタブが、そのままここに対応していました。

| UI の見え方 | SQL での実体 |
|---|---|
| `Published` タブ | `VERSION$1`、`VERSION$2`、… という**不変の版** |
| `My drafts` タブ | `is_live = true` の行（名前を持たない**可変の下書き**） |

下書きは `live` と呼ばれ、**ワークスペースに1つだけ**存在します。ファイルの置き場所も URI で表現されていて、`snow://workspace/<DB>.<スキーマ>.<WS名>/versions/live/` が下書き、`versions/VERSION$2/` が公開済みの版です。

つまりワークスペースの中身は、こういう構造になっています。

- `live` — 編集中の下書き。可変。1つだけ
- `VERSION$N` — 公開済みの履歴。不変。公開するたびに1つ増える

UI だけを使っていると「ファイルごとに履歴がある」ように見えますが、SQL で見ると**番号が振られているのはワークスペース全体**です。ここが今回いちばん分かりにくかったところでした。

## 公開の単位はファイルか、ワークスペースか？

**UI の公開はファイル単位に見えるのに、SQL で見える版はワークスペース単位**。この食い違いを確かめます。

### SQL から公開すると、ワークスペース単位になる

まず SQL 側から。`a.sql` と `b.sql` の2ファイルを置いて、`COMMIT` を1回だけ実行します。

```bash
snow sql -c main --role SYSADMIN -q "ALTER WORKSPACE WSSCOPE_DB.WS.T ADD LIVE VERSION FROM LAST"
snow sql -c main --role SYSADMIN -q "PUT file:///tmp/a.sql snow://workspace/WSSCOPE_DB.WS.T/versions/live/ AUTO_COMPRESS=false OVERWRITE=true"
snow sql -c main --role SYSADMIN -q "PUT file:///tmp/b.sql snow://workspace/WSSCOPE_DB.WS.T/versions/live/ AUTO_COMPRESS=false OVERWRITE=true"
snow sql -c main --role SYSADMIN -q "ALTER WORKSPACE WSSCOPE_DB.WS.T COMMIT"
```

できた `VERSION$2` の中身を見ると、**両方のファイルが入っています**。版は1つしか増えていません。

```text
| name                      | size | last_modified                 |
| /versions/version$2/a.sql | 96   | Thu, 17 Sep 2026 22:12:34 GMT |
| /versions/version$2/b.sql | 96   | Thu, 17 Sep 2026 22:12:38 GMT |
```

そもそも `COMMIT` にファイル名を渡す構文がありません。試すと構文エラーになります。

```text
001003 (42000): SQL compilation error:
syntax error line 1 at position 39 unexpected ''a.sql''.
```

**SQL 側の公開は、ワークスペース単位そのものです。** 下書きに置いてあるものが丸ごと1つの版になります。

### UI から1ファイルだけ公開しても、版は全体で増える

では UI 側はどうか。ワークスペースを再作成し同様にSQLファイルを配置・公開した状態で、 Snowsight から `d.sql` を1つ追加し、**そのファイルだけを Publish** します。公式が「per-file action」と呼んでいる操作です。

その後、できた版の中身を SQL で確認します。

```bash
snow sql -c main --role SYSADMIN -q "LIST 'snow://workspace/WSSCOPE_DB.WS.T/versions/VERSION\$4/'"
```

```text
| name                      | size | last_modified                 |
| /versions/version$4/a.sql | 96   | Thu, 17 Sep 2026 09:38:07 GMT |
| /versions/version$4/b.sql | 96   | Thu, 17 Sep 2026 09:40:38 GMT |
| /versions/version$4/c.sql | 96   | Thu, 17 Sep 2026 21:58:05 GMT |
| /versions/version$4/d.sql | 16   | Thu, 17 Sep 2026 22:00:03 GMT |
```

**`d.sql` 1つを公開しただけなのに、4ファイルすべてが入った版ができました。**

各ファイルの `last_modified` は元の時刻のままです。`a.sql` と `b.sql` は今回まったく触っていませんが、新しい版にはそのまま引き継がれています。

### 結論

整理するとこうなります。

| | 操作の単位 | できる版 |
|---|---|---|
| **UI（Snowsight）** | ファイル単位（どのファイルの変更を公開に含めるか選べる） | **ワークスペース全体のスナップショット** |
| **SQL** | ワークスペース単位（ファイルを指定する構文がない） | **ワークスペース全体のスナップショット** |

公式の「per-file action」は、**どのファイルの変更を公開に含めるかをユーザーが選べる**という操作の粒度を指していたわけです。物理的な版は、どちらから公開しても常にワークスペース全体になります。

UI が版をファイル単位でしか表示しないため、ファイルごとに独立した履歴があるように感じますが、実体は1本の全体履歴です。**SQL で版の中身を `LIST` して初めて、この挙動がはっきり分かりました。**

## SQL から操作するときだけ必要になること

最後に、（あまりないユースケースのように感じますが、）SQL から触る場合の注意点です。

### 公開のたびに `ADD LIVE VERSION` が要る

`COMMIT` すると下書きは消費されてなくなります。**そして SQL の場合、次の下書きは自動では作られません。**

公開直後に `SHOW VERSIONS` を見ても、`is_live = true` の行はありません。この状態で `PUT` するとエラーになります。

```text
099108 (22000): Live version is not found.
```

一方、Snowsight から公開した場合は、サーバー側が次の下書きを用意してくれてるようです。UI に `ADD LIVE VERSION` に相当する操作が存在しないのはそのためです。

| クライアント | 公開後の下書き | `ADD LIVE VERSION` |
|---|---|---|
| Snowsight | 自動で用意される | 不要 |
| SQL / Snowflake CLI | **作られない** | **公開のたびに必要** |

SQL で運用するなら、**「開く → 置く → 公開」を毎回1セット**で回すことになります。

```sql
ALTER WORKSPACE <WS> ADD LIVE VERSION FROM LAST;  -- 開く
PUT file:///path/to/x.sql snow://workspace/<WS>/versions/live/ AUTO_COMPRESS=false OVERWRITE=true;
ALTER WORKSPACE <WS> COMMIT;                      -- 公開する
```

なお `FROM LAST` は、**直前の版の中身をそのまま下書きに引き継ぎます**。だから1ファイルだけ差し替えても、残りのファイルが消えることはありません。

### 過去の版には書き込めない

`VERSION$N` を宛先にして `PUT` しようとすると、はっきり断られます。

```text
099115 (42000): The provided location is not writable. Please specify a live version.
```

「live を指定してください」とメッセージに書かれているとおり、**`PUT` の宛先は下書きだけ**です。公開済みの版が不変であることが、エラーメッセージからも確認できます。

### SQL で操作しても UI の表示は追従する

ちなみに、SQL と UI は同じ状態を見ています。Snowsight を開いたまま SQL で `PUT` すると、ファイル名の横に青い点が現れます。

![SQL の PUT 後に UI で未公開の印が付く](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/ws-sql-put-dot-annotated.png)

`COMMIT` すれば、その点が消えます。

![SQL の COMMIT 後に UI の印が消える](https://raw.githubusercontent.com/alfortx/qiita-articles/main/public/ws-sql-commit-nodot-annotated.png)

Snowsight を一切触らずに SQL だけで操作しても、UI の表示は同じでした。入り口が違うだけで両者は同じものを見ていると言うことです。

## まとめ

共有ワークスペースのファイル公開について、調べて分かったことをまとめます。

- 公開は「自分の下書きを確定させて、他の人に見せる」操作
- ワークスペースは **`live`（可変の下書き）と `VERSION$N`（不変の公開版）の2階層**を持つ。UI の `My drafts` / `Published` タブがこれに対応する
- **UI の公開はファイル単位に見えるが、できる版はワークスペース全体のスナップショット**。1ファイルだけ公開しても、触っていないファイルを含む版が1つ増える
- **SQL の公開はワークスペース単位そのもの**。`COMMIT` にファイルを指定する構文はない

共有ワークスペースは共有したり版管理ができると言うふんわりとしたイメージで使っていましたが今回の検証で解像度が上がりました。
ワークスペースは、StreamlitやAI機能の開発に対応したりと最近機能拡充が目覚ましいですね。今後もどんどん便利になっていくことを期待しつつキャッチアップを続けます！

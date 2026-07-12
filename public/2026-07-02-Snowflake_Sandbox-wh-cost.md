---
title: Snowflake 自腹有料アカウントの財布防衛記録 ― ウェアハウスコスト設定に向き合ってみた
tags:
  - SQL
  - コスト
  - Snowflake
  - ウェアハウス
private: false
updated_at: '2026-07-04T11:24:37+09:00'
id: d4e16fbb560c45780550
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## TL;DR

- 個人の Snowflake アカウントを有料化したタイミングで、ウェアハウスのコスト設定を見直した
- 見直す設定は3つ：**`AUTO_SUSPEND`の短縮 / クエリタイムアウト / リソースモニター**
- アカウント初期作成時に自動生成される `COMPUTE_WH` は `AUTO_SUSPEND = 300`（5分）でコスト管理の穴になりやすい。まず最初にここを確認する

---

## 背景

個人学習をトライアルアカウントで行ってきたが、最近 AI 機能が有料化されたことで、有料アカウントへ契約をした。もちろん自腹。コスト意識が高まり、コストを抑える仕組みを調査してみた。

[クレジット使用状況]![スクリーンショット 2026-07-04 11.22.44.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4076823/1ba331b3-9c88-44ac-ae33-8b836780cd92.png
)

大した消費ではない。しかし日々のコーヒー代の足しにできるお金が流れていると思うと。。。

---

## ウェアハウスの課金時間の考え方

Snowflake のウェアハウスは、クエリを実行したタイミングで起動し、**稼働している時間に対して課金**される。課金に影響するポイントは主に2つある。

**ポイント① アイドル時間も課金される**

クエリが終わっても、ウェアハウスはすぐには停止しない。`AUTO_SUSPEND` に設定した秒数だけ起動し続け、その間も課金が発生する。使用頻度が低い場合、この「待機時間の料金」が積み上がりやすい。

**ポイント② クエリが終わるまで起動し続ける**

重いクエリを流すとウェアハウスはクエリが完了するまで動き続ける。うっかり大規模スキャンや長時間クエリを流してしまった場合、手動でキャンセルしないと延々と課金が続く。

---

## 見直す設定3つ

上記2つのポイントに対処する設定と、月単位のコスト上限を設けるリソースモニターの計3つを整えておくと安心できる。

---

### 1. `AUTO_SUSPEND` を短縮する（ポイント①対策）

ウェアハウスはクエリが終わっても `AUTO_SUSPEND` の秒数だけ起動し続ける。短くするほどアイドル課金は下がる。

```sql
-- 新規作成時に設定する場合
CREATE WAREHOUSE MY_WH
  WAREHOUSE_SIZE      = 'XSMALL'
  AUTO_SUSPEND        = 60       -- 1 分でサスペンド
  AUTO_RESUME         = TRUE
  INITIALLY_SUSPENDED = TRUE;
```

```sql
-- 既存のウェアハウスを変更する場合
ALTER WAREHOUSE MY_WH SET AUTO_SUSPEND = 60;
```

`AUTO_RESUME = TRUE` を合わせて設定しておけば、次のクエリ実行時に自動で再起動するので使い勝手は変わらない。

`INITIALLY_SUSPENDED = TRUE` は CREATE 時のオプションで、作成直後に起動しない設定。Snowsight から作成すると自動で起動してしまうことがあるため、明示的に指定するのが安全。

---

### 2. クエリタイムアウトを設定する（ポイント②対策）

`STATEMENT_TIMEOUT_IN_SECONDS` を設定すると、指定秒数を超えたクエリが自動でキャンセルされる。

```sql
ALTER WAREHOUSE MY_WH SET STATEMENT_TIMEOUT_IN_SECONDS = 180; -- 3 分でキャンセル
```

3 分（180 秒）は一つの目安。意図的に長時間クエリを流す場合は、その都度 `ALTER` で伸ばすか、用途別に別ウェアハウスを用意するのがよい。

---

### 3. リソースモニターで月次上限を設定する

`AUTO_SUSPEND` と `STATEMENT_TIMEOUT` はウェアハウス単体の守りだが、**月全体のクレジット使用量に上限を設ける**のがリソースモニター。設定した上限に達するとウェアハウスが自動で停止する。

```sql
-- ACCOUNTADMIN で実行（リソースモニターの作成には ACCOUNTADMIN が必要）
USE ROLE ACCOUNTADMIN;

CREATE RESOURCE MONITOR MY_MONITOR
  WITH
    CREDIT_QUOTA    = 10        -- 月 10 クレジットで上限
    FREQUENCY       = MONTHLY
    START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 80  PERCENT DO NOTIFY           -- 80% でメール通知
    ON 90  PERCENT DO NOTIFY           -- 90% でメール通知
    ON 100 PERCENT DO SUSPEND          -- 100% で実行中クエリ完了後にサスペンド
    ON 110 PERCENT DO SUSPEND_IMMEDIATE; -- 110% で即時サスペンド
```

作成したリソースモニターをウェアハウスに紐付ける。

```sql
-- SYSADMIN で実行
USE ROLE SYSADMIN;
ALTER WAREHOUSE MY_WH SET RESOURCE_MONITOR = MY_MONITOR;
```

:::note warn
リソースモニターの作成は ACCOUNTADMIN、ウェアハウスへの紐付けは SYSADMIN と、必要なロールが異なる。SYSADMIN のみで両方やろうとするとエラーになる。
:::

---

## まず確認：COMPUTE_WH の auto_suspend

アカウントを作成したらもれなくついてくる `COMPUTE_WH`。こいつの `AUTO_SUSPEND` がデフォルトで **300 秒（5 分）**。クエリが終わっても 5 分間は課金が続く計算になる。まずはこれをなんとかしよう。

現在の設定を確認する。

```sql
SHOW WAREHOUSES LIKE 'COMPUTE_WH';
```

`auto_suspend` 列が `300` になっていたら、以下のどちらかを推奨する。

**① AUTO_SUSPEND を短縮する**

```sql
-- SYSADMIN で実行
USE ROLE SYSADMIN;
ALTER WAREHOUSE COMPUTE_WH SET AUTO_SUSPEND = 60;
```

**② 別のウェアハウスを作って COMPUTE_WH を削除する**

新しいウェアハウスをコスト最適設定で作り直し、`COMPUTE_WH` を削除する方がスッキリする。既存のユーザーが `COMPUTE_WH` をデフォルトに設定したままの場合は削除前に付け替えが必要。

```sql
-- 1. COMPUTE_WH をデフォルトにしているユーザーを確認
SHOW USERS;
-- → DEFAULT_WAREHOUSE = 'COMPUTE_WH' になっているユーザーを確認

-- 2. 別のウェアハウスに付け替え（ユーザー名は実際のものに変更）
ALTER USER <username> SET DEFAULT_WAREHOUSE = 'MY_WH';

-- 3. COMPUTE_WH を削除
DROP WAREHOUSE IF EXISTS COMPUTE_WH;
```

---

## まとめ

| 設定 | 変更のポイント | 効果 |
|---|---|---|
| `AUTO_SUSPEND` | 300 秒 → **60 秒** | アイドル課金を最小化（ポイント①対策） |
| `STATEMENT_TIMEOUT` | 未設定 → **180 秒** | クエリ暴走を自動キャンセル（ポイント②対策） |
| リソースモニター | 未設定 → **月次クレジット上限** | 月単位のコスト暴走を防止 |
| `COMPUTE_WH` の確認 | `AUTO_SUSPEND = 300` なら短縮か削除 | 初期アカウントの課金漏れをふさぐ |

Snowflake はウェアハウスが動いている時間だけ課金される仕組みなので、「止め方」の設計がコストに直結する。有料化のタイミングでまとめて整えておくと後で安心できる。

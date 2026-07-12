---
title: Snowsightにログインしただけでウェアハウスが起動する件を調べた
tags:
  - コスト
  - Snowflake
  - Snowsight
  - ウェアハウス
private: false
updated_at: '2026-07-12T20:08:32+09:00'
id: 17ab5a97bd1571532e0c
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## TL;DR

- Snowsight にログインしただけでウェアハウスが起動することがある
- クエリ履歴を見てもヒットしないのは「Client-generated statements（クライアント生成ステートメント）」がデフォルトのフィルターから除外されているため
- `Monitoring » Query History` のフィルターで `Client-generated statements` を有効にすると確認できる

---

## 状況

Snowsight を開いた直後、何もクエリを打っていないのに、ウェアハウスが「実行中」になっていた。

![Snowsightログイン直後のウェアハウス起動画面](./img_snowsightログイン直後のウェアハウス起動画面.png)

ウェアハウスは起動した瞬間から**最低1分課金**される。使っていないのに毎回これが発生しているとすると、地味に積み上がる・・・

---

## クエリ履歴を確認してもヒットしない

原因を調べようとウェアハウスのクエリ履歴を確認したが、何も出てこない。

![ウェアハウスは起動したのにクエリ履歴には何も出てこない](./img_ウェアハウスは起動したのにクエリ履歴には何も出てこない.png)

ウェアハウスが起動した記録はある。しかしクエリが見当たらない。

---

## 原因：Client-generated statements という仕様

原因がわからなかったのでサポートに問い合わせたところ、既知の仕様とのことだった。

Snowsight の一部のページ（Task Run History や Data Preview など）は、ページを開いた時点でデータ取得のための SQL を自動発行する。`AUTO_RESUME = TRUE` のウェアハウスが紐付いていれば、このタイミングで自動起動する。

この自動発行されるクエリが「**Client-generated statements（クライアント生成ステートメント）**」と呼ばれる分類で、デフォルトのクエリ履歴フィルターには**表示されない**設定になっている。だからクエリ履歴を見ても何もヒットしなかった。

---

## 確認方法：フィルターで表示する

`Monitoring » Query History` を開き、Filters ドロップダウンから `Client-generated statements` チェックボックスを有効にして `Apply Filters` を押す。

![クライアント生成クエリ検索条件](./img_クライアント生成クエリ検索条件.png)

すると、先ほどは見えなかったクエリが出てきた。

![モニタリング画面からクエリを確認できた](./img_モニタリング画面からクエリを確認できた.png)

Snowsight が裏側で発行していたクエリがここで確認できる。

---

## まとめ

| 現象 | 原因 | 確認方法 |
|---|---|---|
| ログインしただけでWHが起動 | Snowsightが一部ページでSQLを自動発行 | 仕様なので回避は難しい |
| クエリ履歴にヒットしない | Client-generated statementsはデフォルトフィルター外 | `Client-generated statements` フィルターを有効化 |

**推奨対応**：

- Snowsight 専用に X-Small のウェアハウスを用意する（本番や分析用と分離する）
- `AUTO_SUSPEND` を短めに設定する（例：60秒）と、Snowsight 起動による課金が最小限になる

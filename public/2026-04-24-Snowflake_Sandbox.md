---
title: 'Snowflake Cortex Search Service で企業名の表記揺れを名寄せする方法・おかわり'
tags:
  - Snowflake
  - CortexSearch
  - 自然言語処理
  - データエンジニアリング
  - 名寄せ
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## TL;DR

- 先日、企業名の名寄せって大変ですよね・・・という会話の中で、Snowflakeの有識者から「 **Cortex Search Service** でできるんじゃないですか？」と言われ、「たしかに！」と思ってやってみた。
- 実は[前編記事](https://qiita.com/ikageso17g/items/a20eff6b3cb14b2e97c7)もあり。今回はオープンデータを使ってより実務に近いユースケースを想定した。

---

## 環境

| 項目 | 内容 |
|------|------|
| Snowflake Edition | Enterprise |
| Cortex Search Service | `snowflake-arctic-embed-l-v2.0`（デフォルト） |
| データソース | EDINET（金融庁）・JPX（日本取引所） |
| 実行環境 | Snowflake Notebooks |

---

## 背景・課題

### データソースの概要

**EDINET** は金融庁が運営する有価証券報告書の電子開示システム。上場企業だけでなく非上場の届出企業なども含め約1万件以上が登録されている。各社に一意の `EDINET_CODE`（例: `E00001`）が付与され、証券コードは上場企業にのみ存在する。

**JPX** は東京証券取引所（東証）が公開する上場銘柄一覧。4桁の証券コードで約4,000〜4,500銘柄を管理している。

#### EDINET テーブル（サンプル）

| EDINET_CODE | COMPANY_NAME_JA | SECURITIES_CODE |
|---|---|---|
| E00004 | カネコ種苗株式会社 | 13760 |
| E00006 | 株式会社　サカタのタネ | 13770 |
| E00007 | ユキグニファクトリー株式会社 | 13750 |
| E00008 | ホクト株式会社 | 13790 |
| E00009 | 株式会社アクシーズ | 13810 |

#### JPX テーブル（サンプル）

| SECURITIES_CODE | COMPANY_NAME | MARKET | INDUSTRY_33 |
|---|---|---|---|
| 1301 | 極洋 | プライム（内国株式） | 水産・農林業 |
| 1305 | ｉＦｒｅｅＥＴＦ　ＴＯＰＩＸ（年１回決算型） | ETF・ETN | - |
| 1332 | ニッスイ | プライム（内国株式） | 水産・農林業 |
| 1375 | ユキグニファクトリー | スタンダード（内国株式） | 水産・農林業 |
| 1377 | サカタのタネ | プライム（内国株式） | 水産・農林業 |

### 名寄せの課題

2つのデータソースを突合しようとすると、同じ企業でも社名の表記が一致しないケースが頻出する。

| パターン | JPX 表記例ー | EDINET 表記例 |
|----------|-----------|---------------|
| 株式会社の位置 | サカタのタネ | 株式会社　サカタのタネ  |
| 略称 vs 正式名 | ＳＧＨＤ | ＳＧホールディングス株式会社 |
| 全角/半角混在 | ＡＮＡホールディングス | ANAホールディングス株式会社 |

→ **Cortex Search Service のセマンティック検索で表記ゆれを吸収できないか**を実験した。

---

## 実験フロー

| STEP | 処理 | 出力 |
|------|------|------|
| 0 | データ品質確認（件数・NULL率・証券コード保有率） | — |
| 1 | 正解データ作成（SECURITIES_CODE 結合） | `EDINET_JPX_GROUND_TRUTH` |
| 2 | Cortex Search Service 作成（EDINET 社名をインデックス化） | `EDINET_COMPANY_SEARCH` |
| 3 | 単件確認（任意の社名で動作確認） | — |
| 4 | 全件マッチング実行 | `EDINET_JPX_MATCH_RESULT` |
| 5 | 精度評価（Precision@1 / Recall@3） | — |

---

## 実装手順

### STEP 0: データ品質確認

件数・NULL率・突合可能件数を事前に確認しておく。

```sql
-- EDINET × JPX の証券コード突合可能件数を確認
SELECT
  COUNT(DISTINCT e.SECURITIES_CODE) AS edinet_with_securities_code,
  COUNT(DISTINCT j.SECURITIES_CODE) AS jpx_codes,
  COUNT(DISTINCT CASE WHEN j.SECURITIES_CODE IS NOT NULL THEN e.SECURITIES_CODE END) AS matched_count
FROM RAW_DB.COMPANY_MATCHING.MV_EDINET_COMPANIES e
LEFT JOIN RAW_DB.COMPANY_MATCHING.MV_JPX_COMPANIES j
  ON LPAD(e.SECURITIES_CODE::VARCHAR, 4, '0') = LPAD(j.SECURITIES_CODE::VARCHAR, 4, '0')
WHERE e.SECURITIES_CODE IS NOT NULL AND e.SECURITIES_CODE != '';
```

出力結果:

| EDINET_WITH_SECURITIES_CODE | JPX_CODES | MATCHED_COUNT |
|---|---|---|
| 3876 | 3775 | 3769 |

EDINET 側で証券コードを持つ企業 3,876 社のうち、JPX と突合可能なのは 3,769 社。

---

### STEP 1: 正解データ作成

証券コードで完全一致結合できるペアを「グランドトゥルース（正解ラベル）」として作成する。これが精度評価の分母になる。

```sql
CREATE OR REPLACE TABLE SANDBOX_DB.WORK.EDINET_JPX_GROUND_TRUTH AS
SELECT
    j.SECURITIES_CODE,
    j.COMPANY_NAME       AS JPX_NAME,
    e.COMPANY_NAME_JA    AS EDINET_NAME,
    e.EDINET_CODE,
    e.CORPORATE_NUMBER,
    j.INDUSTRY_33        AS JPX_INDUSTRY
FROM RAW_DB.COMPANY_MATCHING.MV_JPX_COMPANIES j
INNER JOIN RAW_DB.COMPANY_MATCHING.MV_EDINET_COMPANIES e
    ON LPAD(j.SECURITIES_CODE::VARCHAR, 4, '0') = LPAD(e.SECURITIES_CODE::VARCHAR, 4, '0')
WHERE j.SECURITIES_CODE IS NOT NULL
  AND j.SECURITIES_CODE != '-';
```

出力結果（先頭5件）:

| SECURITIES_CODE | JPX_NAME | EDINET_NAME | EDINET_CODE |
|---|---|---|---|
| 1376 | カネコ種苗 | カネコ種苗株式会社 | E00004 |
| 1377 | サカタのタネ | 株式会社　サカタのタネ | E00006 |
| 1375 | ユキグニファクトリー | ユキグニファクトリー株式会社 | E00007 |
| 1379 | ホクト | ホクト株式会社 | E00008 |
| 1381 | アクシーズ | 株式会社アクシーズ | E00009 |

全 3,775 ペアの正解データが作成された。

---

### STEP 2: Cortex Search Service 作成

Cortex Search Service は SQLで作成できるスキーマオブジェクト。

```sql
-- ① 検索用テーブルを通常テーブルとして作成（MV から CTAS）
CREATE TABLE IF NOT EXISTS SANDBOX_DB.WORK.EDINET_COMPANIES_FOR_SEARCH AS
SELECT COMPANY_NAME_JA, EDINET_CODE, SECURITIES_CODE, CORPORATE_NUMBER, COMPANY_NAME_KANA
FROM RAW_DB.COMPANY_MATCHING.MV_EDINET_COMPANIES
WHERE COMPANY_NAME_JA IS NOT NULL;

-- ② Search Service 作成（数分かかる、ACTIVE になるまで待つ）
CREATE OR REPLACE CORTEX SEARCH SERVICE CORTEX_DB.SEARCH_SERVICES.EDINET_COMPANY_SEARCH
    ON COMPANY_NAME_JA                                                      -- ベクトル化対象
    ATTRIBUTES EDINET_CODE, SECURITIES_CODE, CORPORATE_NUMBER, COMPANY_NAME_KANA  -- 返却カラム
    WAREHOUSE = SANDBOX_WH
    TARGET_LAG = '1 hour'
    AS (
        SELECT COMPANY_NAME_JA, EDINET_CODE, SECURITIES_CODE, CORPORATE_NUMBER, COMPANY_NAME_KANA
        FROM SANDBOX_DB.WORK.EDINET_COMPANIES_FOR_SEARCH
    );

-- ③ ACTIVE になったか確認
SHOW CORTEX SEARCH SERVICES IN DATABASE CORTEX_DB;
```

ハマったポイント：ターゲットテーブルの差分を一定周期で取りに行く機能に **Dynamic Table** を内部で使っており、マテリアライズドビューや外部テーブルをソースにできない。これらをインプットにしたい場合はテーブル化しておく必要あり。

---

### STEP 3: 単件確認（SEARCH_PREVIEW）

全件処理の前に、`SEARCH_PREVIEW()` で単件の動作確認をする。

`SNOWFLAKE.CORTEX.SEARCH_PREVIEW()` は Cortex Search Service に対して **SQL から直接クエリを実行できる関数**。引数にサービス名と JSON 形式のクエリパラメータ（`query`, `columns`, `limit` 等）を渡すと、検索結果を JSON で返す。ワークシートや Notebook 上でサービスの動作を手軽に確認できるが、引数はリテラル（定数）のみで列参照は不可のため、**テスト・検証用途**に限られる。

```sql
-- 結果を行展開して見やすく表示
WITH raw AS (
    SELECT PARSE_JSON(
        SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
            'CORTEX_DB.SEARCH_SERVICES.EDINET_COMPANY_SEARCH',
            '{"query": "トヨタ自動車", "columns": ["COMPANY_NAME_JA","EDINET_CODE","SECURITIES_CODE"], "limit": 3}'
        )
    ):results AS results
)
SELECT
    f.value:COMPANY_NAME_JA::VARCHAR AS EDINET_NAME,
    f.value:EDINET_CODE::VARCHAR     AS EDINET_CODE,
    f.value:SECURITIES_CODE::VARCHAR AS SECURITIES_CODE,
    f.value:score::FLOAT             AS SCORE,
    ROW_NUMBER() OVER (ORDER BY f.value:score::FLOAT DESC) AS RANK
FROM raw, LATERAL FLATTEN(INPUT => results) f;
```

出力結果:

| EDINET_NAME | EDINET_CODE | SECURITIES_CODE | RANK |
|---|---|---|---|
| トヨタ自動車株式会社 | E02144 | 72030 | 1 |
| いすゞ自動車株式会社 | E02143 | 72020 | 2 |
| 国際自動車株式会社 | E41160 | | 3 |

「トヨタ自動車」で検索し、正しく「トヨタ自動車株式会社」が1位で返ってきている！

---

### STEP 4: 全件マッチング

`CORTEX_SEARCH_BATCH()` テーブル関数を使い、カンマ結合（lateral join 相当）で各行に対して検索結果を展開する。

```sql
CREATE OR REPLACE TABLE SANDBOX_DB.WORK.EDINET_JPX_MATCH_RESULT AS
SELECT
    gt.SECURITIES_CODE                  AS JPX_SECURITIES_CODE,
    gt.JPX_NAME,
    gt.EDINET_NAME                      AS GT_EDINET_NAME,
    r.COMPANY_NAME_JA                   AS MATCHED_EDINET_NAME,
    r.EDINET_CODE                       AS MATCHED_EDINET_CODE,
    r.SECURITIES_CODE                   AS MATCHED_SECURITIES_CODE,
    r.METADATA$RANK                     AS MATCH_RANK,
    CASE
        WHEN LPAD(r.SECURITIES_CODE::VARCHAR, 4, '0')
           = LPAD(gt.SECURITIES_CODE::VARCHAR, 4, '0')
        THEN 'CORRECT'
        ELSE 'WRONG'
    END AS JUDGEMENT
FROM SANDBOX_DB.WORK.EDINET_JPX_GROUND_TRUTH AS gt,
TABLE(CORTEX_SEARCH_BATCH(
    service_name => 'CORTEX_DB.SEARCH_SERVICES.EDINET_COMPANY_SEARCH',
    query        => gt.JPX_NAME,
    limit        => 3
)) AS r;
```

`METADATA$RANK` が各企業に対するスコア順の順位（1が最も類似度高い）。
全件処理は数分かかる。実務においてもパフォーマンスは考慮する必要がありそう。WHサイズを上げるか、今回のようにあらかじめマッチング結果をテーブル化しておくか。

出力結果（先頭6件）:

| JPX_SECURITIES_CODE | JPX_NAME | GT_EDINET_NAME | MATCHED_EDINET_NAME | MATCH_RANK | JUDGEMENT |
|---|---|---|---|---|---|
| 1301 | 極洋 | 株式会社　極洋 | 株式会社　極洋 | 1 | CORRECT |
| 1301 | 極洋 | 株式会社　極洋 | ＧＸ　ＰＡＲＴＮＥＲＳ　ＣＯ．，　ＬＩＭＩＴＥＤ | 2 | WRONG |
| 1301 | 極洋 | 株式会社　極洋 | ＧＯＬＯＮＧ　ＨＯＬＤＩＮＧ　ＣＯ．，ＬＩＭＩＴＥＤ | 3 | WRONG |
| 130A | Ｖｅｒｉｔａｓ　Ｉｎ　Ｓｉｌｉｃｏ | 株式会社Ｖｅｒｉｔａｓ　Ｉｎ　Ｓｉｌｉｃｏ | 株式会社Ｖｅｒｉｔａｓ　Ｉｎ　Ｓｉｌｉｃｏ | 1 | CORRECT |
| 130A | Ｖｅｒｉｔａｓ　Ｉｎ　Ｓｉｌｉｃｏ | 株式会社Ｖｅｒｉｔａｓ　Ｉｎ　Ｓｉｌｉｃｏ | ＧＸ　ＰＡＲＴＮＥＲＳ　ＣＯ．，　ＬＩＭＩＴＥＤ | 2 | WRONG |
| 130A | Ｖｅｒｉｔａｓ　Ｉｎ　Ｓｉｌｉｃｏ | 株式会社Ｖｅｒｉｔａｓ　Ｉｎ　Ｓｉｌｉｃｏ | ＧＯＬＯＮＧ　ＨＯＬＤＩＮＧ　ＣＯ．，ＬＩＭＩＴＥＤ | 3 | WRONG |

全 3,775 企業 × 3 候補 = 11,325 行の結果テーブルが生成された。


---

### 名寄せ結果の例

STEP 5 の精度評価の前に、マッチ結果の具体例を確認する。

#### うまくいっている例（CORRECT）

| JPX名 | 正解（EDINET名） | 名寄せ結果 |
|---|---|---|
| 極洋 | 株式会社　極洋 | 株式会社　極洋 |
| ニッスイ | 株式会社ニッスイ | 株式会社ニッスイ |
| トヨタ自動車 | トヨタ自動車株式会社 | トヨタ自動車株式会社 |
| ソニーグループ | ソニーグループ株式会社 | ソニーグループ株式会社 |
| 任天堂 | 任天堂株式会社 | 任天堂株式会社 |

「株式会社」の有無や位置の違いを正しく吸収して名寄せできている。

#### うまくいっていない例（WRONG）

| JPX名 | 正解（EDINET名） | 名寄せ結果 |
|---|---|---|
| 積水ハウス | 積水ハウス株式会社 | 積水ハウス・リート投資法人 |
| 高松コンストラクショングループ | 株式会社髙松コンストラクショングループ | 高松機械工業株式会社 |
| ヤマト | 株式会社ヤマト | ヤマト　モビリティ　＆　Ｍｆｇ．株式会社 |
| 豆蔵 | 株式会社豆蔵 | 株式会社豆蔵ホールディングス |
| ＭＦＳ | 株式会社ＭＦＳ | ＭＦＳインベストメント・マネジメント株式会社 |

類似名称の別法人（ホールディングス等）や社名変更で誤マッチするパターンが見られる。
これは人間がやっても間違えそう。

---

### STEP 5: 精度評価

**Precision@1**（1位候補だけを採用した場合の正解率）

```sql
WITH best AS (
    SELECT * FROM SANDBOX_DB.WORK.EDINET_JPX_MATCH_RESULT WHERE MATCH_RANK = 1
),
eval AS (
    SELECT
        CASE
            WHEN LPAD(g.SECURITIES_CODE::VARCHAR, 4, '0') = LPAD(b.MATCHED_SECURITIES_CODE::VARCHAR, 4, '0')
            THEN 'CORRECT'
            ELSE 'INCORRECT'
        END AS RESULT
    FROM SANDBOX_DB.WORK.EDINET_JPX_GROUND_TRUTH g
    LEFT JOIN best b
        ON LPAD(g.SECURITIES_CODE::VARCHAR, 4, '0') = LPAD(b.JPX_SECURITIES_CODE::VARCHAR, 4, '0')
)
SELECT
    RESULT,
    COUNT(*) AS CNT,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS PCT
FROM eval
GROUP BY RESULT;
```

出力結果:

| RESULT | CNT | PCT |
|---|---|---|
| CORRECT | 3675 | 97.0 |
| INCORRECT | 114 | 3.0 |

やるやん！

**Recall@3**（上位3候補以内に正解が含まれる率）

```sql
WITH top3 AS (
    SELECT * FROM SANDBOX_DB.WORK.EDINET_JPX_MATCH_RESULT WHERE MATCH_RANK <= 3
),
hit AS (
    SELECT
        g.SECURITIES_CODE,
        MAX(CASE
            WHEN LPAD(g.SECURITIES_CODE::VARCHAR, 4, '0') = LPAD(t.MATCHED_SECURITIES_CODE::VARCHAR, 4, '0')
            THEN 1 ELSE 0
        END) AS IN_TOP3
    FROM SANDBOX_DB.WORK.EDINET_JPX_GROUND_TRUTH g
    JOIN top3 t ON LPAD(g.SECURITIES_CODE::VARCHAR, 4, '0') = LPAD(t.JPX_SECURITIES_CODE::VARCHAR, 4, '0')
    GROUP BY g.SECURITIES_CODE
)
SELECT
    SUM(IN_TOP3)                              AS HIT,
    COUNT(*)                                  AS TOTAL,
    ROUND(SUM(IN_TOP3) * 100.0 / COUNT(*), 1) AS RECALL_AT_3_PCT
FROM hit;
```

出力結果:

| HIT | TOTAL | RECALL_AT_3_PCT |
|---|---|---|
| 3739 | 3775 | 99.0 |

やるやん！


---

## まとめ

Snowflake Cortex Search Service を使った企業名名寄せは、シンプルにある程度の精度で実装できた。株式会社の有無程度であればほぼ完璧に名寄せできた。
間違えたパターンは、**人間がやる場合も他の情報との付き合わせが必要**なもの。AIで100%を目指すのではなく、「人間ができる範囲の作業の補助」として考えるのがいい気がした。**類似度スコアが低い場合（AIでも迷う場合）は人間の判断を挟む**ことでガードレールを敷くのがいい気がする。
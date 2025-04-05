---
title: "[R & SQL] データサイエンス100本ノック＋α - 概要・導入"
slug: "start"
date: 2025-01-19T22:33:14+09:00
draft: false
weight: 15
# description: "「データサイエンス100本ノック＋α」の概要とコード実行環境の構築について。"
description: "R & SQL 演習問題集の概要とコード実行環境の構築について。"
summary: "R & SQL 演習問題集の概要とコード実行環境の構築について。"
# categories: ["実践ドリル(基本情報)"]
categories: ["R & SQL 演習問題 - 基本情報"]
# tags: 
# disableShare: false
# UseHugoToc: true
# ShowToc: true
# ShowToc: false
# TocOpen: true
# TocOpen: false
---

**参考記事:**

- {{% ref2 path="book-review" %}}
- {{% ref2 path="/dev/data-engineering/r-sql-auto-query" %}}

## はじめに

当サイトでは、データサイエンス100本ノックの R と SQL をベースにした演習問題を解説します。  
特に、R を用いたデータベース操作についての補足解説も加え、より実務に近い形で学べる内容としています。

- **シリーズ構成**
  - **{{< param k100.site.titleF >}}**  
    100本ノックの中から約 30 本の練習問題を選び、R と SQL を用いて解説します。
  {{% comment %}}
  - **{{< param k100.site.titleP >}}**  
  {{% /comment %}}
  - **オリジナル問題集**  
    オリジナルの演習問題を作成し、より多くの構文を用いたデータ処理を紹介する予定です。
<p>

- **解説方針**  
  各問題について、以下の 3 種類のコードを紹介し解説します。
  - **Rコード (データフレーム操作)**
  - **Rコード (データベース操作)**
  - **SQLクエリ**

サンプルコードは、可読性と効率を重視し、できるだけエレガントな記述を心がけています。  
R と SQL を用いたコーディングの練習問題としてご活用ください。

## 環境構築{#setup}

このシリーズでは、**RStudio や VSCode などの R 開発環境** を使って演習問題に取り組むことを想定しています。
その他の特別なツールは必要ありません。

環境構築のために、以下の 2 つのセットアップ方法を用意しています。  
目的に応じて、適した方法を選んでください。

1. **簡易セットアップ**  
   ローカルにスクリプトやデータを保存せず、すぐに試したい方におすすめです。
2. **標準セットアップ**  
   git コマンドに慣れている方におすすめです。

### 簡易セットアップ

次の R コードを .R ファイル (例: setup.R) にコピー＆ペーストして実行します。

{{< details summary="R コード (簡易セットアップ)" open=false >}}

```r {name="R"}
# pacman を使用してパッケージを管理 ------------
if (!require("pacman")) {
  install.packages("pacman")
  library("pacman")
}

# 必要なパッケージのロード ------------
# 存在しない場合は自動でインストールした後にロードする.
pacman::p_load(
  # tidyverse: 
  magrittr, fs, tibble, dplyr, tidyr, stringr, lubridate, forcats, 
  DBI, dbplyr, duckdb,      # for database
  rsample, recipes, themis, # tidymodels
  vroom, tictoc, jsonlite, withr, janitor, skimr, epikit, 
  install = TRUE,  # 存在しないパッケージをインストールする
  update = FALSE   # 既存のパッケージの更新は行わない
)

# CSVファイルをデータフレームとして読み込む ------------
my_vroom = function(file, col_types) {
  data_url = "https://raw.githubusercontent.com/katsu-ds-lab/ds-drills//main/work/data/"
  tictoc::tic(file)
  on.exit(tictoc::toc())
  on.exit(cat("\n"), add = TRUE)
  csv_url = data_url %>% paste0(file)
  print(csv_url); flush.console()
  d = csv_url %>% 
    vroom::vroom(col_types = col_types) %>% 
    janitor::clean_names() %>% 
    dplyr::glimpse() %T>% 
    { cat("\n") }
  return(d)
}

# customer.birth_day は Dateクラス
df_customer = "customer.csv" %>% my_vroom(col_types = "ccccDiccccc")
# receipt.sales_ymd は integer
df_receipt = "receipt.csv" %>% my_vroom(col_types = "iiciiccnn")
df_store = "store.csv" %>% my_vroom(col_types = "cccccccddd")
df_product = "product.csv" %>% my_vroom(col_types = "ccccnn")
df_category = "category.csv" %>% my_vroom(col_types = "cccccc")
df_geocode = "geocode.csv" %>% my_vroom(col_types = "cccccccnn")

# インメモリモードで一時的な DuckDB 環境を作成する ------------
con = duckdb::duckdb(dbdir = "") %>% duckdb::dbConnect()

# データフレームを DuckDB にテーブルとして書き込む ------------
con %>% DBI::dbWriteTable("customer", df_customer, overwrite = TRUE)
con %>% DBI::dbWriteTable("receipt", df_receipt, overwrite = TRUE)
con %>% DBI::dbWriteTable("store", df_store, overwrite = TRUE)
con %>% DBI::dbWriteTable("product", df_product, overwrite = TRUE)
con %>% DBI::dbWriteTable("category", df_category, overwrite = TRUE)
con %>% DBI::dbWriteTable("geocode", df_geocode, overwrite = TRUE)

# DuckDB のテーブルを dplyr で参照する ------------
db_customer = con %>% dplyr::tbl("customer")
db_receipt = con %>% dplyr::tbl("receipt")
db_store = con %>% dplyr::tbl("store")
db_product = con %>% dplyr::tbl("product")
db_category = con %>% dplyr::tbl("category")
db_geocode = con %>% dplyr::tbl("geocode")

# 関数の定義 ------------

# db_get_query() の定義
# SQLクエリを実行し, データフレーム(tibble)を返す
db_get_query = function(
    statement, con, convert_tibble = TRUE, params = NULL, ...
  ) {
  d = DBI::dbGetQuery(con, statement = statement, params = params, ...)
  if (convert_tibble) d %<>% tibble::as_tibble()
  return(d)
}

# sql_render_ext() の定義
# dbplyr::sql_render のラッパー
# デフォルトでは, バッククォート(`)を削除する
sql_render_ext = function(
    query, con = NULL, 
    cte = TRUE, 
    qualify_all_columns = TRUE, 
    use_star = TRUE, 
    sql_op = 
      dbplyr::sql_options(
        cte = cte, 
        use_star = use_star, 
        qualify_all_columns = qualify_all_columns
      ), 
    subquery = FALSE, lvl = 0, 
    pattern = "`", replacement = ""
  ) {
  ret = tryCatch(
    {
      query %>% 
        dbplyr::sql_render(
          con = con, sql_options = sql_op, subquery = subquery, lvl = lvl
        )
    }, 
    error = function(e) {
      # CTE を生成しない処理に cte = TRUE が指定された場合の措置
      sql_op = 
        dbplyr::sql_options(
          cte = FALSE, 
          use_star = use_star, 
          qualify_all_columns = qualify_all_columns
        )
      query %>% 
        dbplyr::sql_render(
          con = con, sql_options = sql_op, subquery = subquery, lvl = lvl
        )
    }
  )
  if (!is.null(pattern)) {
    ret %<>% gsub(pattern, replacement, .)
  }
  return(ret)
}
```

{{< /details >}}

実行後、DuckDB データベースがメモリ上に作成されます。  
R セッションを再開した場合は、再度このスクリプトを実行してください。

### 標準セットアップ

`~/projects` にファイルを展開する場合の例を以下に示します。

#### 1. GitHub リポジトリをクローン

次のコマンドを実行して、リポジトリをクローンします。

```sh {name="shell", hl_lines=2}
cd ~/projects
git clone https://github.com/katsu-ds-lab/ds-drills.git
```

実行後、`ds-drills` ディレクトリが作成され、以下のような構成になります。

```text {name="ds-drills" hl_lines=12}
work
  ├── database            # DuckDB データベース保存先
  ├── data                # CSVファイル, ER図
  │   ├── category.csv
  │   ├── customer.csv
  │   ├── geocode.csv
  │   ├── product.csv
  │   ├── receipt.csv
  │   ├── store.csv
  │   ├── 100knocks_ER.pdf
  │   └── LICENSE
  ├── init.R              # セットアップスクリプト
  ├── data_setup.R
  ├── env_setup.R
  └── functions.R
```

#### 2. init.R を実行

RStudio などで `init.R` を開いて実行します。  
実行後、`database` に DuckDB データベースファイル `supermarket.duckdb` が作成されます。

```text
work
  └── database
      └── supermarket.duckdb
```

6 個の CSVファイル (`data/*.csv`) を読み込み、それぞれテーブルとして `supermarket.duckdb` に保存しています。

DuckDB データベースをメモリ上に作成する場合は、`data_setup.R` の以下の箇所のコメントアウトを外して `init.R` を実行してください。

```r {name="data_setup.R"}
# 一時的にメモリ上に作成する場合は, 以下の行のコメントアウトを外してください.
is_fbmode = FALSE
```

R セッションを再開した場合は、再度 `init.R` を実行してください。  
2 回目からは、パッケージのインストールは不要なため、セットアップは約 5 秒で完了します。

#### トラブルシューティング

もしエラーが発生した場合は、作業ディレクトリの設定が原因となっている可能性が高いです。

現在の作業ディレクトリを確認するには、次のコマンドを実行してください。

```r {name="R"}
getwd()
```

`init.R` の以下の箇所で作業ディレクトリを設定しています。

```r
work_dir_path = init_path |> dirname()
```

作業ディレクトリとして `init.R` と同じ場所 (`ds-drills/work`) を設定する必要がありますので、
ここを以下のように変更して再実行してみてください。

```r
work_dir_path = "~/projects/ds-drills/work"
```

この変更を行うことで、エラーが解消される可能性があります。

### 利用可能なリソース

環境構築後、以下のリソースを利用できます。

#### 1. 主な R パッケージ

- `dplyr`
- `magrittr`
- `tidyr`
- `tibble`
- `stringr`
- `lubridate`
- `forcats`
- `DBI`
- `dbplyr`
- `duckdb`
- その他 `rsample` など

#### 2. R オブジェクト

##### データベース接続

- `con`

##### データフレーム

- `df_customer`
- `df_receipt`
- `df_product`
- `df_store`
- `df_category`
- `df_geocode`

##### データベースのテーブル参照

- `db_customer`
- `db_receipt`
- `db_product`
- `db_store`
- `db_category`
- `db_geocode`

##### 便利な関数

- `db_get_query()`
- `sql_render_ext()`

#### 3. DuckDB データベースファイル

- `work/database/supermarket.duckdb`

(＊ 簡易セットアップでは、DuckDB データベースはメモリ上に作成されます。)

#### 4. ER図 (データの構造)

- `work/data/100knocks_ER.pdf`

6 個のテーブルの関係を示す ER 図です。  

<div class="gallery-image gallery-base">
  <a href="ER.png" data-width="1692" data-height="928">
    <img src="ER.png" alt="ER図" style="display: block; margin: auto;">
  </a>
</div>
<span style="font-size: 0.94em;">
−{{< href-target-blank url="https://github.com/The-Japan-DataScientist-Society/100knocks-preprocess/blob/master/docker/doc/100knocks_ER.pdf" text="データサイエンス100本ノック（構造化データ加工編）- ER図" >}}より引用
</span>

{{% comment %}}
{{< figure src="ER.png" alt="ER図" title="" caption="" width="100%" link="" 
  rel="noopener" target="_blank" >}}
{{< href-target-blank url="https://github.com/The-Japan-DataScientist-Society/100knocks-preprocess/blob/master/docker/doc/100knocks_ER.pdf" text="データサイエンス100本ノック（構造化データ加工編） - ER図" >}}より引用
{{% /comment %}}

簡易セットアップを行なった場合は、上記リンクから `100knocks_ER.pdf` を保存しておくと便利です。

---

## データベース操作の補足事項

R によるデータベース操作や SQL クエリの自動生成について、以下の記事で紹介しています。

- {{% ref2 path="/dev/data-engineering/r-sql-auto-query" %}}
- {{% ref2 path="/dev/data-engineering/r-sql-translation" %}}

データベース操作に関連するその他の補足事項を `db_get_query()`、`sql_render_ext()` の使い方と共に以下で説明します。  
[環境構築]({{< ref "#setup" >}}) が完了していれば、本セクションのコードはコピー＆ペーストでそのまま実行できます。

### DuckDB を使用するメリット{#duckdb}

本シリーズでは、軽量かつ高速なデータベースエンジン **DuckDB** を使用します。  
DuckDB はデータサイエンス向けに設計された高性能データベースで、次のような特長があります。

- ローカル環境でのデータ分析や R との統合に適している
- PostgreSQL との互換性が高く、学習した内容を他のデータベースにも応用しやすい
- ファイルベースで手軽に扱え、インメモリモードを利用すればさらに高速な処理が可能

特に、教育目的や小規模データセットの分析に最適なため、本シリーズでは DuckDB を採用しています。

### SQL クエリを直接実行する

#### `dbGetQuery()`

`DBI::dbGetQuery()` を使用すると、指定した SQL クエリをデータベースで実行し、結果をデータフレーム (`data.frame` クラス) として取得できます。  
SQL を直接記述して実行する際に便利な関数です。

```r {name="R"}
query = 
  "SELECT sales_ymd, product_cd, amount FROM receipt"
d = DBI::dbGetQuery(con, query)
d %>% head(5)
```

```text
  sales_ymd product_cd amount
1  20181103 P070305012    158
2  20181118 P070701017     81
3  20170712 P060101005    170
4  20190205 P050301001     25
5  20180821 P060102007     90
```

`sql()` と組み合わせると、複雑なクエリの記述が容易になります。

```r {name="R"}
query = sql("
SELECT product_cd, SUM(amount) AS total_sales
FROM receipt
WHERE (sales_ymd >= 20180101)
GROUP BY product_cd
ORDER BY total_sales DESC
"
)

DBI::dbGetQuery(con, query)
```

```text
  product_cd total_sales
1 P071401001     1233100
2 P071401002      429000
3 P071401003      371800
4 P060303001      346320
5 P071401012      305800
...
```

引数 `n` を指定すると、取得するレコード数を制限できます。  

```r {name="R"}
DBI::dbGetQuery(con, query, n = 3)
```

```text
  product_cd total_sales
1 P071401001     1233100
2 P071401002      429000
3 P071401003      371800
```

また、`params` 引数を使用して、SQL のバインドパラメータを活用することができ、柔軟かつ安全にクエリを実行できます。

```r {name="R"} {hl_lines=4}
query = sql("
SELECT product_cd, SUM(amount) AS total_sales
FROM receipt
WHERE (sales_ymd >= ?)
GROUP BY product_cd
ORDER BY total_sales DESC
"
)

DBI::dbGetQuery(con, query, params = list(20190401))
```

```text
  product_cd total_sales
1 P071401001      376200
2 P071401002      158400
3 P071401003      123200
4 P060303001      117216
5 P071401013      110000
...
```

#### `db_get_query()`

独自関数 `db_get_query()` は `dbGetQuery()` のラッパー関数で、次の点を変更しています。

- デフォルトで結果を tibble として返す。
- クエリを第 1 引数として渡す。

使用例は以下の通りです : 

```r {name="R"}
query = sql("
SELECT product_cd, SUM(amount) AS total_sales
FROM receipt
WHERE (sales_ymd >= 20180101)
GROUP BY product_cd
ORDER BY total_sales DESC
"
)

query %>% db_get_query(con, n = 5)
```

実行結果は、次のように tibble として返されます。

```text
# A tibble: 5 × 2
  product_cd total_sales
  <chr>            <dbl>
1 P071401001     1233100
2 P071401002      429000
3 P071401003      371800
4 P060303001      346320
5 P071401012      305800
```

`convert_tibble = FALSE` を指定すると、`data.frame` クラスのデータフレームが返されますが、通常は tibble を使用することをお勧めします。

### 異なるデータベースの SQL を確認する

#### `sql_render()`

`sql_render()` を **データベースシミュレーター `simulate_*()`** と組み合わせて使用すると、異なるデータベース向けの SQL クエリを確認できます。

例えば、**PostgreSQL の SQL をシミュレーションする** 場合は `simulate_postgres()` を使います。

```r {name="R"} {hl_lines=["8-9"]}
db_result = db_customer %>% 
  mutate(
    m = birth_day %>% lubridate::month(), 
    .keep = "used"
  ) %>% 
  head(5)

db_result %>% sql_render(con = simulate_postgres())
```

```sql
<SQL> SELECT `birth_day`, EXTRACT(MONTH FROM `birth_day`) AS `m`
FROM customer
LIMIT 5
```

このように、実際のデータベース接続なしで SQL を確認できるので便利です。

MySQL/MariaDB、Snowflake、Oracle、SQL server での SQL のシミュレーションは、以下のようになります。

- MySQL/MariaDB

```r {name="R"}
db_result %>% sql_render(con = simulate_mysql())
```

```sql
<SQL> SELECT `birth_day`, EXTRACT(month FROM `birth_day`) AS `m`
FROM customer
LIMIT 5
```

- Snowflake

```r {name="R"}
db_result %>% sql_render(con = simulate_snowflake())
```

```sql
<SQL> SELECT `birth_day`, EXTRACT('month', `birth_day`) AS `m`
FROM customer
LIMIT 5
```

- Oracle

```r {name="R"}
db_result %>% sql_render(con = simulate_oracle())
```

```sql
<SQL> SELECT `birth_day`, EXTRACT(month FROM `birth_day`) AS `m`
FROM customer
FETCH FIRST 5 ROWS ONLY
```

- SQL server

```r {name="R"}
db_result %>% sql_render(con = simulate_mssql())
```

```sql
<SQL> SELECT TOP 5 `birth_day`, DATEPART(MONTH, `birth_day`) AS `m`
FROM customer
```

対応しているデータベース一覧は、以下の公式ページで確認できます。

{{< href-target-blank url="https://dbplyr.tidyverse.org/reference/index.html#built-in-database-backends" >}}

#### `sql_render_ext()`

独自関数 `sql_render_ext()` は `sql_render()` のラッパー関数で、以下の点を変更しています。

- 識別子のバッククォート (`) を制御。
- `cte` などの SQL オプション指定を簡略化。

通常、`sql_render()` の出力する SQL では、テーブル名やカラム名の識別子がバッククォート (`) で囲まれます。

```r {name="R"}
db_result = db_customer %>% 
  left_join(
    db_receipt %>% select(customer_id, amount), 
    by = "customer_id"
  ) %>% 
  group_by(customer_id) %>% 
  summarise(sum_amount = sum(amount, na.rm = TRUE)) %>% 
  arrange(customer_id)

db_result %>% sql_render(
    con = simulate_mysql(), 
    sql_options = sql_options(cte = TRUE)
  )
```

```sql
<SQL> WITH `q01` AS (
  SELECT `customer`.*, `amount`
  FROM customer
  LEFT JOIN receipt
    ON (`customer`.`customer_id` = `receipt`.`customer_id`)
)
SELECT `customer_id`, SUM(`amount`) AS `sum_amount`
FROM `q01`
GROUP BY `customer_id`
ORDER BY `customer_id`
```

識別子の囲みをなくすことで SQL の可読性が向上するため、`sql_render_ext()` はデフォルトでバッククォートを削除します。

```r {name="R"}
db_result %>% 
  sql_render_ext(con = simulate_mysql(), cte = TRUE)
```

```sql
<SQL> WITH q01 AS (
  SELECT customer.*, receipt.amount AS amount
  FROM customer
  LEFT JOIN receipt
    ON (customer.customer_id = receipt.customer_id)
)
SELECT customer_id, SUM(amount) AS sum_amount
FROM q01
GROUP BY customer_id
ORDER BY customer_id
```

また、`replacement` 引数に `"\""` を指定すると、識別子をダブルクォートで囲みます。

```r {name="R"}
db_result %>% sql_render_ext(
    con = simulate_mysql(), cte = TRUE, 
    replacement = "\""
  )
```

```sql
<SQL> WITH "q01" AS (
  SELECT "customer".*, "receipt"."amount" AS "amount"
  FROM customer
  LEFT JOIN receipt
    ON ("customer"."customer_id" = "receipt"."customer_id")
)
SELECT "customer_id", SUM("amount") AS "sum_amount"
FROM "q01"
GROUP BY "customer_id"
ORDER BY "customer_id"
```

また、`cte` などの SQL オプションを簡単に指定できます。以下の 2 つのコードは等価です。

```r {name="R"}
# sql_render
db_result %>% 
  sql_render(
    sql_options = 
      sql_options(cte = TRUE, use_star = FALSE, qualify_all_columns = FALSE)
  )
# sql_render_ext
db_result %>% 
  sql_render_ext(cte = TRUE, use_star = FALSE, qualify_all_columns = FALSE)
```

---

## 謝辞

当サイトは、{{< param k100.dss.name >}} 様が作成された素晴らしい教育コンテンツを、更なる発展を目指して作成しました。

また、当サイトで使用している以下のリソースは、{{< param k100.dss.sdc >}} 様によって提供された「{{< param k100.dss.title >}}」の GitHub リポジトリにて公開されているものを使用させていただいています。

- **データ**  
  {{< href-target-blank url="https://github.com/The-Japan-DataScientist-Society/100knocks-preprocess/tree/master/docker/work/data" text="GitHub リポジトリ - データ" >}}

- **ER 図**  
  {{< href-target-blank url="https://github.com/The-Japan-DataScientist-Society/100knocks-preprocess/blob/master/docker/doc/100knocks_ER.pdf" text="GitHub リポジトリ - ER 図" >}}

- **「{{< param k100.site.titleF >}}」の各演習問題**
  - {{< href-target-blank url="https://github.com/The-Japan-DataScientist-Society/100knocks-preprocess/tree/master/docker/doc" text="GitHub リポジトリ - 演習問題" >}}
  - 書籍📘『{{< param products.ds100kdp.name >}}』

より多くの方々がデータサイエンスのスキルをさらに高める一助となれば幸いです。

---

**演習問題 :**

- {{% ref2 path="drill-list" %}}

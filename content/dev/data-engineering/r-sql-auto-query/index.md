---
# Automatic translation from R to SQL query
title: "R でデータベース操作 – dplyr で SQL クエリを自動生成する"
slug: "r-sql-auto-query"
date: 2025-01-12T00:03:13+09:00
draft: false
# draft: true
weight: 35
description: "R からデータベースに接続し、データ加工から SQLクエリ自動生成までの流れについて解説"
# summary: ""
categories: ["R & SQL 演習問題 - 基本情報"]
tags: ["SQL自動生成"]
# image: rdb.webp
# disableShare: false
# ShowReadingTime: false
# ShowWordCount: false
---

データ分析をしていると、データベースとのやり取りが欠かせなくなりますね。
R でも、`dplyr` と `dbplyr` を使えば、データベースを手軽に操作できます。
今回は、これらのパッケージを活用して SQLクエリを生成し、データを取得する方法を簡単に紹介します。

**本記事で紹介するコードは、すべてコピー＆ペーストでそのまま実行できます。**

## dplyr と dbplyr の役割

`dplyr` は R のデータフレーム操作に便利なパッケージですが、`dplyr` のデータベース・バックエンドである `dbplyr` を組み合わせると、データベースに対して `dplyr` の構文を使ってテーブル操作のクエリを作成できます。
データベースのテーブルをデータフレームのように操作し、SQLクエリを自動で作成できるのが大きなメリットです。

## データベースへの接続とデータの準備

R では `DBI` パッケージを介してさまざまなデータベースと接続できますが、今回は手軽に扱える **DuckDB** を使用します。  
DuckDB のメリットについては、以下の記事を参考にしてください。

- {{% ref2 path="/ds-drills/data-tf/base/start#duckdb" text="DuckDB を使用するメリット" %}}

まず、デモ用の売上データ (`df_sales`) を作成し、DuckDB のインメモリデータベースに登録します。  
(未インストールのパッケージは `install.packages()` でインストールしてください。)

インメモリデータベースのため、セッション終了時にデータは自動で削除されます。

```r {name="R"}
library(DBI)
library(dplyr)
library(dbplyr)
library(duckdb)

# DuckDB に接続 (一時データベース)
con = DBI::dbConnect(duckdb::duckdb())

# サンプルのデータフレームを作成
df_sales = tribble(
  ~store, ~sales, ~profit,
  "S001", 15000, 3000,
  "S002", 18000, 3500,
  "S003", 12000, 2500,
  "S004", 20000, 4000,
  "S005", 16000, 3200
)

# store_sales テーブルとしてデータベースに登録
DBI::dbWriteTable(
  con, "store_sales", df_sales, overwrite = TRUE
)
```

## dplyr を用いたテーブル操作

次に、`tbl()` を用いてデータベースの **テーブルを dplyr で参照するオブジェクト** [^1] を作成します。
これにより、`filter()` などの `dplyr` のコードに対し `dbplyr` が別の方法で実行するようになり、コードが SQL に自動変換されます準備が整います。

[^1]: `tbl()` が返すオブジェクトについて、他には以下のような表現が適切かと思います。
    - dplyr のテーブル参照
    - データベースのテーブル参照
    - テーブル参照

```r {name="R"}
db_store_sales = tbl(con, "store_sales")
```

次に、`db_store_sales` に対し `dplyr` を用いてテーブルを操作します。

```r {name="R"}
db_result = 
  db_store_sales %>%
  filter(sales >= 15000) %>%
  arrange(desc(profit))
```

ここでは、フィルタリングとソートをしていますが、このようにテーブル参照に対してテーブル操作をするコードを「`dplyr`コード」と呼びます。

`db_result` は `dplyr`コードを SQLクエリの構造で保持します。この時点ではまだデータベース操作は行われていません。


`show_query()` を使うと、実際の SQLクエリを確認できます。

```r {name="R"}
show_query(db_result)
```

```sql
SELECT store_sales.*
FROM store_sales
WHERE (sales >= 15000.0)
ORDER BY profit DESC
```

`dbplyr` は、`tidyr` の一部の関数 (`pivot_longer()` など) とも互換があります。
`dbplyr` に対応する関数については、以下の公式ドキュメントに一覧があります。

- {{< href-target-blank url="https://dbplyr.tidyverse.org/reference/index.html#dplyr-verbs">}}

## SQLクエリの実行のタイミング

**遅延評価** (lazy evaluation) という仕組みにより、SQLクエリが実際に実行されるのは **データ取得を要求する処理** を行ったときです。例えば、`collect()` を呼び出すと、SQLクエリがデータベースで実行され、全ての結果がデータフレームとして返されます。

> [!WARNING]
> `collect()` の実行結果は R のメモリにロードされるため、大規模なデータが返される場合は要注意です。

```r {name="R"}
df_result = collect(db_result)
df_result %>% head(3) # 出力
```

```text
# A tibble: 3 × 3
  store sales profit
  <chr> <dbl>  <dbl>
1 S004  20000   4000
2 S002  18000   3500
3 S005  16000   3200
```

この後、`df_result` を用いて純粋な R コードで作業を継続できます。

> [!NOTE]
> インタラクティブな表示を含め、`print()` で表示したときにも SQLクエリが実行されます。
>
> ```r
> db_result %>% print(n = 3)
> ```
>
> ```text
> # Source:     SQL [?? x 3]
> # Database:   DuckDB v1.1.3-dev165 [root@Darwin 24.1.0:R 4.4.2/:memory:]
> # Ordered by: desc(profit)
>   store sales profit
>   <chr> <dbl>  <dbl>
> 1 S004  20000   4000
> 2 S002  18000   3500
> 3 S005  16000   3200
> ```
>
> 厳密には、次のような `LIMIT` 付きのクエリが発行され、最初の3行だけが取得されます。
>
> ```sql
> SELECT *
> FROM store_sales
> WHERE (sales >= 15000.0)
> ORDER BY profit DESC
> LIMIT 3
> ```

## まとめ

今回は、DuckDB をバックエンドとして、`dplyr` と `dbplyr` を用いたデータベース操作について解説しました。

- `dplyr` の操作をそのままデータベースに適用できる。
- `show_query()` で SQLクエリを確認できる。
- 遅延評価により、必要なときだけ SQLクエリが実行される。

この手法を活用すれば、大規模データの処理もスムーズになるので便利ですね！

ここまでに載せたコードをまとめると、以下のようになります。

```r {name="R", linenos=false, anchorLineNos=false, lineNoStart=1, hl_lines=["31-34"]}
# 必要なパッケージをロード
library(DBI)
library(dplyr)
library(dbplyr)
library(duckdb)

# DuckDB に接続 (一時データベース)
con = DBI::dbConnect(duckdb::duckdb())

# サンプルのデータフレームを作成
data = tribble(
  ~store, ~sales, ~profit,
  "S001", 15000, 3000,
  "S002", 18000, 3500,
  "S003", 12000, 2500,
  "S004", 20000, 4000,
  "S005", 16000, 3200
)

# store_sales テーブルとしてデータベースに登録
# (テーブルがある場合は上書き)
DBI::dbWriteTable(
  con, "store_sales", data, overwrite = TRUE
)

# store_sales テーブルを dplyr で参照
db_store_sales = tbl(con, "store_sales")

# フィルタリングとソート
# (テーブル操作をSQLクエリの構造で保持)
db_result = 
  db_store_sales %>%
  filter(sales >= 15000) %>%
  arrange(desc(profit))

# SQLクエリの生成・確認
show_query(db_result)
# => 
# <SQL>
# SELECT store_sales.*
# FROM store_sales
# WHERE (sales >= 15000.0)
# ORDER BY profit DESC

# SQLクエリの結果を R 側に取り込む
df_result = collect(db_result)

# 以下、データフレームでの処理
df_result %>% head(3)
# => 
# # A tibble: 3 × 3
#   store sales profit
#   <chr> <dbl>  <dbl>
# 1 S004  20000   4000
# 2 S002  18000   3500
# 3 S005  16000   3200
```

ハイライトした箇所のように、データベースのテーブルをデータフレームライクに操作できます！

次回は、`dplyr` コードと SQL クエリの対応について詳しく解説したいと思います。

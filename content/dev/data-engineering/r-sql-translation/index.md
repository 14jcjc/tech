---
# title: "R でデータベース操作 – dplyr コードはどのように SQLクエリに変換されるか"
# title: "R でデータベース操作 – dplyr は SQL にどう変換される？"
# title: "R で簡単データベース操作！ - dplyr は SQL にどう変換されるのか？ 実例で解説"
title: "dplyr × SQL 対応ガイド｜45の変換パターンを体系的に徹底解説"
slug: "r-sql-translation"
date: 2025-01-16T01:22:07+09:00
draft: false
# draft: true
weight: 40
# description: "R の dplyr コードと SQL の対応を、45 の具体例と共通のシンプルなデータセットで紹介。集計・結合・フィルタリングから、ウィンドウ関数・SQL構文埋め込みなどの応用的な操作まで幅広くカバーします。"
description: R の dplyr コードと SQL の対応を、共通データセットと 45 の具体例で解説。集計からウィンドウ関数、SQL 構文の埋め込みまで幅広く紹介します。
summary: "dplyr を用いたデータベース操作がどのように SQL に変換されるかを、具体例を交えて解説します。"
categories: ["R & SQL 演習問題 - 基本情報"]
tags: ["SQL自動生成"]
image: cover-r-sql-translation.png
# disableShare: false
# readingTime: true
# ShowWordCount: true
# toc: false
# ShowToc: true
TocOpen: false
tableOfContents:
  ordered: false
  startLevel: 2
  endLevel: 5
---

**前回の記事:**

- {{< ref2 path="r-sql-auto-query" >}}

## はじめに

前回は、`dplyr` を使ったテーブル操作と SQL クエリの自動生成について解説しました。

今回は、**`dplyr` コードが SQL にどのように変換されるのか**を、**45パターンの変換例**を交えて紹介します。  
サンプルコードを読みやすくするため、すべての変換例で共通のシンプルなデータセットを使用し、**集計・結合・フィルタリング**などの基本的な処理に加えて、**`dplyr` では直接サポートされていない操作**も取り上げています。

各例の `dplyr` コードと対応する SQL を見比べながら、**SQL の仕組みを理解し、R だけでデータベース操作を行う**際の参考にしていただければと思います。

**本記事で紹介するコードは、すべてコピー＆ペーストでそのまま実行できます。**

### データベースへの接続とデータの準備

データベースは前回に続き DuckDB を使用します。DuckDB のメリットについては、以下の記事を参考にしてください。

- {{% ref2 path="/ds-drills/data-tf/base/start#duckdb" text="DuckDB を使用するメリット" %}}

まず、デモ用の売上データ (`df_sales`) と店舗マスタ (`df_master`) を作成し、DuckDB のインメモリデータベースに登録します。  
(未インストールのパッケージは `install.packages()` でインストールしてください。)  

インメモリデータベースのため、セッション終了時にデータは自動で削除されます。

```r {name="R"}
library(DBI)
library(dplyr)
library(dbplyr)
library(duckdb)
library(tibble)

df_sales = tribble(
  ~store, ~month, ~sales, ~profit, 
  "S001",  4L,     150,   30, 
  "S001",  5L,     170,   34, 
  "S001",  6L,     140,   27, 
  "S001",  7L,     160,   32, 
  "S002",  4L,     NA,    28, 
  "S002",  5L,     160,   31, 
  "S002",  6L,     130,   27, 
  "S002",  7L,     150,   28
)

df_master = tribble(
  ~store, ~name,    ~pref, 
  "S001", "storeA", "Tokyo", 
  "S002", "storeB", "Osaka", 
  "S003", "storeC", "Kanagawa", 
  "S004", "storeD", "Fukuoka"
)

# DuckDB に接続 (一時データベース)
con = DBI::dbConnect(duckdb::duckdb())

# テーブルとしてデータベースに登録
DBI::dbWriteTable(
  con, "store_sales", df_sales, overwrite = TRUE
)
DBI::dbWriteTable(
  con, "store_master", df_master, overwrite = TRUE
)

# テーブルを dplyr で参照
db_sales = tbl(con, "store_sales")
db_master = tbl(con, "store_master")
```

以降では、`db_sales` と `db_master` を用いた R コードを紹介します。

### SQL 変換に関する補足

はじめに補足ですが、dbplyr による SQL 変換が常に最適な SQL を生成するわけではありません。
冗長的な SQLクエリに変換されるケースもあります。

例えば、以下の R コードは、

```r {name="R"}
db_sales %>% filter(!is.na(sales))
```

次の SQLクエリに変換されます。

```sql
SELECT store_sales.*
FROM store_sales
WHERE (NOT((sales IS NULL)))
```

この SQL は論理的には正しいですが、  
`WHERE (NOT((sales IS NULL)))` の部分は  
`WHERE sales IS NOT NULL` と書いた方が簡潔で可読性が高くなります。  
このような点も考慮しながら読み進めていただければと思います。

### dplyr 操作全体と操作内の式について

dbplyr による SQL 変換は、次の **2 つの側面** に分けられます。それぞれの観点から解説します。  

1. **dplyr 操作全体の SQL 変換**  
2. **dplyr 操作内の個々の式の SQL 変換**  

例えば、以下の R コードを考えます。  

```r {name="R"}
db_sales %>% filter(!is.na(sales))
```  

この場合、  

- `filter(, !is.na(sales))` → **「dplyr 操作全体」** に該当  
- `!is.na(sales)` → **「dplyr 操作内の式」** に該当  
- `!` 演算子、`is.na(sales)` → **「dplyr 操作内の個々の式」** に該当  

このように、dplyr の処理は **「操作全体」** と **「操作内の式」** に分けて考えることができます。  


### SQL 変換の基となる SELECT 文

dbplyr は純粋なテーブルに対して次のような `SELECT` 文を生成します。

```r {name="R"}
db_sales %>% show_query()
```

```sql
SELECT *
FROM store_sales
```

すべての dplyr 操作は、上記のような「全ての列を選択する `SELECT` 文」を基にして SQL を生成します。

---

以降では、次の3点を添えて SQL 変換について解説します。

- dplyr によるテーブル操作の R コード
- 自動生成される SQLクエリ
- 実行結果として得られるテーブルの内容

## dplyr 操作全体の SQL 変換

このセクションでは、dplyr 操作全体がどのように SQL に変換されるかについて、代表的な操作を例に挙げて解説します。

### 単一テーブルの操作

#### 列に影響を与える操作

##### `select()`、`rename()`、`relocate()`

`select()` は SQL の `SELECT` 句を修正します。

```r {name="R"}
db_sales %>% 
  select(store, sales) %>% 
  show_query()
```

```sql
SELECT store, sales
FROM store_sales
```

```text
  store sales
  <chr> <dbl>
1 S001    150
2 S001    170
3 S001    140
...
```

`rename()` についても同様に変換されます。

```r {name="R"}
db_sales %>% 
  rename(store_code = store) %>% 
  show_query()
```

```sql
SELECT store AS store_code, "month", sales, profit
FROM store_sales
```

```text
  store_code month sales profit
  <chr>      <int> <dbl>  <dbl>
1 S001           4   150     30
2 S001           5   170     34
3 S001           6   140     27
...
```

`relocate()` についても同様に変換されます。

```r {name="R"}
db_sales %>% 
  relocate(profit, sales, .after = store) %>% 
  show_query()
```

```sql
SELECT store, profit, sales, "month"
FROM store_sales
```

```text
  store profit sales month
  <chr>  <dbl> <dbl> <int>
1 S001      30   150     4
2 S001      34   170     5
3 S001      27   140     6
...
```

ここで、`"month"` がダブルクォートで括られているのは、これが DuckDB の予約語だからです。

##### `mutate()`

`mutate()` は SQL の `SELECT` 句を修正します。

```r {name="R"}
db_sales %>% 
  mutate(
    margin = 100 * profit / sales, 
    .keep = "unused"
  ) %>% 
  show_query()
```

```sql
SELECT store, "month", (100.0 * profit) / sales AS margin
FROM store_sales
```

```text
  store month margin
  <chr> <int>  <dbl>
1 S001      4   20  
2 S001      5   20  
3 S001      6   19.3
...
```

#### 行に影響を与える操作

##### `filter()`

`filter()` は SQL の `WHERE` 句を生成します。

```r {name="R"}
db_sales %>% 
  filter(month == 4L & profit >= 30) %>% 
  show_query()
```

```sql
SELECT store_sales.*
FROM store_sales
WHERE ("month" = 4) AND (profit >= 30.0)
```

```text
  store month sales profit
  <chr> <int> <dbl>  <dbl>
1 S001      4   150     30
```

##### `arrange()`

`arrange()` は SQL の `ORDER BY` 句を生成します。

```r {name="R"}
db_sales %>% 
  arrange(month, desc(profit)) %>% 
  show_query()
```

```sql
SELECT store_sales.*
FROM store_sales
ORDER BY "month", profit DESC
```

```text
  store month sales profit
  <chr> <int> <dbl>  <dbl>
1 S001      4   150     30
2 S002      4    NA     28
3 S001      5   170     34
4 S002      5   160     31
...
```

##### `head()`

`head()` は SQL の `LIMIT` 句を生成します。  
ただし、一部のデータベース (SQL Server など) では、`TOP` 句や `FETCH FIRST` 句が生成される場合があります。

```r {name="R"}
db_sales %>% 
  head(3) %>% 
  show_query()
```

```sql
SELECT store_sales.*
FROM store_sales
LIMIT 3
```

```text
  store month sales profit
  <chr> <int> <dbl>  <dbl>
1 S001      4   150     30
2 S001      5   170     34
3 S001      6   140     27
```

##### `distinct()`

`distinct()` は SQL の `DISTINCT` 修飾子を生成します。

```r {name="R"}
db_sales %>% 
  distinct() %>% 
  show_query()
```

```sql
SELECT DISTINCT store_sales.*
FROM store_sales
```

```text
  store month sales profit
  <chr> <int> <dbl>  <dbl>
1 S002      6   130     27
2 S001      4   150     30
3 S001      7   160     32
...
```

列を指定した場合の例:

```r {name="R"}
db_sales %>% 
  distinct(store) %>% 
  show_query()
```

```sql
SELECT DISTINCT store
FROM store_sales
```

```text
  store
  <chr>
1 S002 
2 S001 
```

#### グループ化と集約の操作

##### `summarise()`

`summarise()` は `mean()` などの集約関数と合わせて `SELECT` 句を修正します。

```r {name="R"}
db_sales %>% 
  summarise(avg_profit = mean(profit)) %>% 
  show_query()
```

```sql
SELECT AVG(profit) AS avg_profit
FROM store_sales
```

```text
  avg_profit
       <dbl>
1       29.6
```

##### `group_by()` \+ `summarise()`

また、`summarise()` は `group_by()` と合わせて `GROUP BY` 句を生成します。

```r {name="R"}
db_sales %>% 
  group_by(store) %>% 
  summarise(avg_profit = mean(profit)) %>% 
  show_query()
```

```sql
SELECT store, AVG(profit) AS avg_profit
FROM store_sales
GROUP BY store
```

```text
  store avg_profit
  <chr>      <dbl>
1 S002        28.5
2 S001        30.8
```

##### `group_by()` \+ `summarise()` \+ `filter()`

さらに、集約後の `filter()` と合わせて `HAVING` 句を生成します。

```r {name="R"}
db_sales %>% 
  group_by(store) %>% 
  summarise(avg_profit = mean(profit)) %>% 
  filter(avg_profit > 30) %>% 
  show_query()
```

```sql
SELECT store, AVG(profit) AS avg_profit
FROM store_sales
GROUP BY store
HAVING (AVG(profit) > 30.0)
```

```text
  store avg_profit
  <chr>      <dbl>
1 S001        30.8
```

### 2つのテーブルによる操作

#### テーブルの結合

##### `inner_join()`、`left_join()`、`right_join()`

`inner_join()` は `INNER JOIN` 句を生成します。

```r {name="R"}
db_sales %>% 
  inner_join(db_master, by = "store") %>% 
  show_query()
```

```sql
SELECT store_sales.*, "name", pref
FROM store_sales
INNER JOIN store_master
  ON (store_sales.store = store_master.store)
```

```text
  store month sales profit name   pref 
  <chr> <int> <dbl>  <dbl> <chr>  <chr>
1 S001      4   150     30 storeA Tokyo
2 S001      5   170     34 storeA Tokyo
3 S001      6   140     27 storeA Tokyo
4 S001      7   160     32 storeA Tokyo
5 S002      4    NA     28 storeB Osaka
...
```

`left_join()`、`right_join()` についても同様です。

##### `full_join()`

`full_join()` は `FULL JOIN` 句を生成します。

```r {name="R"}
db_sales %>% 
  full_join(db_master, by = "store") %>% 
  show_query()
```

```sql
SELECT
  COALESCE(store_sales.store, store_master.store) AS store,
  "month",
  sales,
  profit,
  "name",
  pref
FROM store_sales
FULL JOIN store_master
  ON (store_sales.store = store_master.store)
```

```text
   store month sales profit name   pref    
   <chr> <int> <dbl>  <dbl> <chr>  <chr>   
 1 S001      4   150     30 storeA Tokyo   
 2 S001      5   170     34 storeA Tokyo   
 3 S001      6   140     27 storeA Tokyo   
 4 S001      7   160     32 storeA Tokyo   
 5 S002      4    NA     28 storeB Osaka   
 6 S002      5   160     31 storeB Osaka   
 7 S002      6   130     27 storeB Osaka   
 8 S002      7   150     28 storeB Osaka   
 9 S004     NA    NA     NA storeD Fukuoka 
10 S003     NA    NA     NA storeC Kanagawa
```

##### `cross_join()`

`cross_join()` は `CROSS JOIN` 句を生成します。

```r {name="R"}
db_master %>% 
  select(store) %>% 
  cross_join(db_sales %>% select(month)) %>% 
  show_query()
```

```sql
SELECT store_master.store AS store, "month"
FROM store_master
CROSS JOIN store_sales
```

```text
   store month
   <chr> <int>
 1 S001      4
 2 S001      5
 3 S001      6
 4 S001      7
 ...
```

##### `semi_join()` (準結合)

`semi_join()` は `WHERE EXISTS` を生成します。

```r {name="R"}
db_master %>% 
  semi_join(db_sales, by = "store") %>% 
  show_query()
```

```sql
SELECT store_master.*
FROM store_master
WHERE EXISTS (
  SELECT 1 FROM store_sales
  WHERE (store_master.store = store_sales.store)
)
```

```text
  store name   pref 
  <chr> <chr>  <chr>
1 S001  storeA Tokyo
2 S002  storeB Osaka
```

##### `anti_join()` (アンチ結合)

`anti_join()` は `WHERE NOT EXISTS` を生成します。

```r {name="R"}
db_master %>% 
  anti_join(db_sales, by = "store") %>% 
  show_query()
```

```sql
SELECT store_master.*
FROM store_master
WHERE NOT EXISTS (
  SELECT 1 FROM store_sales
  WHERE (store_master.store = store_sales.store)
)
```

```text
  store name   pref    
  <chr> <chr>  <chr>   
1 S003  storeC Kanagawa
2 S004  storeD Fukuoka 
```

#### テーブルの集合演算

##### `intersect()`

`intersect()` は `INTERSECT` 演算子を生成します。

```r {name="R"}
db_sales %>% 
  select(store) %>% 
  intersect(db_master %>% select(store)) %>% 
  show_query()
```

```sql
(
  SELECT store
  FROM store_sales
)
INTERSECT
(
  SELECT store
  FROM store_master
)
```

```text
  store
  <chr>
1 S002 
2 S001 
```

##### `union()`

`union()` は `UNION` 演算子を生成します。

```r {name="R"}
db_sales %>% 
  select(store) %>% 
  union(db_master %>% select(store)) %>% 
  show_query()
```

```sql
SELECT store
FROM store_sales

UNION

SELECT store
FROM store_master
```

```text
  store
  <chr>
1 S001 
2 S003 
3 S002 
4 S004 
```

##### `union_all()`

`union_all()` は `UNION ALL` 演算子を生成します。

```r {name="R"}
db_sales %>% 
  select(store) %>% 
  union_all(db_master %>% select(store)) %>% 
  show_query()
```

```sql
SELECT store
FROM store_sales

UNION ALL

SELECT store
FROM store_master
```

```text
   store
   <chr>
 1 S001 
 2 S001 
 3 S001 
 4 S001 
 5 S002 
 ...
```

##### `setdiff()`

`setdiff()` は `EXCEPT` 演算子を生成します。

```r {name="R"}
db_master %>% 
  select(store) %>% 
  setdiff(db_sales %>% select(store)) %>% 
  show_query()
```

```sql
(
  SELECT store
  FROM store_master
)
EXCEPT
(
  SELECT store
  FROM store_sales
)
```

```text
  store
  <chr>
1 S004 
2 S003 
```

### その他のテーブル操作

これまでに紹介した代表的なテーブル操作の関数に加えて、`count()`, `slice_min()`, `slice_max()`, `replace_na()`, `pivot_longer()` などの関数もあります。  
これらの関数は、前述の SQL の句や演算子、SQL 関数を組み合わせて変換されます。

以下に、`count()` と `pivot_longer()` を使用した場合の変換例を示します。

#### `count()`

`count()` は次のように、SQL の集約関数 `COUNT()` を用いて `SELECT` 句を修正し、`GROUP BY` 句を生成します。

```r {name="R"}
db_sales %>% 
  count(store, name = "n_month") %>% 
  show_query()
```

```sql
SELECT store, COUNT(*) AS n_month
FROM store_sales
GROUP BY store
```

```text
  store n_month
  <chr>   <dbl>
1 S001        4
2 S002        4
```

#### `pivot_longer()`

`pivot_longer()` は次のように `UNION ALL` 演算子を用いたクエリを生成します。

```r {name="R"}
db_sales %>% 
  tidyr::pivot_longer(
    -c(store, month), names_to = "name", values_to = "amount"
  ) %>% 
  show_query()
```

```sql
SELECT store, "month", 'sales' AS "name", sales AS amount
FROM store_sales

UNION ALL

SELECT store, "month", 'profit' AS "name", profit AS amount
FROM store_sales
```

```text
   store month name   amount
   <chr> <int> <chr>   <dbl>
 1 S001      4 sales     150
 2 S001      5 sales     170
 3 S001      6 sales     140
 4 S001      7 sales     160
 5 S002      4 sales      NA
 ...
```

## dplyr 操作内の式の SQL 変換

このセクションでは、dplyr 操作内で使用する個々の式がどのように SQL に変換されるかに焦点を当てて解説します。  
主に以下の2つの観点を元に、代表的な演算子や関数について例を挙げて解説します。

- dplyr が認識できる式
- dplyr が認識できない式

### dplyr が認識できる式

#### 基本的な演算子

##### 算術演算子 (`+`、`-`、`*`、`/`、`^`)

dplyr の算術演算子は、SQL に変換される際にそれぞれ対応する式や関数にマッピングされます。

```r {name="R"}
db_sales %>% 
  mutate(
    v1 = sales + profit, 
    v2 = 100 * (sales - profit) / sales, 
    v3 = profit ^ 2L, 
    .keep = "none"
  ) %>% 
  show_query()
```

```sql
SELECT
  sales + profit AS v1,
  (100.0 * (sales - profit)) / sales AS v2,
  POW(profit, 2) AS v3
FROM store_sales
```

```text
     v1    v2    v3
  <dbl> <dbl> <dbl>
1   180  80     900
2   204  80    1156
3   167  80.7   729
...
```

##### 比較演算子、論理演算子

dplyr の比較演算子 (`==`、`<`、`%in%` など)、論理演算子 (`&`、`|`、`!` など) は、SQL に変換される際にそれぞれ対応する式にマッピングされます。

```r {name="R"}
db_sales %>% 
  mutate(
    v1 = (sales == 150), 
    v2 = (!(sales > 150)), 
    v3 = (sales != 150 & profit >= 30), 
    v4 = (sales < 150 | profit <= 30), 
    v5 = (store %in% c("S001", "S003")), 
    .keep = "used"
  ) %>% 
  show_query()
```

```sql
SELECT
  store,
  sales,
  profit,
  (sales = 150.0) AS v1,
  (NOT((sales > 150.0))) AS v2,
  (sales != 150.0 AND profit >= 30.0) AS v3,
  (sales < 150.0 OR profit <= 30.0) AS v4,
  (store IN ('S001', 'S003')) AS v5
FROM store_sales
```

```text
  store sales profit v1    v2    v3    v4    v5   
  <chr> <dbl>  <dbl> <lgl> <lgl> <lgl> <lgl> <lgl>
1 S001    150     30 TRUE  TRUE  FALSE TRUE  TRUE 
2 S001    170     34 FALSE FALSE TRUE  FALSE TRUE 
3 S001    140     27 FALSE TRUE  FALSE TRUE  TRUE 
4 S001    160     32 FALSE FALSE TRUE  FALSE TRUE 
5 S002     NA     28 NA    NA    FALSE TRUE  FALSE
...
```

#### 基本的な関数

##### 数学関数、数値の丸め関数

一般的な数学関数、数値の丸め関数は、SQL に変換される際にそれぞれ対応する関数にマッピングされます。

```r {name="R"}
db_sales %>% 
  mutate(
    v1 = log(profit), 
    v2 = sqrt(profit), 
    v3 = sin(profit), 
    v4 = floor(sales / profit), 
    .keep = "none"
  ) %>% 
  show_query()
```

```sql
SELECT
  LN(profit) AS v1,
  SQRT(profit) AS v2,
  SIN(profit) AS v3,
  FLOOR(sales / profit) AS v4
FROM store_sales
```

```text
     v1    v2     v3    v4
  <dbl> <dbl>  <dbl> <dbl>
1  3.40  5.48 -0.988     5
2  3.53  5.83  0.529     5
3  3.30  5.20  0.956     5
4  3.47  5.66  0.551     5
5  3.33  5.29  0.271    NA
...
```

##### キャスト関数

キャスト関数は、SQL に変換される際にそれぞれ対応する関数にマッピングされます。  
DuckDB では、次の例のように `CAST()` を用いた式に変換されます。

```r {name="R"}
db_sales %>% 
  mutate(
    v1 = as.integer(profit), 
    v2 = as.numeric(month), 
    v3 = as.double(month), 
    v4 = as.character(month), 
    v5 = as.Date("2025-04-01"), 
    .keep = "used"
  ) %>% 
  show_query()
```

```sql
SELECT
  "month",
  profit,
  CAST(profit AS INTEGER) AS v1,
  CAST("month" AS NUMERIC) AS v2,
  CAST("month" AS NUMERIC) AS v3,
  CAST("month" AS TEXT) AS v4,
  CAST('2025-04-01' AS DATE) AS v5
FROM store_sales
```

```text
  month profit    v1    v2    v3 v4    v5        
  <int>  <dbl> <int> <dbl> <dbl> <chr> <date>    
1     4     30    30     4     4 4     2025-04-01
2     5     34    34     5     5 5     2025-04-01
3     6     27    27     6     6 6     2025-04-01
...
```

##### 文字列関数

基本的な文字列関数は次の例のように SQL に変換できます。  
`stringr` パッケージの一部の関数にも対応しています。

```r {name="R"}
db_master %>% 
  mutate(
    len = nchar(pref), 
    upp = toupper(pref), 
    sub = substr(name, 6, 6), 
    p = paste(name, pref, sep = "-"), 
    .keep = "used"
  ) %>% 
  show_query()
```

```sql
SELECT
  "name",
  pref,
  LENGTH(pref) AS len,
  UPPER(pref) AS upp,
  SUBSTR("name", 6, 1) AS sub,
  CONCAT_WS('-', "name", pref) AS p
FROM store_master
```

```text
  name   pref       len upp      sub   p              
  <chr>  <chr>    <dbl> <chr>    <chr> <chr>          
1 storeA Tokyo        5 TOKYO    A     storeA-Tokyo   
2 storeB Osaka        5 OSAKA    B     storeB-Osaka   
3 storeC Kanagawa     8 KANAGAWA C     storeC-Kanagawa
4 storeD Fukuoka      7 FUKUOKA  D     storeD-Fukuoka 
```

##### 日付関数

基本的な日付関数は次の例のように SQL に変換できます。  
`lubridate` パッケージの一部の関数にも対応しています。

```r {name="R"}
db_master %>% 
  mutate(
    ymd = lubridate::as_date("2025-04-01"), 
    .keep = "none"
  ) %>% 
  head(1) %>% 
  mutate(
    month = lubridate::month(ymd), 
    add = ymd + lubridate::days(7L), 
    .keep = "used"
  ) %>% 
  show_query(cte = TRUE)
```

```sql
WITH q01 AS (
  SELECT CAST('2025-04-01' AS DATE) AS ymd
  FROM store_master
  LIMIT 1
)
SELECT
  q01.*,
  EXTRACT(MONTH FROM ymd) AS "month",
  ymd + TO_DAYS(CAST(7 AS INTEGER)) AS "add"
FROM q01
```

```text
  ymd        month add                
  <date>     <dbl> <dttm>             
1 2025-04-01     4 2025-04-08 00:00:00
```

##### パターンマッチング

`stringr::str_detect()`、`grepl()` は `filter()` との組み合わせで、`WHERE` 句によるパターンマッチングを実行する式を生成します。

```r {name="R"}
db_master %>% 
  filter(
    stringr::str_detect(pref, "ka$")
  ) %>% 
  show_query()
```

```sql
SELECT store_master.*
FROM store_master
WHERE (REGEXP_MATCHES(pref, 'ka$'))
```

```text
  store name   pref   
  <chr> <chr>  <chr>  
1 S002  storeB Osaka  
2 S004  storeD Fukuoka
```

##### `is.na()`

`is.na()` は `IS NULL` 句を生成します。

```r {name="R"}
db_sales %>% 
  filter(is.na(sales)) %>% 
  show_query()
```

```sql
SELECT store_sales.*
FROM store_sales
WHERE ((sales IS NULL))
```

```text
  store month sales profit
  <chr> <int> <dbl>  <dbl>
1 S002      4    NA     28
```

##### `if_else()`

`if_else()` は `CASE` 式を生成します。

```r {name="R"}
db_sales %>% 
  mutate(
    profit_size = 
      if_else(sales > 150, "big", "small", "none"), 
    .keep = "used"
  ) %>% 
  show_query()
```

```sql {linenos=true, hl_lines=6}
SELECT
  sales,
  CASE 
    WHEN (sales > 150.0) THEN 'big' 
    WHEN NOT (sales > 150.0) THEN 'small' 
    WHEN ((sales > 150.0) IS NULL) THEN 'none' 
  END AS profit_size
FROM store_sales
```

```text
  sales profit_size
  <dbl> <chr>      
1   150 small      
2   170 big        
3   140 small      
4   160 big        
5    NA none       
...
```

SQL の 6行目は、`ELSE 'none'` とリライトするとより簡潔になります。

##### 集約関数 (`summarise()`内)

`mean()` などの集約関数は、`summarise()` 内で使用すると、SQL に変換される際にそれぞれ対応する集約関数にマッピングされます。

```r {name="R"}
db_sales %>% 
  summarise(
    n = n(), 
    n_store = n_distinct(store), 
    avg = mean(sales), 
    per50 = median(sales)
  ) %>% 
  show_query()
```

```sql
SELECT
  COUNT(*) AS n,
  COUNT(DISTINCT row(store)) AS n_store,
  AVG(sales) AS avg,
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY sales) AS per50
FROM store_sales
```

```text
      n n_store   avg per50
  <dbl>   <dbl> <dbl> <dbl>
1     8       2 151.4   150
```

#### ウィンドウ関数

次に、ウィンドウ関数の SQL 変換について解説します。  
R のウィンドウ関数は以下の4種類に分類されます。

- **集約関数:**  
  `n()`、`sum()`、 `mean()`、`max()`、`sd()` など
- **シフト関数:**  
  `lag()`、`lead()`
- **ランキング関数:**  
  `min_rank()`、`dense_rank()`、`ntile()` など
- **累積関数:**  
  `cumsum()`、`cummean()`、`cummax()`、`cumall()` など

SQL では、`AVG(sales) OVER ()` のように、ウィンドウ (範囲) を指定した集約関数などの式をウィンドウ関数と呼びます。

ウィンドウ関数は、次の形式で記述されます。

- `[式] OVER ([パーティション句] [順序句] [フレーム句])`

ここで、`[式]` は関数と変数名の組み合わせです。(例: `AVG(sales)`)  
表現方法が豊富なため、変換後のクエリはこれまでのものよりも複雑になる傾向があります。

##### 集約関数 (`mutate()`内)

`mean()` などの集約関数は `mutate()` 内で使用するとウィンドウ関数として適用され、  
`AVG(sales) OVER ()` のようなウィンドウ関数を生成します。

```r {name="R"}
db_sales %>% 
  mutate(
    n = n(), 
    avg = mean(sales), 
    max = max(sales)
  ) %>% 
  show_query()
```

```sql
SELECT
  store_sales.*,
  COUNT(*) OVER () AS n,
  AVG(sales) OVER () AS avg,
  MAX(sales) OVER () AS max
FROM store_sales
```

```text
  store month sales profit     n   avg   max
  <chr> <int> <dbl>  <dbl> <dbl> <dbl> <dbl>
1 S001      4   150     30     8 151.4   170
2 S001      5   170     34     8 151.4   170
3 S001      6   140     27     8 151.4   170
...
```

`group_by()` を併用すると以下のように変換されます。

```r {name="R"}
db_sales %>% 
  group_by(month) %>% 
  mutate(
    n = n(), 
    avg = mean(sales), 
    max = max(sales)
  ) %>% 
  show_query()
```

```sql
SELECT
  store_sales.*,
  COUNT(*) OVER (PARTITION BY "month") AS n,
  AVG(sales) OVER (PARTITION BY "month") AS avg,
  MAX(sales) OVER (PARTITION BY "month") AS max
FROM store_sales
```

```text
  store month sales profit     n   avg   max
  <chr> <int> <dbl>  <dbl> <dbl> <dbl> <dbl>
1 S001      4   150     30     2   150   150
2 S002      4    NA     28     2   150   150
3 S001      6   140     27     2   135   140
4 S002      6   130     27     2   135   140
5 S001      7   160     32     2   155   160
6 S002      7   150     28     2   155   160
7 S001      5   170     34     2   165   170
8 S002      5   160     31     2   165   170
```

ウィンドウ関数では、`group_by(month)` により `GROUP BY` 句ではなく `OVER ()` 内に  
`PARTITION BY "month"` が生成されます。

また、`window_order()` と `window_frame()` を併用すると、以下のように変換されます。

```r {name="R"}
db_sales %>% 
  group_by(store) %>% 
  window_order(month) %>% 
  window_frame(-1, 1) %>% 
  mutate(
    avg_win = mean(sales)
  ) %>% 
  show_query()
```

```sql
SELECT
  store_sales.*,
  AVG(sales) OVER (
    PARTITION BY store 
    ORDER BY "month" 
    ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
  ) AS avg_win
FROM store_sales
```

```text
# Ordered by: month
  store month sales profit avg_win
  <chr> <int> <dbl>  <dbl>   <dbl>
1 S001      4   150     30   160  
2 S001      5   170     34   153.3
3 S001      6   140     27   156.7
4 S001      7   160     32   150  
5 S002      4    NA     28   160  
6 S002      5   160     31   145  
7 S002      6   130     27   146.7
8 S002      7   150     28   140  
```

`AVG(sales) OVER ()` の内部については、`window_order(month)` により  
`ORDER BY "month"` が生成され、`window_frame(-1, 1)` により  
`ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING` という `ROWS` 句が生成されています。

##### シフト関数 (`lag()`、`lead()`)

`lag()`、`lead()` を用いた場合の SQL 変換については以下のようになります。

```r {name="R"}
db_sales %>% 
  group_by(store) %>% 
  window_order(month) %>% 
  mutate(
    lag_p = lag(profit, 1L), 
    lead_p = lead(profit, 1L)
  ) %>% 
  show_query()
```

```sql
SELECT
  store_sales.*,
  LAG(profit, 1, NULL) OVER (PARTITION BY store ORDER BY "month") AS lag_p,
  LEAD(profit, 1, NULL) OVER (PARTITION BY store ORDER BY "month") AS lead_p
FROM store_sales
```

```text
  store month sales profit lag_p lead_p
  <chr> <int> <dbl>  <dbl> <dbl>  <dbl>
1 S002      4    NA     28    NA     31
2 S002      5   160     31    28     27
3 S002      6   130     27    31     28
4 S002      7   150     28    27     NA
5 S001      4   150     30    NA     34
6 S001      5   170     34    30     27
7 S001      6   140     27    34     32
8 S001      7   160     32    27     NA
```

`OVER ()` の内部については、`group_by(store)` と `window_order(month)` により  
`PARTITION BY store ORDER BY "month"` が生成されています。  
`arrange()` を用いても `OVER` 句に `ORDER BY` は生成されない点にご注意ください。

##### ランキング関数

ランキング関数の SQL 変換についてですが、`min_rank()` を用いた場合の例を以下に示します。

```r {name="R"}
db_sales %>% 
  group_by(store) %>% 
  mutate(
    rank = min_rank(desc(sales)), 
    .keep = "used"
  ) %>% 
  show_query()
```

```sql
SELECT
  store, 
  sales, 
  CASE
    WHEN (NOT((sales IS NULL))) THEN RANK() OVER (
      PARTITION BY 
        store, 
        (CASE WHEN ((sales IS NULL)) THEN 1 ELSE 0 END) 
      ORDER BY sales DESC
    )
  END AS rank
FROM store_sales
```

```text
  store sales  rank
  <chr> <dbl> <dbl>
1 S001    170     1
2 S001    160     2
3 S001    150     3
4 S001    140     4
5 S002     NA    NA
6 S002    160     1
7 S002    150     2
8 S002    130     3
```

2番目のパーティションキー `CASE WHEN (sales IS NULL) THEN 1 ELSE 0 END` は、`sales` が NULL の行をグループ 1、それ以外をグループ 0 に分類するためのものです。  
これにより、`sales IS NULL` の行が別グループとして扱われ、ランク付けの対象から確実に切り離されます。  

`WHEN (NOT (sales IS NULL)) THEN RANK()` という条件があるため、一見、このパーティションキーは不要にも思えます。  
ですが、指定しない場合、NULL の行も含めてランク付けが行われ、データベースのソート設定によっては NULL が通常の値より前に配置されることがあります。その結果、意図しないランク付けが発生する可能性があるため、このキーを用いて NULL のデータが明示的に分離されています。

##### 累積関数

累積関数の SQL 変換についてですが、`cumsum()` を用いた場合の例を以下に示します。

```r {name="R"}
db_sales %>% 
  group_by(store) %>% 
  window_order(month) %>% 
  mutate(
    cum = cumsum(profit)
  ) %>% 
  show_query()
```

```sql
SELECT
  store_sales.*,
  SUM(profit) OVER (
    PARTITION BY store ORDER BY "month" 
    ROWS UNBOUNDED PRECEDING
  ) AS cum
FROM store_sales
```

```text
  store month sales profit   cum
  <chr> <int> <dbl>  <dbl> <dbl>
1 S001      4   150     30    30
2 S001      5   170     34    64
3 S001      6   140     27    91
4 S001      7   160     32   123
5 S002      4    NA     28    28
6 S002      5   160     31    59
7 S002      6   130     27    86
8 S002      7   150     28   114
```

`SUM(profit) OVER ()` の内部については、`group_by(store)` と `window_order(month)` により  
`PARTITION BY store ORDER BY "month"` が生成され、  
最初の行から現在の行までを累積の対象とするために  
`ROWS UNBOUNDED PRECEDING` という `ROWS` 句が生成されています。

### dbplyr が認識できない式{#dbplyr-unknown}

dbplyr が認識できない式については、そのまま SQL に残されます。  
これにより、dplyr でカバーされていないデータベース関数を直接記述することができます。

#### プレフィックス関数

dbplyr が SQL への変換方法を知らない関数は、そのまま SQL に残されます。

SQL 関数は一般的に**大文字・小文字を区別しない**ため、R コード内で SQL 関数を記述する際は**大文字**を使うことを推奨します。
これにより、通常の R の関数と区別しやすくなります。

以下は、SQL 関数 `CEIL()` (小数を切り上げる) と `EVEN()` (数値を偶数に丸める) を使用した例です。

```r {name="R"}
db_sales %>% 
  mutate(
    v1 = CEIL(profit / sales), 
    v2 = EVEN(month)
  ) %>% 
  show_query()
```

```sql
SELECT 
  store_sales.*, 
  CEIL(profit / sales) AS v1, 
  EVEN("month") AS v2
FROM store_sales
```

```text
  store month sales profit    v1    v2
  <chr> <int> <dbl>  <dbl> <dbl> <dbl>
1 S001      4   150     30     1     4
2 S001      5   170     34     1     6
3 S001      6   140     27     1     6
4 S001      7   160     32     1     8
5 S002      4    NA     28    NA     4
...
```

このように dbplyr を活用すれば、dplyr の記法を維持しながらデータベース特有の関数を使うことができます。

#### インフィックス関数

`x %in% y` のように、関数名が引数の間に挟まる形式の関数 (インフィックス関数) も変換されます。  
これにより、次のように `LIKE` などの式を使用することができます。

```r {name="R"}
db_master %>% 
  filter(
    pref %LIKE% "%ka"
  ) %>% 
  show_query()
```

```sql
SELECT store_master.*
FROM store_master
WHERE (pref LIKE '%ka')
```

```text
  store name   pref   
  <chr> <chr>  <chr>  
1 S002  storeB Osaka  
2 S004  storeD Fukuoka
```

#### 特殊な式 (SQL の構文を埋め込む)

SQL 関数は R よりも構文のバリエーションが多いため、R から直接変換できない式もあります。  
これらをクエリに直接埋め込むには、`sql()` 内でリテラル SQL を使用します。

以下は、`IFNULL()` (NULL 値を変換)、`REVERSE()` (文字列を反転)、`QUANTILE_CONT()` (分位数) を用いた例です。

```r {name="R"}
db_sales %>% 
  mutate(
    sales2 = sql("IFNULL(sales, 0.0)"), 
    store_rev = sql("REVERSE(store)"), 
    per25 = sql("QUANTILE_CONT(profit, 0.25) OVER ()")
  ) %>% 
  show_query()
```

```sql
SELECT
  store_sales.*,
  IFNULL(sales, 0.0) AS sales2,
  REVERSE(store) AS store_rev,
  QUANTILE_CONT(profit, 0.25) OVER () AS per25
FROM store_sales
```

```text
  store month sales profit sales2 store_rev per25
  <chr> <int> <dbl>  <dbl>  <dbl> <chr>     <dbl>
1 S001      4   150     30    150 100S       27.8
2 S001      5   170     34    170 100S       27.8
3 S001      6   140     27    140 100S       27.8
4 S001      7   160     32    160 100S       27.8
5 S002      4    NA     28      0 200S       27.8
...
```

これにより、必要な SQL をほぼ自由に生成できるようになります。

> [!WARNING]
> 使用するデータベースに応じて適切な関数を選択してください。

#### dbplyr が認識できない関数をエラーにする

デフォルトでは、dbplyr は変換できない関数をそのまま SQL に渡そうとしますが、`dplyr.strict_sql` オプションを `TRUE` に設定すると、変換できない関数を使用した際に **強制的にエラーを発生させる** ことができます。  

```r {name="R"}
options(dplyr.strict_sql = TRUE)

db_sales %>% mutate(v = EVEN(month))
```

```text
Error in `EVEN()`:
! Don't know how to translate `EVEN()`
```

このオプションを有効にすると、誤った SQL クエリの生成を防ぎ、SQL 変換の正確性を高めるのに役立ちます。  

## まとめ

今回は、R のコードがどのように SQL クエリへと変換されるのかを解説しました。  
dplyr を活用することで、SQL を直接記述せずとも直感的な R コードからクエリを自動生成でき、SQL の構造や書き方を効率的に習得できます。

また、SQL に変換できる形で R コードを書くことで、データ処理をデータベース側で実行できるため、パフォーマンスの向上も期待できます。

R と SQL を組み合わせることで、より柔軟かつ強力なデータ処理が可能になります。  
ぜひ今回紹介した手法を活用し、実践を通じて SQL の理解を深めながら、さらに高度なデータ分析に挑戦してみてください。

## 参考リンク

- **[R for Data Science (2e) - Databases](https://r4ds.hadley.nz/databases)** (*Hadley Wickham 他*)
- **dbplyr** (*公式サイト*)
  - {{< href-target-blank url="https://dbplyr.tidyverse.org/articles/translation-verb.html" text="Verb translation" >}}
  - {{< href-target-blank url="https://dbplyr.tidyverse.org/articles/translation-function.html" text="Function translation" >}}
  - {{< href-target-blank url="https://dbplyr.tidyverse.org/reference/" text="Function reference" >}}

---

{{< aff/moshimo/ds100kdp >}}


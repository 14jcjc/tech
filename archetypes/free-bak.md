---
{{ $id := lower .File.ContentBaseName -}}
title: "{{ .Site.Params.k100.site.rsql }}{{ .Site.Params.k100.site.titleF }} - {{ upper .File.ContentBaseName }} (解説)"
date: "{{ .Date }}"
lastmod:
slug: "{{ $id }}"
# draft: true
description: ""
# summary: ""
# weight: 10
categories: 
  - "データサイエンス100本ノック＋α"
tags: 
params:
  question: 
    edition: standard
    id: "{{ upper $id }}"
# disableShare: false
# UseHugoToc: true
# ShowToc: false
# TocOpen: false
# tableOfContents:
#   ordered: false
#   startLevel: 2
#   endLevel: 5
---

{{% include "/ds-drills/data-tf/free/template/tmp-head.md" %}}

## 演習問題

{{< k100/question >}}

出力イメージ:

```text

```

{{% include "/ds-drills/data-tf/template/tmp-setup.md" %}}

---

解答例を以下に示します。  
各コードの出力結果を統一させるため、`` の昇順でソートしています。

## R (データフレーム操作)

### 利用するデータ

以下のデータを利用します。  

- **`df_`** の (`aaa`) と (`bbb`)

主要なカラムを抜き出してデータの一部を確認します。

```r {name="R"}

```

```text

...
```

### 解答例と実行結果

```r {name="R"}

```

```text

```

### 解説


## R (データベース操作)

### 利用するデータ

以下のデータを利用します。  

- **`db_`** の (`aaa`) と (`bbb`)

主要なカラムを抜き出してデータの一部を確認します。

```r {name="R"}

```

```text

...
```

### 解答例と実行結果

```r {name="R"}

```

```text

```

### 解説

データフレーム操作との相違点は以下の箇所です。

- **`db_result %>% collect()`**  
  データベース操作の結果を R のデータフレーム (tibble) として取得します。

## SQL

### 利用するデータ

以下のデータを利用します。

- **`ttt`** テーブルの (`aaa`) と (`bbb`)

主要なカラムを抜き出してデータの一部を確認します。

```r {name="R"}

```

```text

...
```

### 自動生成された SQL クエリ

データベース操作による結果 (`db_result`) に基づき、自動生成された SQLクエリを `show_query()` で確認できます。

```r {name="R"}

```

```sql {name="SQL"}

```

### 解答例

このクエリをより簡潔な形に書き直すと、次のようになります。

```sql {name="SQL"}

```

### 解説


### 実行結果の確認

この SQLクエリの実行結果は、次のようにして確認できます。

```r {name="R"}

```

```text

```

{{% include "/ds-drills/data-tf/free/template/tmp-foot.md" %}}

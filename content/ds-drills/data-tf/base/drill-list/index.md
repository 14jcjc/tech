---
title: "[R & SQL] 演習問題一覧"
slug: "drill-list"
date: 2025-02-23T01:01:20+09:00
draft: false
# draft: true
weight: 20
description: "当サイトで紹介している R と SQL の演習問題一覧です。"
summary: "当サイトで紹介している R と SQL の演習問題一覧です。"
categories: ["R & SQL 演習問題 - 基本情報"]
tags: 
image: cover-drill-list.png
toc: true
ShowToc: false
# menu: 
#     main:
#       name: "R+SQL 演習問題"
#       weight: 10
#       params: 
#         icon: question
---

- 各設問は、以下の 3 種類のコードを書くというスタイルです。
  - **Rコード (データフレーム操作)**
  - **Rコード (データベース操作)**
  - **SQLクエリ**
- <font color="#F0B007">★</font>の数は難易度を表します。(3 種類のコードから総合的に判定しています。)

<br>

<div class="list-toggle">
  <div class="row-title">
    <!-- <label> -->
  	<label class="disabled-label">
      <input type="radio" name="edition-toggle" value="all" disabled>
      <span>全て</span>
    </label>
    <label>
      <input type="radio" name="edition-toggle" value="standard" checked>
      <span>{{< param k100.site.titleF >}}</span>
    </label>
    <!-- <label> -->
    <label class="disabled-label">
      <input type="radio" name="edition-toggle" value="advanced" disabled>
      <span>オリジナル問題集</span>
    </label>
    {{% comment %}}
      <span>{{< param k100.site.titleP >}}</span>
    {{% /comment %}}
  </div>

  <div class="row-sort">
    <label for="order-1">
      <input type="radio" id="order-1" name="order-toggle" value="id" checked>
      <span>番号順</span>
    </label>
    <label for="order-2">
      <input type="radio" id="order-2" name="order-toggle" value="difficulty-desc">
      <span>難易度降順</span>
    </label>
    <label for="order-3">
      <input type="radio" id="order-3" name="order-toggle" value="difficulty-asc">
      <span>難易度昇順</span>
    </label>
  </div>
</div>

<!-- 全9パターンのリスト (最初はデフォルト以外を非表示にしておく) -->
<div id="list-id-all" class="question-list" style="display:block;">
  <div class="edition-title">全て (番号順)</div>
<!-- 
- [{{% param k100.site.titleP %}}]({{< ref "#a" >}})
- [{{% param k100.site.titleF %}}]({{< ref "#s" >}})
-->

## {{% param k100.site.titleP %}}{#a}
  
  {{< k100/q-list ed="advanced" root=".." sortkey="id" order="asc" >}}

## {{% param k100.site.titleF %}}{#s}
  
  {{< k100/q-list ed="standard" root=".." sortkey="id" order="asc" >}}

  {{% comment %}}
  {{< k100/q-list ed="standard,advanced" root=".." sortkey="id" order="asc" >}}
  {{% /comment %}}
</div>

<div id="list-difficulty-desc-all" class="question-list" style="display:none;">
  <div class="edition-title">全て (難易度降順)</div>
  {{< k100/q-list ed="standard,advanced" root=".." sortkey="difficulty" order="desc" >}}
</div>

<div id="list-difficulty-asc-all" class="question-list" style="display:none;">
  <div class="edition-title">全て (難易度昇順)</div>
  {{< k100/q-list ed="standard,advanced" root=".." sortkey="difficulty" order="asc" >}}
</div>

<div id="list-id-standard" class="question-list" style="display:none;">
  <div class="edition-title">{{< param k100.site.titleF >}} (番号順)</div>
  {{< k100/q-list ed="standard" root=".." sortkey="id" order="asc" >}}
</div>

<div id="list-difficulty-desc-standard" class="question-list" style="display:none;">
  <div class="edition-title">{{< param k100.site.titleF >}} (難易度降順)</div>
  {{< k100/q-list ed="standard" root=".." sortkey="difficulty" order="desc" >}}
</div>

<div id="list-difficulty-asc-standard" class="question-list" style="display:none;">
  <div class="edition-title">{{< param k100.site.titleF >}} (難易度昇順)</div>
  {{< k100/q-list ed="standard" root=".." sortkey="difficulty" order="asc" >}}
</div>

<div id="list-id-advanced" class="question-list" style="display:none;">
  <div class="edition-title">{{< param k100.site.titleP >}} (番号順)</div>
  {{< k100/q-list ed="advanced" root=".." sortkey="id" order="asc" >}}
</div>

<div id="list-difficulty-desc-advanced" class="question-list" style="display:none;">
  <div class="edition-title">{{< param k100.site.titleP >}} (難易度降順)</div>
  {{< k100/q-list ed="advanced" root=".." sortkey="difficulty" order="desc" >}}
</div>

<div id="list-difficulty-asc-advanced" class="question-list" style="display:none;">
  <div class="edition-title">{{< param k100.site.titleP >}} (難易度昇順)</div>
  {{< k100/q-list ed="advanced" root=".." sortkey="difficulty" order="asc" >}}
</div>

<script>
  document.addEventListener("DOMContentLoaded", function() {
    const questionLists = document.querySelectorAll('.question-list');
    const toc = document.querySelector('.widget.archives');
    // const tocContent = document.querySelector('.tocContent');
    // const toc = document.querySelector('.widget--toc');
    // const tocTitle = document.querySelector('.widget-title.section-title'); // h2を取得

    const updateList = () => {
      const selectedOrder = document.querySelector('input[name="order-toggle"]:checked')?.value;
      const selectedEdition = document.querySelector('input[name="edition-toggle"]:checked')?.value;

      if (!selectedOrder || !selectedEdition) return; // チェックされていない場合は処理を中断

      // すべてのリストを非表示に
      questionLists.forEach(list => list.style.display = 'none');

      // 選択されたリストを表示
      const targetList = document.getElementById(`list-${selectedOrder}-${selectedEdition}`);
      if (targetList) {
        targetList.style.display = 'block';
      }

      // 目次の表示・非表示を制御
      if (selectedOrder === "id" && selectedEdition === "all") {
        toc.style.display = "block";
        // tocContent.style.display = "block";
      } else {
        toc.style.display = "none";
        // tocContent.style.display = "none";
      }

      // 選択状態を localStorage に保存
      localStorage.setItem('order', selectedOrder);
      localStorage.setItem('edition', selectedEdition);
    };

    // ページ読み込み時の処理
    // let savedOrder = localStorage.getItem('order');
    // let savedEdition = localStorage.getItem('edition');
    // // ラジオボタンに適用（なければ 'standard' に強制）
    // const orderRadio = document.querySelector(`input[name="order-toggle"][value="${savedOrder || 'id'}"]`);
    // const editionRadio = document.querySelector(`input[name="edition-toggle"][value="${savedEdition || 'standard'}"]`);
    // if (orderRadio) orderRadio.checked = true;
    // if (editionRadio) editionRadio.checked = true;

    // ページ読み込み時の処理
    let savedOrder = localStorage.getItem('order') || "id";
    let savedEdition = localStorage.getItem('edition') || "standard";

    // ラジオボタンに適用
    const orderRadio = document.querySelector(`input[name="order-toggle"][value="${savedOrder}"]`);
    const editionRadio = document.querySelector(`input[name="edition-toggle"][value="${savedEdition}"]`);

    if (orderRadio) orderRadio.checked = true;
    if (editionRadio) editionRadio.checked = true;

    // イベントリスナーを設定
    document.querySelectorAll('input[name="order-toggle"], input[name="edition-toggle"]').forEach(radio => {
      radio.addEventListener('change', updateList);
    });

    // 初回のリスト表示
    updateList();
  });
</script>

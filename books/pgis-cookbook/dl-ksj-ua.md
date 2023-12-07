---
title: "国土数値情報のダウンロード"
---

# はじめに

国土数値情報 https://nlftp.mlit.go.jp/ksj/ で各種データをダウンロードできます。

ここでは、都市地域を例にとって、ダウンロードしてみましょう。

# サイトから目的のデータまで

まずトップページに行きます。

![国土数値情報 トップページ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-ksj/01-ksjtop.png)

下の方にスクロールさせたり、ブラウザのページ内検索機能を使って見つけてください。

![都市地域 (ポリゴン) のありか](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-ksj/03-select.png)

そこをクリックします。

# ダウンロード手順

必要なファイルを見つけます。ここでは広島県の最新のデータを求めています。

![都市地域 (ポリゴン) の広島県の最新データのありか](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-ksj/04-hirosima.png)

アンケートが表示されるので、**できるだけご協力を**お願いします。

![アンケート画面](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-ksj/05-enq.png)

アンケートを入れるかスキップすると、ダウンロードが始まります。

![ダウンロード確認](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-ksj/06-dl.png)

![ダウンロード終了](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-ksj/07-dldone.png)


# QGIS で見てみる

QGISで見てみましょう。

メニューから「レイヤ (L)」→「レイヤを追加」→「ベクタレイヤの追加 Ctrl+Shift+V」を選択します。

![ベクタレイヤを追加するためのメニューのパス](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-fpoly_06-qgis-add-menuall.png)

「データリソースマネージャ|ベクタ」が出ます。サイドのタブでラスタなど好きな形式も選べますが、とりあえず「ベクタ」のままいきましょう。

![データリソースマネージャ ダイアログ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-fpoly_07-qgis-add-dialog-initial.png)

「ソース型」は「ファイル (i)」を選択したまま、``.shp``ファイルを選択します。ファイルを指定した後は、「追加」ボタンをクリックすると、ロードを開始します。

![シェープファイルを選択した直後](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-ksj/08-qgis-add-dialog-selected.png)

都市計画区域が表示されます。

![都市計画区域を表示したところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-ksj/09-qgis-firstview.png)

# おわりに

「国土数値情報（都市地域）」を例にとった、国土数値情報のダウンロードは以上です。
都市地域だけでなく、行政区域など様々なデータがありますので、要チェックのサイトです。

# 出典

本章の作成にあたり、次のデータを使用しました。

* 「国土数値情報 (都市地域データ)」 (国土交通省) https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-A09.html (2023年12月1日取得)

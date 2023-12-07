---
title: "筆ポリゴンのダウンロード"
---

# はじめに

筆ポリゴンをダウンロードします。

全国の農地ポリゴンが集まっているので、試しに使う場合には、ひとつの市町村に限定するようにすることをお勧めします。

# ダウンロード手順

https://www.maff.go.jp/j/tokei/porigon/ から閲覧/ダウンロード用Webアプリに行けます。

![筆ポリゴン公開サイトの上部](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-fpoly/01-website-1.png)

上のWebアプリは閲覧用ですので、下の方にスクロールさせて「筆ポリゴンデータダウンロードページ」のボタンをクリックします。

![筆ポリゴン公開サイトの下部のダウンロードボタンを表示したところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-fpoly/02-website-2.png)

アンケートが表示されるので適切に回答して下さい。

![筆ポリゴンの利用に関するアンケート](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-fpoly/03-predl-enq.png)

完了したらダウンロードページが現れるので、必要な県、市を選択して下さい。

![筆ポリゴンダウンロードページ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-fpoly/04-download.png)

ZIPファイルでダウンロードできます。ダウンロードに成功したら、JSONファイルが入っています。

![ZIPファイルの中身](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-fpoly/05-content.png)

# QGIS で見てみる

QGISで見てみましょう。

メニューから「レイヤ (L)」→「レイヤを追加」→「ベクタレイヤの追加 Ctrl+Shift+V」を選択します。

![ベクタレイヤを追加するためのメニューのパス](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-fpoly/06-qgis-add-menuall.png)

「データリソースマネージャ|ベクタ」が出ます。サイドのタブでラスタなど好きな形式も選べますが、とりあえず「ベクタ」のままいきましょう。

![データリソースマネージャ ダイアログ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-fpoly/07-qgis-add-dialog-initial.png)

「ソース型」は「ファイル (i)」を選択したまま、GeoJSONファイルを選択します。ファイルを指定した後は、「追加」ボタンをクリックすると、ロードを開始します。

![GeoJSONファイルを選択した直後](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-fpoly/08-qgis-add-dialog-selected.png)

ロードは意外と時間がかかり、やや不安になりますが、少し待つと、ロードが終わり、ゆっくり表示されていきます。

![筆ポリゴンを表示したところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dl-fpoly/09-qgis-all.png)

# おわりに

筆ポリゴンのダウンロードは以上です。上述の通り全国のデータがあるので、必要に応じてダウンロード、活用して下さい。

# 出典

本章の作成にあたり、次のデータを使用しました。

* 「筆ポリゴンデータ」 (農林水産省) https://open.fude.maff.go.jp/ (2023年11月28日取得)

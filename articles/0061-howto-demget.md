---
title: "基盤地図情報DEM取得アプリの使い方"
emoji: "😀"
type: "tech"
topics: [GIS]
published: true
---

# はじめに

[基盤地図情報DEM取得アプリの使い方](https://boiledorange73.sakura.ne.jp/jpdemget/)は、10mメッシュかつ地形図から取得した ("10B"と言われます)ものですが、任意の標高データを取得できます。

ここでは使用方法を示したいと思います。

# 基本的な使い方

ページを開くと次のような画面が表示されます。

![初期画面](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/01-initial.png)

マウス、センターホイール、ピンチ等を使って地図上を移動します。取得したい場所付近をクリックすると、照準が出ます。

![照準が出たところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/02-clicked.png)

もっと拡大表示すると、取得範囲が分かります。

![取得範囲を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/03-zoomin.png)

ここで **取得** ボタンを押すと取得できます。

QGIS等で見ることができます。

![QGISで表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/04-qgis.png)

# もっと広い範囲を取得したい

**セル横**、**縦**の値を変更します。最大はそれぞれ1000です。

![横1000縦1000の範囲](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/05-maxsize.png)

さらに広い範囲を取得したい場合には、アンカー位置を「左上」「右上」「右下」「左下」と順に変えて取得してみて下さい。

![アンカー位置を左上にした場合](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/06-ul.png)

![アンカー位置を右上にした場合](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/07-ur.png)

![アンカー位置を右下にした場合](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/08-lr.png)

![アンカー位置を左下にした場合](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/09-ll.png)

全てQGISにロードして、最小値、最大値をそろえると、横2000縦2000相当のラスタが手に入ります。

![縦横2倍をロードした例](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/10-2x2.png)

# おわりに

いかがだったでしょうか？

ご自身にとって必要な箇所をクリックすれば、そこのDEMを取得できます。また、縦横2倍まで取得する方法を示しました。ぜひ取って行って下さい

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

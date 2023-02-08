---
title: "基盤地図情報DEM取得アプリの使い方"
emoji: "😀"
type: "tech"
topics: [GIS]
published: true
---

# はじめに

[基盤地図情報DEM取得アプリ](https://boiledorange73.sakura.ne.jp/jpdemget/)は、10mメッシュかつ地形図から取得した ("10B"と言われます)ものですが、任意の標高データを取得できます。

ここでは使用方法を示したいと思います。

# 基本的な使い方

ページを開くと次のような画面が表示されます。

![初期画面](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/01-initial.png)

マウス、センターホイール、ピンチ等を使って地図上を移動します。

![照準が出たところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/02-zoomin.png)

取得したい場所付近をクリックすると、照準が出ます。同時に、取得範囲の四角形が表示されます (セル数が入っていない場合は表示されません)。

![取得範囲を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/03-clicked.png)

ここで **取得** ボタンを押すと取得できます。

QGIS等で見ることができます。

![QGISで表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/04-qgis.png)

# 取得範囲を調整する

## 取得セル数の変更

「セル数」を変更することができます。ただし**最大セル数**は縦横ともに**1000**です。

次に横セル数を300に変更した際の取得範囲を示します。前の取得範囲は横200、縦200でしたので、比較してみて下さい。

![横セル数を300に変更した際の取得範囲](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/05-chcells.png)

## 照準のドラッグ

**照準はドラッグできます**。照準を動かすと取得範囲も連動して動ぐので、取得範囲の調整に使えます。

## 取得範囲ボックスの展開方向の変更

取得範囲は、照準位置とセル数から決まりますが、照準位置からどの方向に展開するかは、ラジオボタン (ここでは「照準原点」と言うことにします)で変更できます。

まず、あまり深く考えないなら、照準原点に中心を指定するのをお勧めします。

![中心を選択した際の取得範囲の展開](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/06-1-cc.png)

照準原点に左上を選択すると、照準位置が左上隅にくるように取得範囲を決めます。このため、照準から左下方向に展開されます。

![左上を選択した際の取得範囲の展開](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/06-2-ul.png)

右上だと次のようになります。

![右上を選択した際の取得範囲の展開](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/06-3-ur.png)

右下だと左上に展開します。

![右上を選択した際の取得範囲の展開](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/06-4-lr.png)

左下も確認して下さい。

![左下を選択した際の取得範囲の展開](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/06-5-ll.png)

# もっと広い範囲を取得したい

**セル横**、**縦**の値を変更します。最大はそれぞれ1000です。

![横1000縦1000の範囲](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/07-1-maxsize.png)

さらに広い範囲を取得したい場合には、アンカー位置を「左上」「右上」「右下」「左下」と順に変えて取得してみて下さい。

![アンカー位置を左上にした場合](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/07-2-ul.png)

![アンカー位置を右上にした場合](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/07-3-ur.png)

![アンカー位置を右下にした場合](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/07-4-lr.png)

![アンカー位置を左下にした場合](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/07-5-ll.png)

全てQGISにロードして、最小値、最大値をそろえると、横2000縦2000相当のラスタが手に入ります。

![縦横2倍をロードした例](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0061/10-2x2.png)

# おわりに

いかがだったでしょうか？

ご自身にとって必要な箇所をクリックすれば、そこのDEMを取得できます。また、縦横2倍まで取得する方法を示しました。ぜひ取って行って下さい

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

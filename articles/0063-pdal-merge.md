---
title: "Cloud Optimized と言われたらとりあえず大きめのファイルを作ってQGISで見るものである"
emoji: "😀"
type: "tech"
topics: [PDAL, GIS]
published: true
---

# はじめに

COPCを表示できるのが分かったのですが、せっかく Cloud Optimized を銘打ってるのだから、もっと広い範囲をマージしてもいいのではないかと思いました。

ということで、テストとして、6ファイルをマージしたうえで、マージしたファイルを表示してみました。

あと、ふとQGISで Cloud Optimized GeoTIFF とか開けるから、COPCもいけるか検討してみまいた。

## 材料

https://wakayamaken.geocloud.jp/mp/22 から ``06QC282_org.txt``, ``06QC284_org.txt``, ``06QC291_org.txt``, ``06QC293_org.txt``, ``06QC382_org.txt``, ``06QC391_org.txt`` を取得しました。これらの6ファイルを使用します。

# COPCファイルを作ってマージすると失敗した

まず、textからCOPCへの変換を行った後、これらをマージしたらCOPCそのままで出力できるだろうと判断しました。

変換はおいといて、マージは次のようになります。

```
% pdal merge *_org.laz out.laz

% pdal info --metadata out.laz
...
    "copc": false,
...
```

COPCになってない。ダメじゃん…。

なお余談ですが、textからCOPCへの変換で約5分、マージに約3分かかりました。

# LASファイルを作ってマージしてCOPCにするといけた

気を取り直して、LASでマージ出力してから最後のCOPCにする方針を取りました。

```sh:convert_all.sh
#!/bin/sh

for fin in *_org.txt
do
  fout=`echo $fin|sed 's/\\.txt$/.las/'`
  echo "$fin -> $fout"
  pdal translate \
    --reader text \
    --readers.text.separator="," \
    --readers.text.header="ID,X,Y,Z,B" \
    --readers.text.spatialreference="EPSG:6674" \
    "$fin" \
    "$fout"
done
```

```
% ./convert_all.sh
```

```
% pdal merge *_org.las merged.las
```

```
% pdal translate \
  --writer copc --writers.copc.forward=all \
  merged.las merged.copc.laz
```

```
% pdal info --metadata merged.copc.laz
{
...
    "compressed": true,
    "copc": true,
...
```

これなら OK そうです。

なお、textからlas変換で約4分、マージに約1分、lasからCOPCに3分かかりました。

# では見てみましょう

https://viewer.copc.io/?copc=https://boiledorange73.sakura.ne.jp/merged.copc.laz

![merged.copc.laz全体](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0063/01-overview.png)

前よりも広く見えてますね。概ね妙寺小学校の校区内はカバーできてるのではないでしょうか。

# QGISで対応してるが .copc.laz でないといけない

QGISで、Cloud Optimized GeoTIFF などに対応してくれていたので、 https://boiledorange73.sakura.ne.jp/merged.copc.laz を開けるのではないか?と思ったので、開こうとしてみました。

![QGISの点群レイヤ追加ダイアログ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0063/02-qgis-dialog.png)

うん、この「プロトコル」は見たことがある。COGでは、こちらで URL を指定したらロードできた。ということは、これもいけそう。

と思ったら、ひとつ問題。

ファイル名を ``.copc.laz``にしないといけません。

ファイル名から判定しているっぽくて、食ってくれません。

## 断面図を楽しむ

QGISで「ビュー (V)」→「標高断面図」を開くと、指定したルートに沿った断面図を出せます。ためしに、三谷から新田を引いて、新田の標高の低さを確認しています。紀ノ川の河川敷にあるグラウンドより低いように見えます。さすが増水するとコイやらナマズやらが田んぼに迷い込むところだけあります。

![標高断面図を見る](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0063/03-qgis-cross.png)


# おわりに

いかがだったでしょうか？

マージするのに、COPC をマージしても LAS になってしまうという問題はありましたが、マージしてから COPC にすれば OK で、順番だけ気を付ければうまくいくことが確認できました。

それと、COPC はネットを介して QGIS が食ってくれるので、なかなか面白そうです。

# 出展

本稿の画像および merged.copc.laz の作成にあたっては「3次元点群データ」(和歌山県) (https://wakayamaken.geocloud.jp/mp/22) を使用しました。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

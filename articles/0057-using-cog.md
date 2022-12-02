---
title: "Cloud Optimized GeoTIFFを置いてみました"
emoji: "😀"
type: "tech"
topics: [GIS, GDAL]
published: true
---

# はじめに

## FOSS4G Advent Calendar 2022

この記事は [FOSS4G Advent Calendar 2022](https://qiita.com/advent-calendar/2022/foss4g) として書いたものです。

しかしなんでみんなハードル高くしていくんや…。

## (余談) 某サーバから持ち出したものです

本節は、分からない方は読み飛ばして下さい。

某サーバが終了します。ここで言うのも何ですが、申し訳ありません。

某サーバ閉鎖に関連して、代替のサービスを検討しないといけないと思いますが、たとえば、地図画像配信サービスは他サービスも多いので、そちらを使って頂くこととして、こちらは廃止でいいと判断しました。

残るのと言えば、アレとアレと、あとはアレぐらいです。

一つ目のアレのPostGISマニュアルは静的ページを公開できるところに回しました。CC BY-SAなのでGitHubにも持ち出しています。

二つ目のアレは、迅速測図、東京測量図原図はオープンデータになっているということで、 https://boiledorange73.sakura.ne.jp/ に持ち出しました。

あと、三つ目のアレは、1秒1回以上のアクセスが来てるサービスですが、職場の持っているプログラムであり、おいそれと持ち出せない状態で、実質廃止ですが、一般には触れられないところでひっそり生きながらえさせる方向です。

## 本稿の概要

持ち出し先サーバでは、迅速測図、東京測量図原図のタイルサービスの提供と共に、地図データをダウンロードできるようにしています。

地図データの書式としては、mbtiles と共に **Cloud Optimized GeoTIFF (COG)** を出しています。

今回はCOGの話をしてみたいと思います。

# まずCOGとは

Cloud Optimized GeoTIFF の仕様は https://github.com/cogeotiff/cog-spec/blob/master/spec.md にあります。

IFD (Image File Directory)を使って、各ファイルではタイル化している、ようです。歯切れ悪いな。

IFDについては、https://zenn.dev/mith_mmk/articles/076ef011b063fa あたりを参照してみるとなんとなく分かります。タイルについても、このページ内で"TileOffsets" や "TileByteCounts" 等で検索してみて下さい。

タイルのイメージについては、https://www.element84.com/blog/cloud-optimized-geotiff-vs-the-meta-raster-format あたりも参照になると思います。

要は、ピラミッド画像を一つのファイルに入れておいて、各ピラミッド画像について、タイル化して、そのタイルのオフセット（ファイル上の先頭位置）をTIFFタグ"TileOffsets"で全て保存してある、というものです。

まずTIFFタグにあたる部分付近だけダウンロードして、TIFFタグを順に読んで辿っていくと、所望するタイルのオフセットが分かる、ということなんだろうと思います。
そして、HTTPリクエストで"Range"フィールドを指定して、ファイルのうちタイルに該当する部分だけをダウンロードします。
こうやって、表示に最低限必要なところだけをダウンロードするだけで済ますことで、大幅なダウンロード時間短縮ができるようです。

# WMSサーバのビルドに失敗

なんでCOGを置こうとしたかというと、WMSサーバのビルドに失敗したためです。

共有サーバを借りました。ここは FreeBSD が使えるためです。

しかし、よくよく考えたら、ここ10年ほどは、PortsとPackagesに頼りっぱなしで、ソースを落として自前でビルとする機会がずいぶんと少なくなっていました。

その状態で、WMSサーバとして考えていた MapServer のビルドに失敗しました。失敗というか、うまくいかずに諦めました。

タイルサービスも同じサイトで出しているので、特に困らないのですが、WMSの代替が欲しいなと思った次第です。

# 作ってみよう

まず、GDALで本格的な作業をする際は作業キャッシュを大増量させます。今回は10Gバイトです。

```
set GDAL_CACHEMAX=10240
```

東京5000のGeoTIFF (ピラミッド無し)を、オリジナルのEPSG:3100からEPSG:3857に投影変換を行いつつ複写。

```
gdalwarp -t_srs epsg:3857 -r lanczos tokyo5000-3100.tif tokyo5000-3857.tif
```

5分かかりました。

次にCOGにしてみます。

```
gdal_translate tokyo5000-3857.tif cog-tokyo5000-3857.tif
  -of COG -co COMPRESS=DEFLATE -co RESAMPLING=LANCZOS
```

これ、4分かかっただけです。思ったより早かったです。

次に迅速測図でも同じことをしてみます。

まず、EPSG:3100からEPSG:3857への投影変換です。

```
gdalwarp -t_srs epsg:3857 -r lanczos kanto_rapid-3100.tif kanto_rapid-3857.tif
```
これは、62分かかりました。概ね1時間。

次に、迅速測図 COGにしました。

```
gdal_translate kanto_rapid-3857.tif cog-kanto_rapid-3857.tif -of COG -co BIGTIFF=YES -co COMPRESS=DEFLATE -co RESAMPLING=LANCZOS
```

この作業は、89分かかりました。概ね1.5時間。

なお、この作業で "BIGTIFF=YES" の指定を忘れると、

```
ERROR 1: TIFFAppendToStrip:Maximum TIFF file size exceeded. Use BIGTIFF=YES creation option.
```

と言われます。ただ、BIGTIFFでないTIFFで出力しようとして失敗しただけなので、BIGTIFFで出せば問題ありません。


# QGISで使ってみよう

QGISで普通に**ラスタレイヤを追加**を選択して下さい。

![QGISメニュー](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0057/01-menu.png)

ダイアログが出るので、「ソースタイプ」を「プロトコル」にします。

続いて、URI欄にCOGファイルのURL (ここでは ``https://boiledorange73.sakura.ne.jp/data/cog-kanto_rapid-3857.tif`` ) を指定します。

![データソースマネージャ|ラスタ ダイアログ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0057/02-dialog.png)

以上です。見慣れた迅速測図が見ることができます。

![迅速測図を表示中](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0057/03-qgis-start.png)

移動、拡大、縮小も可能です。

![迅速測図で谷田部内町を表示中](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0057/04-qgis-yatabe.png)

## WSLと比べて遅くない (個人の感想)

ダウンロードに時間がややかかる (2秒程度) 場合があったり、すぐに見れたり、バラバラです。

拡大、縮小で時間がかかり気味に思います。とすると、キャッシュにヒットしているかそうでないかの差なのかも知れません。

WMSと比べて、私は、似たような応答時間に感じました。

# gdal_translate で取り込んでみよう

GDALのデータソース指定は、こっそりいろんなプロトコルに対応してくれてたりします。

HTTP/HTTPSにも対応しています。

```csh
% gdal_translate -projwin 15557500 4256300 15560000 4254500 \
  /vsicurl/https://boiledorange73.sakura.ne.jp/data/cog-kanto_rapid-3857.tif \
  -of GTIFF out.tif
Input file size is 102837, 105139
0...10...20...30...40...50...60...70...80...90...100 - done.
% 
```

ダウンロード開始まで1秒弱待ちますが、始まるとすぐ終わります。

QGISで開いてみて、しっかり見れました。

![ダウンロードした迅速測図をQGISで表示中](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0057/05-downloaded.png)

# おわりに

いかがだったでしょうか?

大縮尺表示 (縮小表示) の時に、COGでないラスタだとダウンロードに非常に時間がかかると思われるのですが、大してかからずに表示できました。

これは、もうWMS使わなくても、COGが使えるなら**COGで充分行けそう**な気がしました。

ただ、WMSだとHTTPのクエリパラメータをいじるだけで所望の画像が得られたのでwgetとかと相性が良かったのですが、COGだとQGISかGDALかを入れないと使いにくいのがちょっとイヤだなと思いました。

今後は、DEMのラスタファイルを同じようにして全国を見れるようにするのもいいかも知れないなと思ったりしてますが、地理院さんの複製承認が必要なので、今のところは公開していません。


# 出典

COGファイルのソースデータは、農研機構農業環境変動研究センター ( https://habs.rad.naro.go.jp/ ) が作成なさったもので、[CC-BY 2.1 JP](https://creativecommons.org/licenses/by/2.1/jp/)の下に提供されています。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

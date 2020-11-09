---
title: "兵庫県1m DEMをさわってみる"
emoji: "😀"
type: "tech"
topics: [QGIS]
published: true
---
# はじめに

https://web.pref.hyogo.lg.jp/press/20200110_4566.html によると、兵庫県内の1mメッシュ(!)のDSM, DEMm CS立体図がダウンロードできます。

これは、これまでに類を見ない、大変なことです。全国整備されている基盤地図情報は5mメッシュと10mメッシュです。

さらに、ライセンスはCC-BYです。

そこで、試しに DEM を落として、様子を見てみました。

# データのダウンロード

https://www.geospatial.jp/ckan/dataset/2010-2018-hyogo-geo-potal から CS立体図, DSM, DEM がダウンロードできます。

DEMは https://www.geospatial.jp/ckan/dataset/2010-2018-hyogo-geo-dem にて取得できます。

ZIPファイルがおかれていますので、必要なものをダウンロードします。

求める地域とZIPファイル名との関係が不明な場合には、"Indexmap (GeoJSONファイル)"を使用します。

"Indexmap (GeoJSONファイル)"をクリックすると、地図が表示されます。さらに地図上部のタブで「GeoJSON」を選択します。「Map Viewer」ではGeoJSONの属性が表示されません。メッシュをクリックすると"meshID"という属性の値が表示されます。これがZIPファイル名のプリフィックスです。

今回は、旧日高町付近を狙って 05LF90 にしました。

# GDALに食わせる

ZIPを展開すると、4つのtxtファイル(DEM_05LF901_1g.txt, DEM_05LF902_1g.txt, DEM_05LF903_1g.txt, DEM_05LF904_1g.txt)が現れました。

## まずは様子を見る

うち一つの様子を``gdalinfo``で見てみます。

```
gdalinfo DEM_05LF901_1g.txt
```

多少時間がかかります。たぶん全部読まないと画像サイズ等が計算できないからだろうと思います。

```
Driver: XYZ/ASCII Gridded XYZ
Files: DEM_05LF901_1g.txt
Size is 2000, 1500
Origin = (40000.000000000000000,-58500.000000000000000)
Pixel Size = (1.000000000000000,1.000000000000000)
Corner Coordinates:
Upper Left  (   40000.000,  -58500.000)
Lower Left  (   40000.000,  -57000.000)
Upper Right (   42000.000,  -58500.000)
Lower Right (   42000.000,  -57000.000)
Center      (   41000.000,  -57750.000)
Band 1 Block=2000x1 Type=Float32, ColorInterp=Undefined
  Min=5.090 Max=98.760
```

ということで、XYZファイルで、GDALで読める(OGRではないです)ことが分かりました。

## 空間参照系を推定する

このデータで使われているのは、たぶん、平面直角座標系のV系です。

まず、単位が度でないのは明らかです。そうすると、わが国の陸域ではUTMか平面直角座標系のいずれかです。

Y値が負になっているので、UTMは外れます。UTMの原点が赤道上であるため、北半球ではY値が負になることはありえません。なお、普通は南半球でも正数となるようにオフセットを足します（北半球とは別の系となります）。

平面直角系の何系かは適用範囲から分かります。[わかりやすい平面直角座標系](https://www.gsi.go.jp/sokuchikijun/jpc.html)から、兵庫県ならV系であることが分かります。

平面直角座標系V系の空間参照系コードは``EPSG:6673``です。

## GeoTIFFにする

gdalwarpで複数のXYZファイルをまとめて一つのGeoTIFFにしてみます。

```
gdalwarp -s_srs EPSG:6673 DEM_05LF901_1g.txt DEM_05LF902_1g.txt DEM_05LF903_1g.txt DEM_05LF904_1g.txt DEM_05LF90.tif
```

やはり、XYZファイルの読み込みに時間がかかるので、ちょっと待ってください。

今回は一つのファイルにまとめましたが、必ずしもまとめる必要はありませんが、XYZファイルだと時間がかかるので、GeoTIFF等の一般的なラスタへの変換はやっておいた方がいいかと思います。

```
gdalinfo DEM_05LF90.tif
Driver: GTiff/GeoTIFF
Files: DEM_05LF90.tif
Size is 4000, 3000
Coordinate System is:
PROJCRS["JGD2011 / Japan Plane Rectangular CS V",
(中略)
Data axis to CRS axis mapping: 2,1
Origin = (40000.000000000000000,-57000.000000000000000)
Pixel Size = (1.000000000000000,-1.000000000000000)
Metadata:
  AREA_OR_POINT=Area
Image Structure Metadata:
  INTERLEAVE=BAND
Corner Coordinates:
Upper Left  (   40000.000,  -57000.000) (134d46'27.00"E, 35d29' 7.51"N)
Lower Left  (   40000.000,  -60000.000) (134d46'26.47"E, 35d27'30.16"N)
Upper Right (   44000.000,  -57000.000) (134d49' 5.70"E, 35d29' 6.90"N)
Lower Right (   44000.000,  -60000.000) (134d49' 5.12"E, 35d27'29.55"N)
Center      (   42000.000,  -58500.000) (134d47'46.07"E, 35d28'18.53"N)
Band 1 Block=4000x1 Type=Float32, ColorInterp=Gray
```

ピクセルサイズも変わらないですし、大丈夫ですね。

# QGISで眺める

まずは、ラスタレイヤを追加して、グレースケールで見てみます。

![グレースケール](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0028/1.png)
出典：兵庫県_全域DEM（2010年度～2018年度）、兵庫県

次に陰影図を見ます。陰影図の作成は「ラスタ」→「解析」→「陰影図 (hillshade) ...」を選択します。

![陰影](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0028/2.png)
出典：兵庫県_全域DEM（2010年度～2018年度）、兵庫県

田んぼのあぜが見えます。

3D表示もやってみましょう。詳細は[標準装備のQGISで三次元表示ができるよ!](0022-pgis-qgis3d)を参照して下さい。

![3次元表示](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0028/3.png)
出典：兵庫県_全域DEM（2010年度～2018年度）、兵庫県

今回は、標高のグレースケールだけで表示していますが、空中写真等をかぶせると、結構見ごたえのある絵になります。

# おわりに

いかがだったでしょうか。

兵庫県さんが出されたのは1mメッシュという非常に細かいデータです。これを自由に扱えるのですから、とんでもない時代になったと思います。

このデータはGDALが難なく食べられるので、特に追加的なコード作成もなく使いまわせます。ただし、XYZファイルは読み込みに時間がかかるので、GeoTIFF等に変換しておいた方が良いと思います。

現時点の私は、兵庫県を舞台にして何かしているわけではないので、実際に使う場面は無いのですが、そんな者にも触らせてくれるのはありがたいです。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

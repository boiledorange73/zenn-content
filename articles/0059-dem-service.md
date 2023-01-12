---
title: "DEMのCOGを始めました"
emoji: "😀"
type: "tech"
topics: [GIS, MapServer, GDAL, PostGIS]
published: true
---

# とにかく言いたいこと

**[基盤地図情報DEM配信サービス](https://boiledorange73.sakura.ne.jp/dem.html)を公開しました！**

# あらためて、はじめに

[基盤地図情報DEM配信サービス](https://boiledorange73.sakura.ne.jp/dem.html)を公開しました。

基盤地図情報数値標高モデルの10mメッシュの任意範囲を取り出すものです。

本サービスでは、WCS (Web Coverage Service)とCOG (Cloud Optimized GeoTIFF)で出していますが、今回はCOG限定。

基盤地図情報からCOGを作るまでを見ていきます。

なお、COGについては["Cloud Optimized GeoTIFFを置いてみました](../0057-using-cog)を参照して下さい。


# 基盤地図情報DEMからGDALに読めるまで


## fgddem2pgsql

基盤地図情報はOGRで読めます。が、DEMは読めません。

と思ったら、こんなところに、基盤地図情報DEMをPostGISラスタのダンプ形式に変換するプログラムがあるじゃないですか。Visual C++でもビルド可能ですし。こんなの用意してくれてる人って、誰かは知らないですけど、とても素晴らしい人ですね、誰かは知らないですけど。

## PostGISに入れる

```
CERATE EXTENSION postgis;
CERATE EXTENSION postgis_raster;
```

```
CREATE TABLE dem10b (
    rid SERIAL PRIMARY KEY,
    rast RASTER,
    filename text
);
CREATE INDEX ix_dem10b_rast ON dem10b USING GiST (st_convexhull(rast));
```

PostGISラスタでは、直接のインデックスを作らずに``ST_ConvexHull(rast)``という関数インデックスにしている点です。

あと、fgddem2pgsqlでは``rid``の番号は付かないので、``rid``は``SERIAL``にしておいて下さい。

# Cloud Optimized GeoTIFF 作成

## GDALを使用

GDALを導入する必要があります。

GDALを使う際には GDAL_CACHEMAX を指定すると、速度向上が期待できます。おそらく磁気ハードディスク時代よりは速度向上率が下がっているとは思いますが、それでもgdalwap等の実行速度に重要な影響を及ぼすので、設定することをお勧めします。

GDAL_CACHEMAX の単位はメガバイトです。

たとえば、Windows コマンドプロンプトで次のようにすると、20,480メガバイト (20ギガバイト)のキャッシュを確保します。

```
set GDAL_CACHEMAX=20480
```

## gdalwarpでGeoTIFF生成

gdalwarpでPostGISラスタからGeoTIFFをに変換します (gdal_translate ではうまくいきませんでした)。

``PG``データソースに``mode=2``がありますが、これは「1テーブル 1ラスタ」モードを指定しています。テーブルの全てのタプルをまとめて一つのラスタとみなします。

デフォルト (``mode=1``)では、「1行 1ラスタ」モードになります。

ここでは、XMLファイル1個で1行を構成していて、かつ、全国まとめて1ラスタとしたいので``mode=2``が適切です。

それと、バッチファイルの改行は ``^`` です。``\``など、適宜読み替えて下さい。

```
gdalwarp  ^
  -of GTiff ^
  "PG:host=localhost port=5432 dbname='(テーブル名)' table='(テーブル名)' mode='2' password='(パスワード)'" ^
  -srcnodata "-9999" -dstnodata "-9999" ^
  -co COMPRESS=DEFLATE -co BIGTIFF=YES ^
  dem10b.tif
```

## 標高の最大値最小値を見てみよう

いちおう確認してみました。

時間もかかるので確認しなくてもいいですが、ちゃんと読み込めているっぽいのが分かります。

```
gdalinfo -mm dem10b.tif
Computed Min/Max=-134.600,3774.900
```

-134mってどこかはともかく、3774.9mは富士山ですね。

## COGを作る

COGは、少なくともGDAL 3.4.1 では、GDALがサポートするフォーマットの一つです。

ですので、あまり深く考えずに変換できます。

```
gdal_translate ^
  -of COG  -r average ^
  -co COMPRESS=DEFLATE -co BIGTIFF=YES ^
  dem10b.tif cog-dem10b.tif
```

# QGISで見てみましょう

## とりあえず見てみよう

[Cloud Optimized GeoTIFFを置いてみました](0057-using-cog)を参照して、QGISで開いてみて下さい。COGのURLは[基盤地図情報DEM配信サービス](https://boiledorange73.sakura.ne.jp/dem.html)に書いていますのでご覧下さい。

![初期状態](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0059/01-qgis-initial.png)

日本列島が見えたけど、これ、グレースケールの最大値と最小値がめちゃくちゃなので、変更します。

レイヤのプロパティを開いて、最大値と最小値を上書きします。

![プロパティを開こうとしているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0059/02-qgis-prop.png)

![最大値と最小値の上書きをしているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0059/03-qgis-symbology.png)

![最大値と最小値を整えた日本列島](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0059/04-qgis-jp.png)

富士山や日本アルプスのあたりが白で見えるようになりました。小縮尺では標高は平均値ですので、精度は考えないこと。

当然、拡大も可能です。

![拡大したところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0059/05-qgis-large.png)

## 陰影図を見てみよう

次に、陰影図を見てみましょう。

レイヤのダイアログで「陰影図」を選択します。なお、Z係数は "0.00001" としています。また、ややきれいにしたいので、リサンプリングに「キュービック」を使いました。

![プロパティで陰影図の設定をしているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0059/06-qgis-hillshadedialog.png)

陰影図はQGIS側で作っています。サーバには陰影図作成の負荷がかかっていません。

富士山を見てみましょう。

![富士山の陰影図](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0059/07-qgis-hillshade-fuji.png)

ほら！ほら！富士山が見えました。

![青影山の陰影図](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0059/08-qgis-hillshade-aokage.png)

そして、西にめちゃくちゃずらしたら青影山も見えました。日本中どこでも陰影図が見ることができるのです。

# おわりに

いかがだったでしょうか？

前半では、基盤地図情報DEMを fgddem2pgsql でPostGISラスタに変換して、GDALを使ってGeoTIFFに変換して、さらにCOGを生成しました。

また、後半では、QGISで確認しました。陰影図の負担はクライアント側にかかっているのと、シームレスになっているのがいいですね。

好き勝手に使って下さい。

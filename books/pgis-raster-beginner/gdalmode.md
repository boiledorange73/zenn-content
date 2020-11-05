---
title: "modeという接続文字列"
---
# はじめに

[指定位置の標高を得るクエリとインデックス](pickvalue)で、GDALのデータソースは、ファイルパスだけでなく、接続文字列で指定でいることを示しました。肝心のPostGISラスタの接続文字列は https://gdal.org/drivers/raster/postgisraster.html で紹介されています。

```
PG:"[host=''] [port:''] dbname='' [user=''] [password=''] [schema=''] [table=''] [column=''] [where=''] [mode=''] [outdb_resolution='']"
```

``host``, ``port``, ``dbname``, ``user``, ``password`` はデータベースにログインするのに必要な情報です。
``schema``, ``table``, ``column`` でカラムまでを指定するのに使われる情報なのは分かるかと思います。
``outdb_resolution``は、データベース外ラスタの話ですので、ここでは説明しません。

で、本題。``mode``ってなにさ？

あと、``where``は、たぶんフィルタリングの条件式を指定するんだろうと思うのですが、これも説明します。

## 複数のラスタを持っていると仮定します

基盤地図情報サイト https://www.gsi.go.jp/kiban/ から10mメッシュでBタイプのDEMを4個持ってきました。具体的には、``FG-GML-5133-42-dem10b-20161001.xml``, ``FG-GML-5133-43-dem10b-20161001.xml``, ``FG-GML-5133-52-dem10b-20161001.xml``, ``FG-GML-5133-53-dem10b-20161001.xml`` です。

## 変換して格納して下さい

基盤地図情報のDEMからPostGISラスタへの変換は、私は https://github.com/boiledorange73/fgddem2pgsql というものを使いました。

```bash
$ git clone https://github.com/boiledorange73/fgddem2pgsql
...
$ cd fgddem2pgsql
$ aclocal
$ autoconf
$ ./configure --with-expat=/usr/local -prefix=(path)
$ make
...
$ make install
...
```

``fgddem2pgsql``は、次のようにして動かします。

```bash
$ fgddem2pgsql -p -I dem10 FG-GML-5133-42-dem10b-20161001.xml > table.sql

$ for xml in *.xml
do
  sql=`echo $xml|sed 's/\\.xml$/.sql/'`
  echo "$sql"
  fgddem2pgsql -a dem10 "$xml" > "$sql"
  echo $sql
done
```

まず``table.sql``を、続いて他の``.sql``ファイルを実行していきます。

```
psql -d raster -f table.sql
psql -d raster -f FG-GML-5133-42-dem10b-20161001.sql
psql -d raster -f FG-GML-5133-43-dem10b-20161001.sql
psql -d raster -f FG-GML-5133-52-dem10b-20161001.sql
psql -d raster -f FG-GML-5133-53-dem10b-20161001.sql
```

# modeには1と2がある

``mode=1``は、ONE_RASTER_PER_ROW で、タプルごとに一つのラスタである、という意味です。はテーブルに複数のファイルを持っているみたいなかんじになります。
``mode=2``は、ONE_RASTER_PER_TABLE で、タプルごとにあるラスタをまとめて、テーブルで一つのラスタを形成する、という意味です。


# mode=1 とする

ではさっそ``gdalinfo``で様子を見てみましょう。

```
% gdalinfo "PG:dbname=raster table=dem10 mode=1 user=postgres password=****"
Warning 1: Cannot find (valid) information about public.dem10 table in raster_columns view. The raster table load would take a lot of time. Please, execute AddRasterConstraints PostGIS function to register this table as raster table in raster_columns view. This will save a lot of time.
Driver: PostGISRaster/PostGIS Raster driver
Files: none associated
Size is 0, 0
Subdatasets:
  SUBDATASET_1_NAME=PG:dbname=raster user=postgres password=****  schema='public' table='dem10' column='rast' where='"rid" = 1'
  SUBDATASET_1_DESC=PostGIS Raster at public.dem10 (rast), with "rid" = 1
  SUBDATASET_2_NAME=PG:dbname=raster user=postgres password=****  schema='public' table='dem10' column='rast' where='"rid" = 2'
  SUBDATASET_2_DESC=PostGIS Raster at public.dem10 (rast), with "rid" = 2
  SUBDATASET_3_NAME=PG:dbname=raster user=postgres password=****  schema='public' table='dem10' column='rast' where='"rid" = 3'
  SUBDATASET_3_DESC=PostGIS Raster at public.dem10 (rast), with "rid" = 3
  SUBDATASET_4_NAME=PG:dbname=raster user=postgres password=****  schema='public' table='dem10' column='rast' where='"rid" = 4'
  SUBDATASET_4_DESC=PostGIS Raster at public.dem10 (rast), with "rid" = 4
```

警告は無視して下さい。

``Subdatasets:``で、ラスタデータが4個あると言っています。それぞれ全く別ものと認識しているので、個々のラスタデータの内容は表示されません。

ただ、これを見ると、個別のラスタは``where``で絞るといいみたいなことに気づきます。

その通りやってみます。

```
% gdalinfo "PG:dbname=raster table=dem10 mode=1 where='rid=1' user=postgres password=****"
Driver: PostGISRaster/PostGIS Raster driver
Files: none associated
Size is 1125, 750
Origin = (133.250000000000000,34.416666667000001)
Pixel Size = (0.000111111111111,-0.000111111112000)
Corner Coordinates:
Upper Left  ( 133.2500000,  34.4166667)
Lower Left  ( 133.2500000,  34.3333333)
Upper Right ( 133.3750000,  34.4166667)
Lower Right ( 133.3750000,  34.3333333)
Center      ( 133.3125000,  34.3750000)
Band 1 Block=1125x750 Type=Float32, ColorInterp=Gray
  NoData Value=-9999
```

見れました。

``rid``を``2``, ``3``, ``4`` としても、それぞれの情報が出てきます。

# mode=2 とする

```
% gdalinfo "PG:dbname=raster table=dem10 mode=2 user=postgres password=****"
Warning 1: Cannot find (valid) information about public.dem10 table in raster_columns view. The raster table load would take a lot of time. Please, execute AddRasterConstraints PostGIS function to register this table as raster table in raster_columns view. This will save a lot of time.
Driver: PostGISRaster/PostGIS Raster driver
Files: none associated
Size is 2250, 1500
Origin = (133.250000000000000,34.500000000000000)
Pixel Size = (0.000111111111111,-0.000111111111333)
Corner Coordinates:
Upper Left  ( 133.2500000,  34.5000000)
Lower Left  ( 133.2500000,  34.3333333)
Upper Right ( 133.5000000,  34.5000000)
Lower Right ( 133.5000000,  34.3333333)
Center      ( 133.3750000,  34.4166667)
Band 1 Block=2048x1500 Type=Float32, ColorInterp=Gray
  NoData Value=-9999
```

警告は無視します。

``Size``フィールドに注目を。これ、縦横が倍になってますね。4個のラスタデータが1個に認識されています。

# QGISで見てみよう

## 「PostGISレイヤを追加」から追加すると問題あり

![PostGISレイヤを追加から追加した実行例](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/gdalmode/1.png?raw=true)

1個分しか出てないんじゃないか。


## 接続文字列で開いてみる

[PostGISラスタにGeoTIFFデータを格納する](raster2pgsql)で、GDALでは、データソースはファイルパスだけでなく、接続文字列にも対応していることを説明しました。また、QGISで接続文字列が使えることも示しました。

接続文字列を使えるなら、``mode=2``で4個のラスタを1個ととらえるようにすると、4個全部表示できるかも知れません。

やってみましょう。

「ラスタレイヤを追加」で、データソースを``PG:dbname=raster table=dem10 mode=2 user=**** password=****`` と指定します。

![「ラスタレイヤを追加」でデータソースを指定したところ](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/gdalmode/2.png?raw=true)

QGISでは次のように表示されました。

![GDAL接続文字列で開いた実行例](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/gdalmode/3.png?raw=true)

上記より多く表示されているので、``mode=2``が反映されたことが確認できました。

# おわりに

GDALの接続文字列のうち``mode``を中心に紹介しました。また、``where``も若干説明しました。

``mode``によって``gdalinfo``等のGDALツールでのPostGISラスタの見え方が違ってくる点と、QGISでも見え方が違う点について、分かって頂けたと思います。

# 出典

本記事では、国土地理院が発行する[基盤地図情報](https://www.gsi.go.jp/kiban/) を使用しました。

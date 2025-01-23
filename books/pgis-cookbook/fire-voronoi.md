---
title: "ボロノイ分割でおおざっぱな消防署の管轄範囲を作ろう"
---
# はじめに

ボロノイ分割を使って、おおざっぱな消防署の管轄範囲を作ってみます。

ボロノイ分割は、いくつかサイト（ポイント）がある状況で、最も近くにあるサイトで領域を分割するものです。ボロノイ分割で分割された領域は、その領域内のどこから見ても、その領域の中にあるサイトが最も近いサイトとなります。
https://ja.wikipedia.org/wiki/%E3%83%9C%E3%83%AD%E3%83%8E%E3%82%A4%E5%9B%B3 などを参照してみて下さい。

今回は、消防署のポイントデータをサイトとしたボロノイ分割を行って、管轄範囲を作ってみましょう。

ただし、地形や道路などを考慮すべきところ、何もかも無視して、サイトからの直線距離だけで分割を行っているので、本当の管轄範囲とは合致しないものと考えて下さい。

# 必要なデータ

消防署データは、国土交通省国土政策局が提供する「国土数値情報 (消防署)」を使います。

  * [国土数値情報のダウンロード](dl-ksj-ua)

# PGDUMPデータ作成とインポート

[PostGISを入れるところからやってみよう](../../b1de0a18073af70946e0) / [各種データを ogr2ogr でインポートしてみよう](../../b1de0a18073af70946e0/viewer/import-ogr2ogr) を参照して下さい。

国土数値情報 (消防署) には、消防署 (ポイント)と管轄範囲 (ポリゴン)があります。

管轄範囲を作ると言ってるのに、国土数値情報に既にあるんだから作る必要がないんじゃないのかと思った人には、思ったら負けです、と言いたいです。

## 消防署 (ポイント)

なお、``P17-12_34_FireStation.shp`` の座標参照系の情報が同梱されていなかったので、``EPSG:6668`` と指定しました。

座標参照系については [PostGIS入門](../../caea8d4c77dbba2e23a0) / [空間参照系の概要](../../caea8d4c77dbba2e23a0/viewer/srs) などをご覧下さい。

```batch
ogr2ogr -f PGDUMP ^
  --config PG_USE_COPY YES ^
  -lco SPATIAL_INDEX=GIST ^
  -lco GEOMETRY_NAME=geom ^
  -oo ENCODING=CP932 ^
  -nln firestation6668 ^
  -a_srs EPSG:6668 ^
  firestation6668.sql P17-12_34_FireStation.shp

set PGCLIENTENCODING=UTF8
psql -d db -f firestation6668.sql
set PGCLIENTENCODING=
```

## geometry_columns で確認

ジオメトリタイプと座標参照系(空間参照系とも)を確認しておきましょう。``geometry_columns``というビューで確認できます。

座標参照系については [PostGIS入門](../../caea8d4c77dbba2e23a0) / [空間参照系の概要](../../caea8d4c77dbba2e23a0/viewer/srs) などをご覧下さい。


```
db=# select * from geometry_columns;
 f_table_catalog | f_table_schema |       f_table_name       | f_geometry_column | coord_dimension | srid |    type
-----------------+----------------+--------------------------+-------------------+-----------------+------+------------
...
db              | public         | firestation6668          | geom              |               2 | 6668 | POINT
...
```

EPSG:6668 (JGD 2011 地理座標系)のラインストリングであることが確認できました。

# QGISで見てみる

まず、``firestation6668``テーブル (消防署)を見てみましょう。背景には地理院地図の白地図を使ってみましょう。

[PostGISを入れるところからやってみよう](../../b1de0a18073af70946e0) / [WMTSで地域を絞り込んだうえでPostGISレイヤを表示する](../../b1de0a18073af70946e0/viewer/qgis-wmts) を参照して、追加してみて下さい。

![QGISでfirestation6668の地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/fire-voronoi/01-firestations.png)

データはやや古くて、たとえば、因島消防署が土生町に本署、重井町に出張所があるようになっていますが、今は本署は中庄町に移動して、重井の出張所は廃止されています。でも**国土数値情報 (消防署)のデータのまま**で行きたいと思います。

# UTM座標系でボロノイ分割を行う

## UTM座標系の消防署データを作る

地理座標系から投影座標系に変換してからボロノイ分割を行います。広島県は UTM 53 に属しています。JGD 2011 の UTM 53 の SRID は **EPSG:6690** です。

今回はテーブルを作って、インデックスを作って、そこに貼りました。

まず、テーブルとインデックスを作ります。

```
CREATE TABLE firestation6690 (
  gid SERIAL PRIMARY KEY,
  p17_001 TEXT,
  p17_002 TEXT,
  p17_003 INT,
  p17_004 TEXT,
  geom GEOMETRY(POINT, 6690)
);
CREATE INDEX ON firestation6690 USING GiST (geom);
```

次いで、そのままコピーする勢いの ``INSERT`` を行いますが、``geom`` だけ、座標系変換を行っています。

```
INSERT INTO firestation6690(p17_001, p17_002, p17_003, p17_004, geom)
SELECT p17_001, p17_002, p17_003, p17_004, ST_Transform(geom, 6690) AS geom
FROM firestation6668;
```

## ボロノイ分割を行う

次にボロノイ領域 (ボロノイ分割の結果のポリゴン) を入れるためのテーブルを作ります。

ここで ``p17_001`` などの**属性に該当するカラムも作っている**ことに注意して下さい。

```
CREATE TABLE firestation_voronoi6690 (
  gid SERIAL PRIMARY KEY,
  p17_001 TEXT,
  p17_002 TEXT,
  p17_003 INT,
  p17_004 TEXT,
  geom GEOMETRY(POLYGON, 6690)
);
CREATE INDEX ON firestation_voronoi6690 USING GiST (geom);
```

ボロノイ領域の生成は ``ST_VoronoiPolygons()`` (https://postgis.net/docs/ja/ST_VoronoiPolygons.html) で行いますが、引数はマルチポイントです。マルチポイントを作るには、ポイントを集計関数 ``ST_Collect(geom)`` でまとめます。

同時に実行するには ``ST_VoronoiPolygons(ST_Collect(geom))`` とします。

```
INSERT INTO firestation_voronoi6690(geom)
SELECT (ST_Dump(ST_VoronoiPolygons(ST_Collect(geom)))).geom
FROM firestation6690;
```

## QGISで確認しておく

![QGISでEPSG:6690で消防署とボロノイ分割の実行結果を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/fire-voronoi/02-voronoi_6690.png)

これはいいながめですね。

## 属性データを入れる

ボロノイ領域 (ポリゴン) には属性データが入っていませんので、入れてみます。

さきほどの消防署ポイントデータとボロノイ領域とを見ると、ポリゴンの中にポイントは1個しかありません。そのまま ``UPDATE`` で、属性をポイントからポリゴンにコピーしてみましょう。

カラム値を引き出すサブクエリでは、``ST_Contains()`` を使っています。これらのような、二つのジオメトリの空間的な関係を見る関数は「テスト関数」と言っています。テスト関数については、[PostGIS入門](../../caea8d4c77dbba2e23a0) / [PostGISの空間関係をテストする関数たち](../../caea8d4c77dbba2e23a0/viewer/testing) を参照して下さい。

```
UPDATE firestation_voronoi6690
SET
  p17_001 = (
    SELECT p17_001 FROM firestation6690
    WHERE ST_Contains(firestation_voronoi6690.geom, firestation6690.geom)
    LIMIT 1
  ),
  p17_002 = (
    SELECT p17_002 FROM firestation6690
    WHERE ST_Contains(firestation_voronoi6690.geom, firestation6690.geom)
    LIMIT 1
  ),
  p17_003 = (
    SELECT p17_003 FROM firestation6690
    WHERE ST_Contains(firestation_voronoi6690.geom, firestation6690.geom)
    LIMIT 1
  ),
  p17_004 = (
    SELECT p17_004 FROM firestation6690
    WHERE ST_Contains(firestation_voronoi6690.geom, firestation6690.geom)
    LIMIT 1
  )
;
```

これで、ポリゴンをクリックすると、属性データを見ることができます。

![QGISでポリゴンに属性が付いたことを確認しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/fire-voronoi/03-voronio_attr.png)


## 管轄区域を見てみる

国土数値情報 (消防署) には、管轄区域ポリゴンもあるので、これも QGIS で重ねてみましょう。

![QGISでEPSG:6690で消防署とボロノイ分割の実行結果と管轄区域を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/fire-voronoi/04-voronoi_jurisdiction_6690.png)

これを見てると、もしかして消防署の設置の際に、県北（けんほく）は特に、地図上のボロノイ分割的な何かを考えて配置したのかもと思ってしまいました。

# 地理座標系によるボロノイ分割

ここまで UTM ベースでボロノイ分割を行いました。

最後に、ためしに地理座標系によるボロノイ分割を行ってみます。ただし、本来は**地理座標系では行うべきではない**ものです。

```
CREATE TABLE firestation_voronoi6668 (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(POLYGON, 6668)
);
CREATE INDEX ON firestation_voronoi6668 USING GiST (geom);

INSERT INTO firestation_voronoi6668(geom)
SELECT (ST_Dump(ST_VoronoiPolygons(ST_Collect(geom)))).geom
FROM firestation6668;
```

``ST_VoronoiPolygons()`` にジオグラフィでなくジオメトリを渡していますが、この関数は、ジオグラフィを引数に取ることができず、経度緯度の座標値のままでボロノイ分割を行います。

ちょうど**正距円筒図法**の地図上でボロノイ分割を行うのと同じになります。正距円筒図法は経線（南北）方向に押しつぶされているので、おそらくUTMや平面直角に投影すると、経線（南北）方向に伸びることになるはずです。

QGIS で重ね合わせてみましょう。

![QGISでEPSG:6690とEPSG:6668でボロノイ分割を行った場合の比較](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/fire-voronoi/05-voronoi_6668.png)

境界線が赤になってるのが UTM、青なのが地理座標系です。

青い境界線の方が、やや南北方向に伸びていると思いますので、予想どおりでした。

# おわりに

いかがだったでしょうか。国土数値情報 (消防署) で、UTM座標系に変換したうえで、ボロノイ分割を行いました。

ボロノイ分割で、ざっくりした管轄区域ができそうなのが見れたと思います。ただ、地図上の2次元の距離だけで分割していて、地形も道路も何も見てないので、あくまで「ざっくりした」ものしかできません。ただ、思ったより管轄区域とボロノイ分割が合っているように見えました。

それと、ボロノイ分割の関数に対しては、地理座標系のジオメトリをそのままでは使わずに、必ず適切な投影座標系を使用して下さい。

# 参照

* https://postgis.net/docs/ja/ST_VoronoiPolygons.html
* [PostGISを入れるところからやってみよう](../../b1de0a18073af70946e0) / [各種データを ogr2ogr でインポートしてみよう](../../b1de0a18073af70946e0/viewer/import-ogr2ogr)
* [PostGIS入門](../../caea8d4c77dbba2e23a0) / [空間参照系の概要](../../caea8d4c77dbba2e23a0/viewer/srs)
* [PostGISを入れるところからやってみよう](../../b1de0a18073af70946e0) / [WMTSで地域を絞り込んだうえでPostGISレイヤを表示する](../../b1de0a18073af70946e0/viewer/qgis-wmts)
* [PostGIS入門](../../caea8d4c77dbba2e23a0) / [PostGISの空間関係をテストする関数たち](../../caea8d4c77dbba2e23a0/viewer/testing)

# 出典

本章の作成にあたり、次のデータを使用しました。

* 「国土数値情報 (消防署)」 (国土交通省) https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-A09.html (2023年12月19日取得)
* 「地理院タイル」 (国土地理院) https://maps.gsi.go.jp/development/ichiran.html (2023年12月26日取得)

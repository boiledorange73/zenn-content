---
title: "広島県内の「非武装地帯」を設定する"
---
# はじめに

PostGIS公式サイトの「PostGIS 入門」の「ジオメトリ構築演習」という章 ( https://postgis.net/workshops/ja/postgis-intro/geometry_returning_exercises.html ) の中で、

>> ‘Park Slope’ 街区と ‘Carroll Gardens’ 街区とが戦争を起こしました!

唐突過ぎる！

世界全体を見回すと戦争が依然として行われていて、そのたび人の生命、尊厳や財産等をすりつぶしていて暗澹たる気分になるのですが、現実に起こりえなくてみんな困らないものは好きです。

ということで、Park Slope と Garroll Gardens との例にならって、広島県内の市区町に非武装地帯を置いてみましょう。

# 必要なデータ

「国土数値情報 (行政区域)」を使います。広島県だけを使います。テーブル名は ``mncpl_hiroshima_6671`` とし、ジオメトリの座標参照系は ``EPSG:6671`` (JGD 2011 平面直角座標系 3系) にします。

  * [国土数値情報のダウンロード](dl-ksj-ua)

# PGDUMPデータ作成とインポート

[PostGISを入れるところからやってみよう](../../b1de0a18073af70946e0)にある[各種データを ogr2ogr でインポートしてみよう](../../b1de0a18073af70946e0/viewer/import-ogr2ogr)を参照して下さい。

そのものずばりが出ています (テーブル名は変わります)。

## SQLの生成

座標参照系を、平面直角座標系 3系 (``EPSG:6671``) にしています。もちろんジオメトリでも構いませんが、今回はジオメトリで ``ST_Buffer()`` を使いたいので、平面直角座標系にします。空間参照系については、[PostGIS入門](../../caea8d4c77dbba2e23a0) の [空間参照系の概要](../../caea8d4c77dbba2e23a0/viewer/srs) をご覧下さい。

```
ogr2ogr -f PGDUMP ^
  --config PG_USE_COPY YES ^
  -lco SPATIAL_INDEX=GIST ^
  -lco GEOMETRY_NAME=geom ^
  -oo ENCODING=CP932 ^
  -t_srs EPSG:6671 ^
  -nln mncpl_hiroshima_6671 ^
  mncpl_hiroshima_6671.sql N03-23_34_230101.shp
```

## インポート

```
set PGCLIENTENCODING=UTF8
psql -d db -f mncpl_hiroshima_6671.sql
set PGCLIENTENCODING=
```

# QGISで見てみる

まず、``mncpl_hiroshima_6671``テーブル (行政区域)を見てみましょう。

![QGISでmncpl_hiroshima_6671の地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dmz/01-hiroshima.png)

# 非武装地帯を作る

境界線から双方同じ距離だけ引き下がったところまでが非武装地帯となります。

言い換えると、双方について、相手方に同じ距離だけ進行されてしまったところが非武装地帯の境界となります。

なので、通常の膨らますバッファを作って、相互にインタセクトする部分を DMZ とします。

## バッファで広島の市区町村を膨らます

相手に進行した分を求めるには、``ST_Buffer()`` で、指定した距離分膨らませます。``ST_Buffer()``の詳細情報については https://postgis.net/docs/ja/ST_Buffer.html をご覧下さい。

クエリは次のようになります。

```
SELECT n03_001,n03_002,n03_003,n03_004,n03_007,(ST_Dump(geom)).geom AS geom
INTO buffered_hiroshima_6671
FROM (
  SELECT n03_001,n03_002,n03_003,n03_004,n03_007,ST_Multi(ST_Union(ST_Buffer(geom, 1000))) AS geom
  FROM mncpl_hiroshima_6671
  GROUP BY n03_001,n03_002,n03_003,n03_004,n03_007
) AS Q;
```

少しだけ解説をします。

まず Q の中身だけを見て下さい。先にこちらが実行されます。

```
  -- Q の中身
  SELECT n03_001,n03_002,n03_003,n03_004,n03_007,ST_Multi(ST_Union(ST_Buffer(geom, 1000))) AS geom
  FROM mncpl_hiroshima_6671
  GROUP BY n03_001,n03_002,n03_003,n03_004,n03_007
```

バッファを作って、UNIONで結合させています。この際、``ST_Multi()``で、単一ジオメトリをマルチ系ジオメトリに強制しています。

続いて、Qの外側が実行されます。

```
SELECT n03_001,n03_002,n03_003,n03_004,n03_007,(ST_Dump(geom)).geom AS geom
FROM () AS Q
```

``ST_Dump()``で単一ポリゴンにしています。

``SELECT INTO``で叩き込んでいますが、これでは、QGISに読ませようとすると、ジオメトリタイプとSRIDを指定する必要があります。

## ジオメトリタイプとSRIDを指定する

``geometry_columns`` を見ても、やはりジオメトリタイプとSRIDが指定されていません。

```
db=# SELECT * FROM geometry_columns;
 f_table_catalog | f_table_schema |       f_table_name       | f_geometry_column | coord_dimension | srid |    type
-----------------+----------------+--------------------------+-------------------+-----------------+------+------------
...
 db              | public         | buffered_hiroshima_6671  | geom              |               2 |    0 | GEOMETRY
...
```

これは、QGISでレイヤ追加しようとすると、結構まずい状況です。

![QGISでジオメトリタイプとSRIDを指定していないテーブルを追加しようとしているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dmz/02-qgis_not_specified_source.png)

このように、テーブルをレイヤとして追加しようとして、テーブルをダブルクリックしようとしても、追加できません。ジオメトリタイプとSRIDを手動で指定しないと追加できません。

これを解決するには、ジオメトリタイプとSRIDを指定します。たとえば次のクエリで、ジオメトリタイプとSRIDを強制できます。

```
ALTER TABLE buffered_hiroshima_6671 ALTER COLUMN geom TYPE GEOMETRY(POLYGON, 6671);
```

これを行った結果を示します。

```
db=# ALTER TABLE buffered_hiroshima_6671 ALTER COLUMN geom TYPE GEOMETRY(POLYGON, 6671);
ALTER TABLE
db=# SELECT * FROM geometry_columns;
 f_table_catalog | f_table_schema |       f_table_name       | f_geometry_column | coord_dimension | srid |    type
-----------------+----------------+--------------------------+-------------------+-----------------+------+------------
...
 db              | public         | buffered_hiroshima_6671  | geom              |               2 | 6671 | POLYGON
...
```

この後で、テーブルをレイヤとして追加しようとすると、ジオメトリタイプとSRIDが指定されているので、ダブルクリックするだけでレイヤの追加ができます。

![QGISでジオメトリタイプとSRIDを指定したところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dmz/03-qgis_specified_source.png)

## 膨らましたポリゴンを見てみる

では見てみましょう。

![QGISで膨らましたポリゴンを表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dmz/04-buffered.png)

ちょっと分かりにくいので、バッファのポリゴンの塗りつぶしを半透過にしてみます。

![QGISで膨らましたポリゴンを半透過で表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dmz/05-buffered-semitransparent.png)

DMZっぽいものも見えて、いい眺めになりましたね!

## DMZを計算する

バッファが相互に共有している (インタセクトしている) 部分を計算し、これを DMZ とします。

[PostGISを入れるところからやってみよう](../../caea8d4c77dbba2e23a0)にある[ジオメトリの足し算、引き算をする](../../caea8d4c77dbba2e23a0/viewer/editor)を参照して下さい。

```
SELECT ST_Intersection(Q1.geom, Q2.geom), Q1.n03_007 AS n03_007_1, Q2.n03_007 AS n03_007_2
INTO dmz_6671
FROM buffered_hiroshima_6671 AS Q1, buffered_hiroshima_6671 AS Q2
WHERE Q1.n03_007 < Q2.n03_007;
```

WHERE句で ``Q1.n03_007 < Q2.n03_007`` としている点に注意して下さい。

WHERE句で ``Q1.n03_007 <> Q2.n03_007`` とする場合には、Q1=尾道市, Q2=尾道市 の時は WHERE が FALSE になって弾かれるのですが、これだけでは足りません。

なぜなら、Q1=尾道市, Q2=福山市 の場合も、Q1=福山市, Q2=尾道市 の場合も WHERE が TRUE になってしまうため、全く同じ DMZ 候補が二重に作成されてしまいます。

``Q1.n03_007 < Q2.n03_007`` だと、Q1=尾道市, Q2=福山市 の場合は WHERE が TRUE になりますが、Q1=福山市, Q2=尾道市 の場合は WHERE が FALSE になり、二重作成を防ぐことができます。

## できた!

![QGISでDMZを表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/dmz/06-dmz.png)

すばらしい眺めですね!

でもこれって戦国乱世みたいですね。

# おわりに

いかがだったでしょうか?

ポリゴンを膨らませて、相互の関係から新たなポリゴンを作ってみました。また、``ST_Union()``や``ST_Dump()``を併用して単一ポリゴンを生成するといったこともしています。

なお、広島県内の各市町で仲が悪いということは、あまりないと思います。

# 参照

* [PostGISを入れるところからやってみよう](../../b1de0a18073af70946e0) / [各種データを ogr2ogr でインポートしてみよう](../../b1de0a18073af70946e0/viewer/import-ogr2ogr)
* [PostGIS入門](../../caea8d4c77dbba2e23a0) / [空間参照系の概要](../../caea8d4c77dbba2e23a0/viewer/srs)
* https://postgis.net/docs/ja/ST_Buffer.html
* [PostGISを入れるところからやってみよう](../../caea8d4c77dbba2e23a0) / [ジオメトリの足し算、引き算をする](../../caea8d4c77dbba2e23a0/viewer/editor)
# 出典

本章の作成にあたり、次のデータを使用しました。

* 「筆ポリゴンデータ」 (農林水産省) https://open.fude.maff.go.jp/ (2023年11月28日取得)
* 「国土数値情報 (都市地域データ)」 (国土交通省) https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-A09.html (2023年12月1日取得)


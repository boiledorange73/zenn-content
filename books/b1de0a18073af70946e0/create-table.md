---
title: "とりあえずPostGISのテーブル作ってみようか"
---
# はじめに

標題の通り、「とりあえず」、まっさらの状態からテーブルを作るところまでやってみます。

# データベース作成とエクステンション作成

シェルからなら、createdbで作成、psqlからエクステンション作成、を順次行います。

```csh
% createdb db
% psql db
```

```psql
db=# create extension postgis;
CREATE EXTENSION
db=# 
```

Windowsではコマンドプロンプトよりpsqlを呼び出した方が早そうだったので、次のようになりました。

```psql
postgres=# create database db;
CREATE DATABASE
postgres=# \c db
データベース "db" にユーザ"postgres"として接続しました。
db=# create extension postgis;
CREATE EXTENSION
db=# 
```

# テーブルを作る

``CREATE TABLE``でテーブルを作りますが、そこにジオメトリ型もつっこめます。その際に型修飾を指定します。

```sql
CREATE TABLE tbl (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(POINT, 4612)
);
```

型修飾子の第１パラメータはジオメトリタイプ、第２パラメータはSRIDです。

主キーは単一の整数カラムにしておいた方が無難です。クライアントが単一の整数カラムのみ有効というのはよく見ます。

## ジオメトリタイプ?

ジオメトリは、より細かく見ると、点、線、面といった種類に分けられます。これらをジオメトリタイプと呼んでいす。基本的にポイント、ラインストリング、ポリゴン、マルチポイント、マルチラインストリング、マルチポリゴンを押さえておけば大丈夫です（本当はもう少しあります）。

## SRID?

SRIDはSRS(Spatial Reference System, 空間参照系)のIDです。PostGISでは整数です。[空間参照系の概要](https://boiledorange73.qrunch.io/entries/WQUrnoALXaSeon9R)を参照して下さい。

# 作ったジオメトリカラムを確認する

OpenGIS Simple Feature Access では、GEOMETRY_COLUMNS というテーブルを作成し、データベースに存在するジオメトリカラムの一覧を見せることが求められています。クライアントがDBMSの種類を問わずジオメトリカラム情報を取得するためです。

```
db=# SELECT * FROM geometry_columns;
 f_table_catalog | f_table_schema | f_table_name | f_geometry_column | coord_dimension | srid | type  
-----------------+----------------+--------------+-------------------+-----------------+------+-------
 db              | public         | tbl          | geom              |               2 | 4612 | POINT
(1 行)
```

現在のPostGISは、GEOMETRY_COLUMNSはテーブルでなくビューです。よって、GEOMETRY_COLUMNSは編集できません、ていうか手動で編集するものではありません。

# もうひとつのジオメトリカラムの作り方

"OpenGIS Implementation Specification for Geographic information - Simple feature access - 1.1" (OGC 05-134)では、ジオメトリカラムの追加のためにAddGeometryColumn関数を用意するよう求められていました。PostGISには、後方互換のためか残っています。

```
db=# CREATE TABLE tbl2 (
  gid INT PRIMARY KEY
);

db=# SELECT AddGeometryColumn('tbl2', 'geom', 4612, 'point', 2);
               addgeometrycolumn               
-----------------------------------------------
 public.tbl2.geom SRID:4612 TYPE:point DIMS:2 
(1 行)


db=# SELECT * FROM geometry_columns;
 f_table_catalog | f_table_schema | f_table_name | f_geometry_column | coord_dimension | srid | type  
-----------------+----------------+--------------+-------------------+-----------------+------+-------
 db              | public         | tbl          | geom              |               2 | 4612 | POINT
 db              | public         | tbl2         | geom              |               2 | 4612 | POINT
(2 行)
```

# 空間インデックスをはる

空間インデックスは、次の通り実行することではれます。

```sql
CREATE INDEX ix_tbl_geom ON tbl USING GiST (geom);
```

一般に、幾何データのインデックスを作成しようとすると、R木等を実装しないといけませんが、PostGISはGiSTに乗っかってR木のようなものを作っています。このため、"USING GiST"が必要となっています。


# データを挿入する
## SQL/MM標準を使う

ジオメトリを生成したら、それをいたって普通に``INSERT``を使って挿入できます。

```
db=# INSERT INTO tbl (gid, geom) SELECT 1, ST_GeomFromText('POINT(135 35)', 4612);
INSERT 0 1
```

SRIDが異なるとエラーが出ます。

```
db=# INSERT INTO tbl (gid, geom) SELECT 2, ST_GeomFromText('POINT(135 35)', 4326);
ERROR:  Geometry SRID (4326) does not match column SRID (4612)
```

ジオメトリカラムの型修飾子で指定されているジオメトリタイプと異なるジオメトリタイプを挿入しようとするとエラーが出ます。

```
db=# INSERT INTO tbl (gid, geom) SELECT 2, ST_GeomFromText('LINESTRING(135 35, 135.1 35)', 4612);
ERROR:  Geometry type (LineString) does not match column type (Point)
```

## EWKTを使ってCOPY

ST_GeomFromTextは、COPYでは使えません。

```psql
db=# COPY tbl (gid, geom) FROM STDIN;
3	ST_GeomFromText('POINT(135.1 35)', 4612)
\.
ERROR:  parse error - invalid geometry
HINT:  "ST" <-- parse error at position 2 within geometry
CONTEXT:  COPY tbl, line 1, column geom: "ST_GeomFromText('POINT(135.1 35)', 4612)"
```

ST_GeomFromTextでなくWKTを直接書き込むとどうでしょう？

```psql
db=# COPY tbl (gid, geom) FROM STDIN;
3	POINT(135.1 35)
\.
ERROR:  Geometry SRID (0) does not match column SRID (4326)
db=#
```

WKT文字列をジオメトリにキャストしてくれるところまではいったのですが、WKTをジオメトリにキャストすると、生成されたジオメトリのSRIDは0となります。SRIDが異なるためにエラーが出ました。

次のように書いてみましょう。

```
db=# COPY tbl (gid, geom) FROM STDIN;
3	SRID=4612;POINT(135.1 35)
\.
```

WKTに``SRID=4326;``を追加していますが、それ以外は変えていません。これでCOPYが成功します。

## EWKT

PostGISでは、``SRID=4326;POINT(0 0)``といった、WKTを拡張した書式を受け付けます。PostGISの世界では、EWKT (Extended Well-Known Text) と呼ばれています。

とりあえずは、SRIDを先頭に付けた書式、と見て下さい。

```psql
db=# SELECT ST_SRID('SRID=4326;POINT(0 0)'::GEOMETRY);
 st_srid
---------
    4326
(1 行)
```

上の例は、EWKT文字列をジオメトリにキャストした結果生成されたジオメトリのSRIDを見ていますが、ご覧の通り、4326になっています。これで、ST_GeomFromTextを使わずとも、SRIDを指定したWKTが書けるようになります。

# おわりに

ジオメトリカラムの作成方法2種類、ジオメトリカラムの確認方法、インデックスのはりかた、データ挿入の方法を示しました。

typemodという言葉に慣れていないとしても、varchar(10)といったものと同じなので、あまり深く考えなくていいかと思います。
インデックスも"USING GiST"を付ければOKです。こういった、特に何も考えずに便利な機能を利用可能にしてくれるのはPostgreSQL/PostGISの良いところですね。

EWKTという言葉が出てきましたが、とりあえずSRIDを挿入できるWKT、と考えて差し支えありません。

# 余談: typemodのコードについて

システムカタログを探って、tbl.geomについて、typemodのコードを見てみます。

```
db=# SELECT pg_attribute.atttypmod
FROM pg_attribute
  INNER JOIN pg_class ON pg_class.oid=pg_attribute.attrelid
WHERE pg_class.relname='tbl' AND pg_attribute.attname='geom';
 atttypmod 
-----------
         4
(1 行)
```

liblwgeom/liblwgeom.h で定義されているマクロを参照してみました。
上位ビットから見ると、次のような意味を持つようです。

|ビット|説明|
|:-----|:----|
|0|コメントには"plus/minus"と記述|
|1-2|予備|
|3-23|SRID (0-2097151)|
|24-29|タイプ (0-63)|
|30|Z座標の有無|
|31|M座標の有無|

最上位ビットについては、たぶん0にして負数にしないことを指しているように思います。

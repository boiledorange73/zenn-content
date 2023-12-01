---
title: "シェープファイルのデータをインポートしてみよう ogr2ogr編"
---

# はじめに

[GDAL/OGR](https://gdal.org/index.html)は、ラスタデータの相互変換ライブラリとCUIツール(GDAL)と、ベクタデータの相互変換ライブラリとCUIツール(OGR)とからなるパッケージです。

今回は、ラスタデータ、ベクタデータに関する紹介はしません。

シェープファイルからPostGIS用データに変換するためのツールとして、GDAL/OGRの``ogr2ogr``を使用します。

## なぜ ogr2ogr ?

あえて``shp2pgsql``を使わずに``ogr2ogr``を使うのは次の理由があるからです。

  * QGISに必ずバンドルされるから
  * OGRは様々なデータに対応しているので、様々なフォーマットのデータをインポートできるから

## 正確にはインポートでなく SQL を生成

OGRが対応するフォーマットに``PGDUMP``と``PostgreSQL``があります。``PGDUMP``はPostGIS用のSQLを吐かせるもので、``PostgreSQL``はPostgreSQLに接続します。

今回は、``PGDUMP``を使用してPostGIS用のSQLを吐かせます。

# PostGIS用データに変換するための基本

## 基本オプション

```
ogr2ogr -f PGDUMP ^
  --config PG_USE_COPY YES ^
  -lco SPATIAL_INDEX=GIST ^
  -lco GEOMETRY_NAME=geom ^
  -oo ENCODING=(文字コード) ^
  -nln (テーブル名) ^
  (適当な出力ファイル名) (入力ファイル名1) (入力ファイル名2) ...
```

*出力ファイル名と入力ファイル名とが逆です*。こうしないと複数の入力ファイルをまとめて単一の出力ファイルに出力することができないので、仕方ないです。

オプションの意味は次の通りです。

  * ``--config PG_USE_COPY YES`` - COPYコマンドを使ったSQLを履いてくれます。こうしないとINSERTコマンドを使うため、インポートが遅くなります。
  * ``-nln (テーブル名)`` - インポート先のテーブル名を指定します。デフォルトでは入力ファイル名のプレフィクスを使ってしまうので、基本的には、この指定は必要とお考え下さい。
  * ``-lco SPATIAL_INDEX=GIST`` - ジオメトリカラムに対してGiSTインデックスを作ります。基本的に空間インデックスは作っておいた方がいいです。
  * ``-lco GEOMETRY_NAME=geom`` - ジオメトリカラム名を geom にします。デフォルトではwkb_geometryとなります。この指定は必須でもありませんが、ジオメトリカラム名はご自分で統一した名前を使っておいたほうが絶対にいいです。
  * ``-oo ENCODING=(文字コード)`` - 入力データの文字コードを指定します。日本語圏では``SJIS``(Shift JIS)または一部独自拡張した``CP932``が多く使われます。

## 国土数値情報(行政区域)を変換する

たとえば、国土数値情報(行政区域)の N03-23_34_230101.shp をテーブル名 ``t3`` に格納するSQLに変換する場合は、次のようにします。

```batch
ogr2ogr -f PGDUMP ^
  --config PG_USE_COPY YES ^
  -lco SPATIAL_INDEX=GIST ^
  -lco GEOMETRY_NAME=geom ^
  -oo ENCODING=CP932 ^
  -nln t3 ^
  t3.sql N03-23_34_230101.shp
```

## インポート

次のコマンドでOKです。

```sh
% psql -d db -f t3.sql
```

## Windowsでインポートする場合の注意

Windowsのコマンドプロンプトや、それを元にしている OSGeo4W Shell などでは、CP932をデフォルトの文字コードと認識しているため、UTF8のファイルを読ませようとすると、非ASCIIがあると、エラーが発生します。

回避するには、環境変数``PGCLIENTENCODING``を設定します。

```batch
> set PGCLIENTENCODING=UTF8
> psql -d db -f t3.sql
```

# どんな感じのSQLが作られているか

どんなかんじのSQLが作られているか、先頭から少し見てみましょう。

```sql
SET standard_conforming_strings = ON;
DROP TABLE IF EXISTS "public"."t3" CASCADE;
BEGIN;
CREATE TABLE "public"."t3"();
ALTER TABLE "public"."t3" ADD COLUMN "ogc_fid" SERIAL CONSTRAINT "t3_pk" PRIMARY KEY;
SELECT AddGeometryColumn('public','t3','geom',6668,'POLYGON',2);
COMMENT ON TABLE "public"."t3" IS NULL;
ALTER TABLE "public"."t3" ADD COLUMN "objectid" FLOAT8;
ALTER TABLE "public"."t3" ADD COLUMN "n03_001" VARCHAR(10);
ALTER TABLE "public"."t3" ADD COLUMN "n03_002" VARCHAR(254);
ALTER TABLE "public"."t3" ADD COLUMN "n03_003" VARCHAR(254);
ALTER TABLE "public"."t3" ADD COLUMN "n03_004" VARCHAR(254);
ALTER TABLE "public"."t3" ADD COLUMN "n03_007" VARCHAR(5);
ALTER TABLE "public"."t3" ADD COLUMN "shape_leng" NUMERIC(23,15);
ALTER TABLE "public"."t3" ADD COLUMN "shape_area" NUMERIC(23,15);
COPY "public"."t3" ("geom", "objectid", "n03_001", "n03_002", "n03_003", "n03_004", "n03_007", "shape_leng", "shape_area") FROM STDIN;
01...40	1	広島県	\N	広島市	広島市中区	34101	0.185224800844014	0.001501901176319
...
```

普通なら ``CREAE TABLE``ステートメントで一気にカラム定義を入れていきますが、カラムが全くないテーブルを作ってから、後で、``AddGeometryColumn()``でジオメトリカラムを追加し、``ALTER TABLE``で他のカラムを追加していっています。

確かに間違いではないですが、**こんなの初めて見ました**。

``COPY``以下は[シェープファイルのデータをインポートしてみよう コマンドライン編](import-cli)で見たのとだいたい同じです。ただし、ジオメトリカラムがカラム列の先頭に来るか(ogr2ogr)、末尾に来るか(shp2pgsql)の違いがあります。

# もう少し機能追加した変換

## 入力ファイルの座標系から別の座標系に変換して出力したい場合

入力ファイルの空間参照系から別の空間参照系に変換したい場合には、``ogr2ogr`` に次のオプションを付けます。

  * ``-t_srs (座標系定義)`` - 入力ファイルの座標系からこのオプションで指定した座標系に変換計算を行ったうえで出力します。

たとえば、入力ファイル ``N03-23_34_230101.shp`` のジオメトリを全て JGD2011 平面直角座標系3系 に変換したうえで、PGDUMP形式で、テーブル名を t3_6672 として ``t3_6672.sql`` に出力するには、次のようにします。

```batch
ogr2ogr -f PGDUMP ^
  --config PG_USE_COPY YES ^
  -lco SPATIAL_INDEX=GIST ^
  -lco GEOMETRY_NAME=geom ^
  -oo ENCODING=CP932 ^
  -nln t3_6672 ^
  -t_srs EPSG:6672 ^
  t3_6672.sql N03-23_34_230101.shp
```


## 座標参照系の差し替えだけをしたい場合

入力ファイルの空間参照系から別の空間参照系に差し替えたいけど、計算時間をかけたくないので座標値変換計算をしたくない場合には、``ogr2ogr`` に次のオプションを付けます。

  * ``-a_srs (座標系定義)`` - 入力ファイルの座標値を変更せずに座標系を、指定した座標系に差し替えて出力します。

たとえば、入力ファイル ``N03-23_34_230101.shp`` の空間参照系は EPSG:6668 (JGD2011 経度緯度) で、EPSG:4326 (WGS84 経度緯度)に差し替えても差し支えないと判断されるとします。EPSG:4326に差し替えたうえで、PGDUMP形式で、テーブル名を t3_4326 として ``t3_4326.sql`` に出力するには、次のようにします。

```batch
ogr2ogr -f PGDUMP ^
  --config PG_USE_COPY YES ^
  -lco SPATIAL_INDEX=GIST ^
  -lco GEOMETRY_NAME=geom ^
  -oo ENCODING=CP932 ^
  -nln t3_4326 ^
  -a_srs EPSG:4326 ^
  t3_4326.sql N03-23_34_230101.shp
```

# 複数ファイルがある場合

[シェープファイルのデータをインポートしてみよう コマンドライン編](import-cli)で、``shp2pgsql``にはテーブル作成とデータ作成とを分離できると書いていました。

データのみ作成する場合は、次のオプションが使えます。

  * ``-lco CREATE_TABLE=OFF`` - テーブルを生成しません。

ただ、テーブルのみを作成する方法がないようです。

入力ファイルが複数していできるので、そちらを使うといいと思います。

# おわりに

ここでは ``ogr2ogr``を使った場合の SQL ファイルの作り方を紹介しました。

``shp2pgsql``をお持ちでない場合に使ってみて下さい。

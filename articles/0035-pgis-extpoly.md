---
title: "ST_Buffer()を使ってジオメトリコレクションからポリゴン要素だけを抜き出す"
emoji: "😀"
type: "tech"
topics: [PostGIS, 小ネタ]
published: true
---
# はじめに

``ST_Buffer(geom, 0.0)``とすると、なぜかポリゴンだけが残るというネタです。

ジオメトリコレクションに対するバッファは、要素ごとにバッファを施したジオメトリを生成して、妥当なジオメトリだけ残すようにフィルタリングして、結果をまとめて返す、というようなことをしていると思われます。
0.0でバッファを施すと、ラインストリング要素とポイント要素は面積がゼロになるポリゴンになり、これらは不正ジオメトリとなります。
フィルタリングで消されるのだと思います。

まずはジオメトリコレクションを用意します。

```sql
db=# WITH t1 AS (
  SELECT ST_Collect(ARRAY[
    'POINT(1 1)'::GEOMETRY,
    'LINESTRING(1 1, 2 2)'::GEOMETRY,
    'POLYGON((1 1, 2 2, 2 1, 1 1))'::GEOMETRY
  ]) AS geom
)
SELECT ST_AsText(geom) FROM t1;

                                   st_astext                                   
-------------------------------------------------------------------------------
 GEOMETRYCOLLECTION(POINT(1 1),LINESTRING(1 1,2 2),POLYGON((1 1,2 2,2 1,1 1)))
(1 行)
```

ここからポリゴンだけを出してみましょう。

```sql
db=# WITH t1 AS (
  SELECT ST_Collect(ARRAY[
    'POINT(1 1)'::GEOMETRY,
    'LINESTRING(1 1, 2 2)'::GEOMETRY,
    'POLYGON((1 1, 2 2, 2 1, 1 1))'::GEOMETRY
  ]) AS geom
)
SELECT ST_AsText(
  ST_Buffer(geom, 0.0)
) FROM t1;

         st_astext          
----------------------------
 POLYGON((1 1,2 2,2 1,1 1))
(1 行)
```

複数個のポリゴンがあるとマルチポリゴンになります。

```sql
db=# WITH t1 AS (
  SELECT ST_Collect(ARRAY[
    'POINT(1 1)'::GEOMETRY,
    'LINESTRING(1 1, 2 2)'::GEOMETRY,
    'POLYGON((1 1, 2 2, 2 1, 1 1))'::GEOMETRY,
    'POLYGON((2 2, 3 3, 3 2, 2 2))'::GEOMETRY
  ]) AS geom
)
SELECT ST_AsText(
  ST_Buffer(geom, 0.0)
) FROM t1;

                       st_astext                       
-------------------------------------------------------
 MULTIPOLYGON(((1 1,2 2,2 1,1 1)),((2 2,3 3,3 2,2 2)))
(1 行)
```

マルチポリゴンが混じっている場合には、シングルポリゴンにダンプしたあと全部でマルチポリゴンにまとめられたかんじになります。

```sql
WITH t1 AS (
  SELECT ST_Collect(ARRAY[
    'POINT(1 1)'::GEOMETRY,
    'LINESTRING(1 1, 2 2)'::GEOMETRY,
    'POLYGON((1 1, 2 2, 2 1, 1 1))'::GEOMETRY,
    'MULTIPOLYGON(((2 2, 3 3, 3 2, 2 2)),((3 3, 4 4, 4 3, 3 3)))'::GEOMETRY
  ]) AS geom
)
SELECT ST_AsText(
  ST_Buffer(geom, 0.0)
) FROM t1;

                                 st_astext                                 
---------------------------------------------------------------------------
 MULTIPOLYGON(((1 1,2 2,2 1,1 1)),((2 2,3 3,3 2,2 2)),((3 3,4 4,4 3,3 3)))
(1 行)
```

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

---
title: "ジオメトリコレクションは嫌い"
---
# はじめに

ジオメトリコレクション (GEOMETRYCOLLECTION) は、マルチ系ジオメトリに近いですが、通常のマルチ系ジオメトリは、対応するシングル系ジオメトリを集めたものに対して、ジオメトリコレクションは、複数のジオメトリタイプを混ぜ込むことができます。なので、ジオメトリタイプを気にせずに、なんでも集めて一つののジオメトリにすることができます。

そういう融通が利きすぎるところが嫌いです。

だいたいGISデータを整備する際にはジオメトリタイプを統一させるものです。ジオメトリタイプごとに、デスクトップGISで表示設定をする際に指定するものが違ってきますし、ジオメトリデータの処理も違ってきますし、何もかも違ってきます。**どう扱えと**。

たいていの場合は、意図してジオメトリコレクションを生成するのではなく、関数の処理結果が思った通りにいかない場合の返り値に現れてしまうのが基本です。

でもジオメトリコレクションができてしまうのはしょうがないので、ここから特定のジオメトリタイプの要素だけを抜き出して、どうにか作業が続行できるようにすることがあります。

# なぜそこまで毛嫌いするのか

次のクエリを見てみて下さい。

```
CREATE TABLE t_geomcol_1 (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(MULTIPOLYGON)
);

WITH polys AS (
  SELECT 'POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))'::GEOMETRY AS geom
  UNION SELECT 'POLYGON((15 0, 25 0, 25 10, 15 10, 15 0))'::GEOMETRY AS geom
),
geomcol AS (
  SELECT ST_Collect(
    ST_Intersection('POLYGON((10 2, 20 2, 20 8, 10 8, 10 2))'::GEOMETRY, geom)
  ) AS geom
  FROM polys
)
INSERT INTO t_geomcol_1 (geom)
SELECT geom FROM geomcol;
```

``poly`` は、1辺10の正方形がX軸を底に二つ並んでいるデータセットです。そこに``POLYGON((10 2, 20 2, 20 8, 10 8, 10 2))``でマスクをかけて得られたポリゴンを``ST_Collect``で集めてマルチポリゴンとしてテーブル``t_geomcol_1``に挿入しようとしています。

結果は次の通りです。

```
ERROR:  Geometry type (GeometryCollection) does not match column type (MultiPolygon)
```

``INSERT``のかわりに``SELECT``で様子を見てみましょう。この際に``ST_AsEWKT()``でEWKT出力を見た方がいいです。

```
WITH polys AS (
  SELECT 'POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))'::GEOMETRY AS geom
  UNION SELECT 'POLYGON((15 0, 25 0, 25 10, 15 10, 15 0))'::GEOMETRY AS geom
),
geomcol AS (
  SELECT ST_Collect(
    ST_Intersection('POLYGON((10 2, 20 2, 20 8, 10 8, 10 2))'::GEOMETRY, geom)
  ) AS geom
  FROM polys
)
SELECT ST_AsEWKT(geom) FROM geomcol;
```

この結果は次の通りです。

```
                                   st_asewkt
-------------------------------------------------------------------------------
 GEOMETRYCOLLECTION(LINESTRING(10 8,10 2),POLYGON((20 8,20 2,15 2,15 8,20 8)))
(1 行)
```

マスク長方形と左側の正方形が境界線のみを共有しているので、ラインストリングで返ってきます。マスク長方形と右側正方形は面を共有しているので、ポリゴンが返ってきます。それがまざってるので、ジオメトリコレクションが返ってしまっています。

そして、``t_geomcol_1.geom``カラムはマルチポリゴンが入るのが期待しているのですが、ジオメトリコレクションが入ってくるので、当然、冒頭のようなエラーが出るわけです。

# ジオメトリコレクションから特定のジオメトリだけを抜き出す

```
ST_CollectionExtract(geometry collection, integer type);
```

抽出対象のタイプを第2引数の整数で指定します。指定する値は次の通りです。

* 1 - ポイント
* 2 - ラインストリング
* 3 - ポリゴン

## ではさっそくマルチポリゴンだけにする

```
WITH polys AS (
  SELECT 'POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))'::GEOMETRY AS geom
  UNION SELECT 'POLYGON((15 0, 25 0, 25 10, 15 10, 15 0))'::GEOMETRY AS geom
),
geomcol AS (
  SELECT ST_Collect(
    ST_Intersection('POLYGON((10 2, 20 2, 20 8, 10 8, 10 2))'::GEOMETRY, geom)
  ) AS geom
  FROM polys
)
SELECT ST_AsEWKT(ST_CollectionExtract(geom, 3)) FROM geomcol;
```

結果は次の通りです。

```
                 st_asewkt
--------------------------------------------
 MULTIPOLYGON(((20 8,20 2,15 2,15 8,20 8)))
(1 行)
```

ポリゴン要素数が1のマルチポリゴンが出てきました。

## シングル系ジオメトリを渡した場合

シングル系ジオメトリを渡した場合には、シングル系ジオメトリを返すので注意して下さい。

```
db=# SELECT ST_AsEWKT(
  ST_CollectionExtract('POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))'::GEOMETRY, 3)
);

             st_asewkt
------------------------------------
 POLYGON((0 0,10 0,10 10,0 10,0 0))
(1 行)
```

もしシングル系ジオメトリを渡しそうなら、どこかで``ST_Multi()``でマルチ系ポリゴンに切り替えるといいです。

## 抽出されなかった要素の扱い

``geomcol``のポリゴンを抽出するのはいいのですが、ラインストリングは抽出しませんでした。抽出しなかったジオメトリ要素はどうしたらいいでしょう？

通常は捨てます。

ジオメトリコレクションは、ジオメトリタイプが揃っていたものの一部が不用意に変わってしまった結果として生成されたものです。その不用意に変わったものは、本当に不要かを検討する必要がありますが、捨てることが多いです。

# おわりに

ジオメトリコレクションが出たら必要なジオメトリタイプだけ抽出する方法を示しました。

不用意にジオメトリコレクションが現れても、これで冷静に対応できるようになるのではないでしょうか。

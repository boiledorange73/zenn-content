---
title: "PostGISラスタのNODATA値がピクセルタイプによってどう変わるか"
emoji: "😀"
type: "tech"
topics: [PostGIS]
published: true
---
# はじめに

``ST_MapAlgebraExpr()``でピクセルタイプを指定でいますが、ピクセルタイプが変わることでNODATA値が変わる場合があります。

[ピクセル演算をやってみよう](https://zenn.dev/boiledorange73/books/pgis-raster-beginner/viewer/calcpixel)([PostGIS ラスタ 入門](https://zenn.dev/boiledorange73/books/pgis-raster-beginner/))でも、NODATA値が変わっていることが示されています。

実際にどう変わっているのか調べてみました。

# ラスタを生成する関数たち

まず、1バンドを持つラスタを作ります。

次のクエリは``ST_MakeEmptyRaster()``で空のラスタを作り、``ST_AddBand()``で一つのバンドを追加しています。

バンド追加の際に、NODATA値を9999.0にしています。

```
-- R1: 空のラスタを作る
WITH R1 AS (
  SELECT ST_MakeEmptyRaster(100,100, 135, 35, 0.01, -0.01, 0, 0, 4326) AS rast
),
-- R2: R1.rastにバンドを追加、NODATA値9999.0
R2 AS (
  SELECT ST_AddBand(rast, '64BF', 0.0, 9999.0) AS rast FROM R1
)
SELECT (ST_MetaData(rast)).*, ST_BandNoDataValue(rast) AS NODATA FROM R2;
```

結果は次のようになります。``ST_MetaData()``で、ラスタのメタデータが得られ、``ST_BandNoDataValue()``でバンド（バンド番号が指定されていない場合には1と仮定）のNODATA値が得られています。

```
 upperleftx | upperlefty | width | height | scalex | scaley | skewx | skewy | srid | numbands | nodata
------------+------------+-------+--------+--------+--------+-------+-------+------+----------+--------
        135 |         35 |   100 |    100 |   0.01 |  -0.01 |     0 |     0 | 4326 |        1 |   9999
(1 行)
```

# 8ビット符号なし整数に変換するとNODATA値はどうなるか

64ビット浮動小数点数のバンドを``ST_MapAlgebraExpr()``で8ビット符号なし整数のバンドに変換するとNODATA値どうなるでしょうか。

```
-- R1: 空のラスタを作る
WITH R1 AS (
  SELECT ST_MakeEmptyRaster(100,100, 135, 35, 0.01, -0.01, 0, 0, 4326) AS rast
),
-- R2: R1.rastにバンドを追加、NODATA値9999.0
R2 AS (
  SELECT ST_AddBand(rast, '64BF', 0.0, 9999.0) AS rast FROM R1
),
-- R3: R2.rastでST_MapAlgebraExpr()を実行
R3 AS (
  SELECT ST_MapAlgebraExpr(rast, '8BUI', '[rast]') AS rast FROM R2
)
SELECT (ST_MetaData(rast)).*, ST_BandNoDataValue(rast) AS NODATA FROM R3;
```

NODATA値を見てみると``255``でした。

```
 upperleftx | upperlefty | width | height | scalex | scaley | skewx | skewy | srid | numbands | nodata
------------+------------+-------+--------+--------+--------+-------+-------+------+----------+--------
        135 |         35 |   100 |    100 |   0.01 |  -0.01 |     0 |     0 | 4326 |        1 |    255
(1 行)
```

# NODATA値が-9999の時はどうなるか

64ビット浮動小数点数のバンドを``ST_MapAlgebraExpr()``で8ビット符号なし整数のバンドに変換するのは同じですが、NODATA値を``-9999.0``にするとどうなるでしょうか。

```
-- R1: 空のラスタを作る
WITH R1 AS (
  SELECT ST_MakeEmptyRaster(100,100, 135, 35, 0.01, -0.01, 0, 0, 4326) AS rast
),
-- R2: R1.rastにバンドを追加、NODATA値-9999.0
R2 AS (
  SELECT ST_AddBand(rast::raster, '64BF', 0.0, -9999.0) AS rast FROM R1
),
-- R3: R2.rastでST_MapAlgebraExpr()を実行
R3 AS (
  SELECT ST_MapAlgebraExpr(rast, '8BUI', '[rast]') AS rast FROM R2
)
SELECT (ST_MetaData(rast)).*, ST_BandNoDataValue(rast) AS NODATA FROM R3;
```

NODATA値は``0``になりました。

```
 upperleftx | upperlefty | width | height | scalex | scaley | skewx | skewy | srid | numbands | nodata
------------+------------+-------+--------+--------+--------+-------+-------+------+----------+--------
        135 |         35 |   100 |    100 |   0.01 |  -0.01 |     0 |     0 | 4326 |        1 |      0
(1 行)
```

# NODATA値が1の時はどうなるか

NODATA値がピクセルタイプの最大値も最小値も越えていない場合には、どうなるでしょうか。

```
-- R1: 空のラスタを作る
WITH R1 AS (
  SELECT ST_MakeEmptyRaster(100,100, 135, 35, 0.01, -0.01, 0, 0, 4326) AS rast
),
-- R2: R1.rastにバンドを追加、NODATA値-9999.0
R2 AS (
  SELECT ST_AddBand(rast::raster, '64BF', 0.0, 1.0) AS rast FROM R1
),
-- R3: R2.rastでST_MapAlgebraExpr()を実行
R3 AS (
  SELECT ST_MapAlgebraExpr(rast, '8BUI', '[rast]') AS rast FROM R2
)
SELECT (ST_MetaData(rast)).*, ST_BandNoDataValue(rast) AS NODATA FROM R3;
```

NODATA値は``1``になりました。

```
 upperleftx | upperlefty | width | height | scalex | scaley | skewx | skewy | srid | numbands | nodata
------------+------------+-------+--------+--------+--------+-------+-------+------+----------+--------
        135 |         35 |   100 |    100 |   0.01 |  -0.01 |     0 |     0 | 4326 |        1 |      1
(1 行)
```

# たぶんこういうことなんだろう

NODATA値がピクセルタイプ変更によってピクセルタイプの範囲外になってしまう場合には、変更先バンドのピクセルタイプの最大値より大きい場合には最大値に、変更先の最小値より小さい場合には最小値に、それぞれ設定され、NODATA値がピクセルタイプの最大値も最小値も越えていない場合には、NODATA値が保存されます。

# おわりに

ピクセルタイプが変わることでNODATA値が変わる場合があるし、変わらない場合もあることを紹介しました。``ST_MapAlgebraExpr()`` 等によってピクセルタイプを変更する時には、NODATA値を保存するだけのデータ長を確保するのとあわせて、NODATA値がどう変わるかを把握しながら変更する必要があります。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

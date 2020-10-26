---
title: "ジオグラフィを引数に取ることができる関数"
---
# はじめに

前節では、基本はジオメトリ型で、ジオグラフィ型の方が有利な時だけジオグラフィに変換するのを個人的にお勧めしていました。

というのも…たとえば次の実行結果を見て下さい。

```postgres
db=# SELECT ST_X('POINT(1 2)'::GEOMETRY);
 st_x 
------
    1
(1 row)

db=# SELECT ST_X('POINT(1 2)'::GEOGRAPHY);
ERROR:  function st_x(geography) does not exist
LINE 1: SELECT ST_X('POINT(1 2)'::GEOGRAPHY);
               ^
HINT:  No function matches the given name and argument types. You might need to add explicit type casts.
```

``ST_X()``はジオグラフィを食ってくれないんです。

ここでは、ジオグラフィ型を引数に取ることができる関数を書き出してみます。

# PostGIS 2.3.0でジオグラフィ型を引数に取ることができる関数

## 計測関数

  + ST_Area - 面積を計算します
  + ST_Azimuth - あるポイントからあるポイントへの方位を計算します
  + ST_Distance - 2つのポイントの間の距離を計算します
  + ST_Length - ラインストリングの長さを返します
  + ST_Perimeter - ポリゴンの境界の長さを返します
  + ST_Project - 指定したポイントから指定した方位に指定した距離を移動した先のポイントを計算します

## 処理関数

  + ST_Buffer - 指定した図形をもとに指定したサイズだけ拡大した図形を生成します
  + ST_Intersection - 2つの図形の共有部分を生成します

これらは、UTMまたはランベルト正積方位図法に投影したうえで処理を行い、再び極座標系に戻す操作をしています。

## 関係関数
  + ST_CoveredBy - AがBに包含されるかどうかをテストします
  + ST_Covers - AがBを包含するかどうかをテストします。ポリゴンがポイントをカバーするかを判断する場合のみ対応しています
  + ST_DWithIn - AとBの距離が指定した距離以下かどうかをテストします
  + ST_Intersects - AとBに共有部分があるかどうかをテストします。ジオグラフィでは、この関数は0.00001メートルの距離許容を持ち、回転楕円体計算でなく球面を使います。

## エディタ

  + ST_Segmentize - 与えられた距離を超える線分を持たないよう分割します。

## その他
### コンストラクタ

  + ST_GeogFromText
  + ST_GeogFromWKB
  + ST_GeographyFromText

### 出力関数

  + ST_AsBinary
  + ST_AsEWKT
  + ST_AsGeoJSON
  + ST_AsHEXEWKB
  + ST_AsKML
  + ST_AsSVG
  + ST_AsText

### 演算子
  + A = B - AのバウンダリボックスとBのバウンダリボックスが同じならTRUEを返します。
  + A <-> B - ジオグラフィAとBの間の球面上の距離を返します。PostgreSQL 9.5より前ではバウンディングボックスの重心の距離になってしまいます。
  + A && B - AのバウンダリボックスとBのバウンダリボックスがインタセクトするならTRUEを返します。

### アクセサ
  + ST_Summary - ジオグラフィの要約文 (タイプ、座標次元、リング数やポイント数等)を文字列で返します。

# ジオグラフィ型を引数に取らない関数を使いたい場合

ジオグラフィ型を使っていて、使いたい関数がジオグラフィ型を引数に取らない場合には、引数をジオメトリ型にキャストします。

ジオメトリ型へのキャストは、``::GEOMETRY`` を変数やリテラルの末尾に追加します。

```psql
db=# WITH P AS (SELECT 'SRID=4326;POINT(135 35)'::GEOGRAPHY AS geog)
  SELECT ST_X(geog) FROM P;
ERROR:  function st_x(geography) does not exist
行 2: SELECT ST_X(geog) FROM P;
             ^
HINT:  No function matches the given name and argument types. You might need to add explicit type casts.
db=# WITH P AS (SELECT 'SRID=4326;POINT(135 35)'::GEOGRAPHY AS geog)
  SELECT ST_X(geog::GEOMETRY) FROM P;
 st_x 
------
  135
(1 行)
```

# おわりに

ここでは、ジオグラフィ型を引数に取る関数の一覧を示しました。思ったよりも少ない、と思われたかもしれません。

しかし、ジオメトリ型とジオグラフィ型は相互にキャストしあえます。ジオメトリ型を取るべきか、ジオグラフィ型を取るべきか、で思い悩む必要はありません。どうにでもなります。

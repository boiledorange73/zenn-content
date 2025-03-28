---
title: "シングル系とマルチ系"
---
# はじめに

シングル系 (POINT, LINESTRING, POLYGON) とマルチ系 (MULTIPOINT, MULTILINESTRING, MULTIPOLYGON) とは、POINT と MULTIPOINT のように空間次元が同じであっても別のジオメトリタイプになります。

このため、返り値が知らない間にマルチ系になったり、逆にシングル系になったりすると、ジオメトリタイプが合わないエラーが不用意に発生することがあります。これに対応するためには、シングル系とマルチ系とを一方に揃える処理が必要です。

「ジオメトリタイプがPOLYGONなのにMULTIPOLYGONが来たのでエラー」とか言われた時に驚かないための方法を考えます。

## ここでの用語

あくまでここでの用語です。

シングル系は、POINT, LINESTRING, POLYGON を指し、マルチ系は MULTIPOINT, MULTILINESTRING, MULTIPOLYGON を指します。

# シングル系とマルチ系とが混ざる問題

シングル系とマルチ系とが混ざるとエラーが発生することがあります。たとえば、シングル系カラムにマルチ系ジオメトリを入れ込む場合です。

```
db=# CREATE TABLE spt (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(POINT)
);

db=# INSERT INTO spt (geom) SELECT 'MULTIPOINT(1 1, 2 2)';

ERROR:  Geometry type (MultiPoint) does not match column type (Point)
```

同じ"MULTI"が付いているかついていないかだけの違いなのですが、"MULTIPOINT"と"POINT"を混ぜようとして失敗しました。

この問題の回避策を見てみましょう。

## 最近は自動で変換してくれるけど無理にマルチ系に変換する

``ST_Multi``関数で、シングル系を、要素数 1 のマルチ系に変換してくれます。なお元からマルチ系なら加工せずに返します。

```
db=# SELECT ST_AsEWKT(ST_Multi('POINT(0 0)'::GEOMETRY));

   st_asewkt
-----------------
 MULTIPOINT(0 0)
(1 row)
```

ただし、3.x からは、気を利かせてくれる場合があります。ジオメトリタイプがマルチ系の場合にシングル系を持ってくると、自動で``ST_Multi``を実行してくれます。

```
db=# CREATE TABLE mpt (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(MULTIPOINT)
);
db=# INSERT INTO mpt (geom) SELECT 'POINT(10 10)';
INSERT 0 1
db=# SELECT ST_AsEWKT(geom) FROM mpt;
    st_asewkt
-------------------
 MULTIPOINT(10 10)
(1 row)
```

以前は、これはジオメトリタイプが合わないということでエラーになっていましたので、後方互換を考える場合には``ST_Multi``を使うようにして下さい。

```
WITH point AS (
  SELECT 'POINT(20 10)'::GEOMETRY geom
)
INSERT INTO mpt (geom) SELECT ST_Multi(geom) geom FROM point;
```

## シングル系にするにはダンプだけど注意が必要

マルチ系を無理やりシングル系にするには``ST_Dump``を使って「ダンプ」を行います。ただし、やや注意が必要です。

まず、返り値はジオメトリ型でなく、``geometry_dump``型になります。これは、ダンプされたジオメトリを持つ``geom``要素と、抽出したシングルジオメトリがマルチ系ジオメトリ内の何番目にいたかを示す``path``要素からなります。

```
WITH points AS (
  SELECT 1 gid, 'first' name, 'MULTIPOINT(0 0, 1 1)'::GEOMETRY geom
)
SELECT ST_Dump(geom) FROM points;

                     st_dump
--------------------------------------------------
 ({1},010100000000000000000000000000000000000000)
 ({2},0101000000000000000000F03F000000000000F03F)
(2 rows)
```


``path``要素は通常は使いません。なので、だいたい``(ST_Dump(geom)).geom``というふうに使います。

もうひとつ注意すべき点は、ダンプ対象となっているカラム以外のカラムには同じ値が入ることです。

```
WITH points AS (
  SELECT 1 gid, 'first' name, 'MULTIPOINT(0 0, 1 1)'::GEOMETRY geom
)
SELECT gid, name, (ST_Dump(geom)).geom FROM points;
```

```
 gid | name  |                    geom
-----+-------+--------------------------------------------
   1 | first | 010100000000000000000000000000000000000000
   1 | first | 0101000000000000000000F03F000000000000F03F
(2 rows)
```

この結果を、``gid``にUNIQUE属性がついているテーブルに``INSERT INTO``しようとしたら、確実にエラーが出ます。
回避策は``ST_Dump``を使う場合には、UNIQUE属性のついているカラムに値を不用意に入れない等、気を付ける必要があります。

ただ、むしろエラーが出て止まってくれた方が、手戻りがとんでもなく大変になるまで突き進んだ後で問題に気付くよりかは、よほど助かると思います。

# おわりに

ここでは、シングル系とマルチ系との相互変換の方法を紹介しました。また、最近のPostGISではマルチ系カラムにシングル系ジオメトリを入れても大丈夫という（前から触っていた人からしたら）驚きの情報を示しました。

間ある地形、シングル系

不用意にジオメトリタイプが変わっても驚かずに、冷静に対応できるようになると思います。

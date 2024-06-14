---
title: "ジオメトリ計測関数"
---
# はじめに

地理空間情報を扱う際には、ジオメトリ間の距離や、線ジオメトリの総延長、面ジオメトリの面積といった計算が必要になる局面は多くあります。これらの関数を「計測関数」と呼んでいて、ここでは、計測関数の主だったものをご紹介します。

## 地理座標系ジオメトリは使わないで

ジオメトリ計測関数では、基本的にはデカルト距離を返します。

地理座標系 (経度、緯度)のジオメトリを与えた場合には、地理座標系の数値をそのまま使うため、長さの単位が「度」とか、面積の単位が「平方度」とかいう、謎な単位になってしまっています。返される値は、使えないような値です。

地理座標系のままでは使わないでください。投影座標系にするか、ジオグラフィ型を使います。


# 関数たち

## ST_Area

[ST_Area](https://postgis.net/docs/ja/ST_Area.html) は、面ジオメトリの面積を求めます。面ジオメトリでない場合には ``0`` を返します。

### 正方形の面積

1辺 ``20`` の正方形の面積を求めてみます。面積は 400 になりますね。

```
db=# SELECT ST_Area('POLYGON((0 0, 20 0, 20 20, 0 20, 0 0))'::GEOMETRY);
 st_area
---------
     400
```

ちゃんと ``400`` が出力されました。

### 穴あき正方形の面積

1辺 ``20`` の正方形に、1辺 ``10`` の穴をあけてみました。面積を求めてみます。面積は ``400-100 = 300`` になりますね。

```
db=# SELECT ST_Area('POLYGON((0 0, 20 0, 20 20, 0 20, 0 0),(5 5, 5 15, 15 15, 15 5, 5 5))'::GEOMETRY);
 st_area
---------
     300
```

これもちゃんと ``300`` が出力されました。


### 線ジオメトリの「面積」

ラインストリングの面積を求めてみましょう。…これは、面じゃないので、面積なんて出てこないですよね。

```
db=# SELECT ST_Area('LINESTRING(0 0, 20 0, 20 20, 0 20, 0 0)'::GEOMETRY);
 st_area
---------
       0
```

面でないので ``0`` が出力されます。

## ST_Length と ST_Perimeter

[ST_Length](https://postgis.net/docs/ja/ST_Length.html) は、線ジオメトリの長さを求めます。線ジオメトリでない場合には ``0`` を返します。
[ST_Perimeter](https://postgis.net/docs/ja/ST_Perimeter.html) は、面ジオメトリの周囲長を求めます。面ジオメトリでない場合には ``0`` を返します。

### 線ジオメトリに対する ST_Length (正しい使い方)

右に``10``行って上に``10``行って右に``10``行くラインストリングの長さを求めます。

合計で``30``になればいいですね。

```
db=# SELECT ST_Length('LINESTRING(0 0, 10 0, 10 10, 10 20)'::GEOMETRY);
 st_length
-----------
        30
```

ちゃんと答が出ました。

### 面ジオメトリに対する ST_Perimeter (正しい使い方)

1辺 ``20`` の正方形に、1辺 ``10`` の穴をあけたポリゴンの周囲長を見てみましょう。``4*20+4*10 = 80+40 = 120`` になるはずです。


```
db=# SELECT ST_Perimeter(
  'POLYGON((0 0, 20 0, 20 20, 0 20, 0 0),(5 5, 5 15, 15 15, 15 5, 5 5))'::GEOMETRY
);
 st_perimeter
--------------
          120
```

ちゃんと``120``と出ました。


### 線ジオメトリに対する ST_Perimeter (0が返る)

ラインストリングはポリゴンではないので「周囲」という概念が無く、周囲長はありません。

```
db=# SELECT ST_Perimeter('LINESTRING(0 0, 10 0, 10 10, 10 20)'::GEOMETRY);
 st_perimeter
--------------
            0
```

``0``を返します。

### 面ジオメトリに対する ST_Length (0が返る)

ポリゴンはラインストリングではないので``0``を返します。


```
db=# SELECT ST_Length(
  'POLYGON((0 0, 20 0, 20 20, 0 20, 0 0),(5 5, 5 15, 15 15, 15 5, 5 5))'::GEOMETRY
);
 st_length
-----------
         0
```

## ST_Distance と ST_MaxDistance と ST_ShortestLine と ST_LongestLine

  * [ST_Distance](https://postgis.net/docs/ja/ST_Distance.html) は、ジオメトリ間の最短距離を求めます。
  * [ST_MaxDistance](https://postgis.net/docs/ja/ST_MaxDistance.html) は、ジオメトリ間の最長距離を求めます。

点と点はもとより、点と面も可能です。点と点だけは距離が一意に決まるのですが、それ以外の場合には一意ではないので「最短距離」「最長距離」は違う値になります。

  * [ST_ShortestLine](https://postgis.net/docs/ja/ST_ShortestLine.html) は、ジオメトリ間の最短ラインを求めます。
  * [ST_LongestLine](https://postgis.net/docs/ja/ST_LongestLine.html) は、ジオメトリ間の最長ラインを求めます。

これらの関数の返り値となるラインジオメトリの長さが、``ST_Distance`` や ``ST_MaxDistance`` の返り値と同じになります。

### 点と点の最短距離/ライン

```
db=# SELECT ST_Distance('POINT(0 0)'::GEOMETRY, 'POINT(10 10)'::GEOMETRY);
    st_distance
--------------------
 14.142135623730951
```

### 点と点の最短ライン

```
db=# SELECT ST_AsText(ST_ShortestLine('POINT(0 0)'::GEOMETRY, 'POINT(10 10)'::GEOMETRY));
       st_astext
-----------------------
 LINESTRING(0 0,10 10)
```

![点と点のライン](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/measurement/1.png)


### 点と点の最長距離

```
db=# SELECT ST_MaxDistance('POINT(0 0)'::GEOMETRY, 'POINT(10 10)'::GEOMETRY);
   st_maxdistance
--------------------
 14.142135623730951
```

点と点の最短距離と同じになります。


### 点と点の最長ライン

```
db=# SELECT ST_AsText(ST_LongestLine('POINT(0 0)'::GEOMETRY, 'POINT(10 10)'::GEOMETRY));
       st_astext
-----------------------
 LINESTRING(0 0,10 10)
```

前のセクションから想像がつくと思いますが、点と点の最短ラインと同じになります。

### 点と線の最短距離

```
db=# SELECT ST_Distance('POINT(0 0)'::GEOMETRY, 'LINESTRING(10 -10, 10 10)'::GEOMETRY);
 st_distance
-------------
          10
```

### 点と線の最短ライン

```
db=# SELECT ST_AsText(ST_ShortestLine(
  'POINT(0 0)'::GEOMETRY,
  'LINESTRING(10 -10, 10 10)'::GEOMETRY
));
      st_astext
----------------------
 LINESTRING(0 0,10 0)
```

![点と線の最短ライン](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/measurement/2-1.png)

### 点と線の最長距離

```
db=# SELECT ST_MaxDistance('POINT(0 0)'::GEOMETRY, 'LINESTRING(10 -10, 10 10)'::GEOMETRY);
   st_maxdistance
--------------------
 14.142135623730951
```

### 点と線の最長ライン

```
db=# SELECT ST_AsText(ST_LongestLine(
  'POINT(0 0)'::GEOMETRY,
  'LINESTRING(10 -10, 10 10)'::GEOMETRY
));

       st_astext
------------------------
 LINESTRING(0 0,10 -10)
```

![点と線の最長ライン](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/measurement/2-2.png)


### ポリゴンとポリゴンの最短距離

```
db=# SELECT ST_Distance(
  'POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))'::GEOMETRY,
  'POLYGON((25 5, 30 10, 25 15, 20 10, 25 5))'::GEOMETRY
);
 st_distance
-------------
          10
```

### ポリゴンとポリゴンの最短ライン

```
db=# SELECT ST_AsText(ST_ShortestLine(
  'POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))'::GEOMETRY,
  'POLYGON((25 5, 30 10, 25 15, 20 10, 25 5))'::GEOMETRY
));
        st_astext
-------------------------
 LINESTRING(10 10,20 10)
```

![ポリゴンとポリゴンの最短ライン](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/measurement/3-1.png)


### ポリゴンとポリゴンの最長距離

```
db=# SELECT ST_MaxDistance(
  'POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))'::GEOMETRY,
  'POLYGON((25 5, 30 10, 25 15, 20 10, 25 5))'::GEOMETRY
);
   st_maxdistance
--------------------
 31.622776601683793
```

### ポリゴンとポリゴンの最長ライン

```
db=# SELECT ST_AsText(ST_LongestLine(
  'POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))'::GEOMETRY,
  'POLYGON((25 5, 30 10, 25 15, 10 20, 25 5))'::GEOMETRY
));
       st_astext
-----------------------
 LINESTRING(0 0,30 10)
```

![ポリゴンとポリゴンの最長いライン](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/measurement/3-2.png)


# おわりに

今回は、線の総延長、面積、距離等の計算を行う計測系関数の一部をご紹介しました。

距離計測関数については、最短、最長があるので関数の数が倍になっていますが、必要に応じて使い分けて下さい。

また、地理座標系（経度、緯度）の場合には、地理座標系の数値をそのまま使うため、おかしな値になります。地理座標系のままでは使わないでください。投影座標系にするか、ジオグラフィ型を使います。

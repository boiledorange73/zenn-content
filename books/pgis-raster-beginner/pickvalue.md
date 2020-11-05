---
title: "指定位置の標高を得るクエリとインデックス"
---
# はじめに

[PostGISラスタにGeoTIFFデータを格納する](raster2pgsql)で入れたデータを少し使ってみましょう。

# 指定位置の標高を見る

指定した位置のピクセルデータを見てみましょう。``ST_Value``を使います。この関数は、指定した位置の値を取得します。

第二引数がEWKTで表現されたジオメトリです。EWKBについては https://zenn.dev/boiledorange73/books/caea8d4c77dbba2e23a0/viewer/conv-ewkt-geom を参照して下さい。

``SRID=4612``となっているのは、ラスタの空間参照系がJGD2000経度緯度であり、それにあわせています。

また、複数のバンドを持つラスタの場合には、バンド番号を指定します。番号は1はじまりです。

```sql
raster=# SELECT ST_Value(rast, 'SRID=4612;POINT(133.37 34.39)'::GEOMETRY) FROM dem;
     st_value
-------------------
 237.3000030517578
(1 行)
```

上記のクエリと結果から、東経133.37度、北緯34.39度の地点の標高は 237.3メートル であると分かりました。

これを応用して、経度緯度を受け付けて、このクエリを実行して、標高を返す、というウェブサービスはかなり簡単にできそうです（測量法に基づく使用承認が必要だろうと思います）。

## ST_Value()はピクセル値を得る関数です

``ST_Value``を指定した位置の値を取得する関数と説明しましたが、もう少し踏み込むと、指定した位置を含むピクセルの値を取得する関数です。ピクセル座標系で列と行とを指定して、そのピクセルの値を得る形式も用意されています。いずれの形式でも、この関数は、ピクセル値を得るものであって、それ以上のことは行いません。

指定した位置の値を得るのに、近傍のピクセルを取得して補間を行うことを期待するかも知れませんが、そのような操作は行いません。

# たくさんのラスタファイルを格納している場合

今回はラスタファイルが1つだけなのでこれでいいのですが、たくさん格納している時は問題になります。

ここからは実際に手を動かすことができないのですが、1125*750のラスタファイルを4797個格納しているとします。

このデータベースでは、上のようなクエリだと大変なことになります。

```
db=# SELECT ST_Value(rast, 'SRID=4612;POINT(133.37 34.39)'::GEOMETRY) FROM dem10;
NOTICE:  Attempting to get pixel value with out of range raster coordinates: (-52920, 11490)
NOTICE:  Attempting to get pixel value with out of range raster coordinates: (
NOTICE:  Attempting to get pixel value
NOTICE:  Attempting
NOTICE:
...
```

端折っていますが、ずーっと``NOTICE``が出てきます（たぶん4796行）。

実行時間を見てみました。

```
db=# explain analyze SELECT ST_Value(rast, 'SRID=4612;POINT(133.37 34.39)'::GEOMETRY);
NOTICE:
...
                                                  QUERY PLAN                                                   
---------------------------------------------------------------------------------------------------------------
 Seq Scan on dem10  (cost=0.00..1302.22 rows=4797 width=8) (actual time=1211.548..83482.535 rows=4797 loops=1)
 Planning Time: 129.496 ms
 Execution Time: 83523.111 ms
```

83秒です。とんでもない実行時間になりました。

## ST_Intersects()で対象を絞り込む

まず、``NOTICE``が出る問題に対処しましょう。

``WHERE``節で``ST_Intersects()``を使って、対象となるラスタだけに絞り込みます。

```
db=# SELECT ST_Value(rast, 'SRID=4612;POINT(133.37 34.39)'::GEOMETRY) FROM dem10 WHERE ST_Intersects(rast,'SRID=4612;POINT(133.37 34.39)'::GEOMETRY);
     st_value      
-------------------
 237.3000030517578
(1 行)
```

NOTICEが出なくなりました。

## インデックスを作る

実行時間の短縮は、やはりインデックスです。インデックスは``CREATE INDEX``で作りますが、ジオメトリ/ジオグラフィでは``USING GiST``を追加しましたね。ラスタカラムでは、さらに追加して、関数インデックスとして指定します。

```
db=# CREATE INDEX ix_dem10_rast ON dem10 USING gist (st_convexhull(rast));
```

## 最終的にはこんなかんじ

```
db=# explain analyze SELECT ST_Value(rast, 'SRID=4612;POINT(133.37 34.39)'::GEOMETRY)
  FROM dem10 WHERE ST_Intersects(rast,'SRID=4612;POINT(133.37 34.39)'::GEOMETRY);
                                                       QUERY PLAN                                                       
------------------------------------------------------------------------------------------------------------------------
 Index Scan using ix_dem10_rast on dem10  (cost=0.15..10.92 rows=1 width=8) (actual time=13.898..13.903 rows=1 loops=1)
   Index Cond: ((rast)::geometry && '010100002004120000A4703D0AD7AB604052B81E85EB314140'::geometry)
   Filter: _st_intersects('010100002004120000A4703D0AD7AB604052B81E85EB314140'::geometry, rast, NULL::integer)
 Planning Time: 0.177 ms
 Execution Time: 13.931 ms
```

大量のラスタファイルがあっても、こんなかんじで、13ミリ秒で対応できました。

遅いと思うかもしれませんが、83秒と比べたら、とても短い時間で実行できたことになります。

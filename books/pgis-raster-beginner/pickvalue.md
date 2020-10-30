---
title: "指定位置の標高を得るクエリとインデックス"
---
# はじめに

[PostGISラスタにGeoTIFFデータを格納する](raster2pgsql)で入れたデータを少し使ってみましょう。

# 指定位置の標高を見る

指定した位置のピクセルデータを見てみましょう。

```sql
raster=# SELECT ST_Value(rast, 'SRID=4612;POINT(133.37 34.39)'::GEOMETRY) FROM dem;
     st_value
-------------------
 237.3000030517578
(1 行)
```

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

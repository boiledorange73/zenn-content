---
title: "簡略化をやってみよう"
---

# はじめに


# 簡略化させてみよう

まず、受け皿となるテーブルを作ります。

```
CREATE TABLE simplified_3857 (
  gid SERIAL PRIMARY KEY,
  city_code TEXT,
  geom GEOMETRY(MULTIPOLYGON, 3857)
);
CREATE INDEX ix_simplified_3857_city_code ON simplified_3857 (city_code);
CREATE INDEX ix_simplified_3857_geom ON simplified_3857 USING GiST(geom);
```

なんてことはない、普通に``gid``と、属性とマルチポリゴンのジオメトリを持つ、いたって普通のテーブルです。

ここに、トポジオメトリカラムを持つテーブルから、トポジオメトリは簡略化し、それ以外の属性は変更なく、複写します。

```
INSERT INTO simplified_3857 (city_code, geom)
SELECT city_code, ST_Simplify(topo,100)
FROM topogeom_t1_3857 ORDER BY gid;
```

``ST_Simplify``の引数は次の通りです。

1. トポジオメトリ
2. 許容範囲、この値が大きいほど粗くなる

# おわりに

# 出典
本記事では、国土交通省国土政策局が発行する国土数値情報（行政区域）を利用しました。


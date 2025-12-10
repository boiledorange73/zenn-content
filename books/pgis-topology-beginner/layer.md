---
title: "階層レイヤー構造をつくっちゃおう!"
---

# はじめに

トポジオメトリー (=レイヤー) は、階層構造を持つことができます。

たとえば市区町村ポリゴンが集まって都道府県ポリゴンができていますよね? これが階層構造です。

## 親子レイヤーとプリミティブとの関係

ここで親子関係がどうなるかをはっきりさせます。

|親レイヤー|子レイヤー|
|---------|---------|
|レイヤー1 |レイヤー2 |

![プリミティブとレイヤー階層の関係](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/layer/whatislayer.png)

レイヤー1は、親レイヤーを持たないレイヤーです。この場合にはノード、エッジ、フェイスがある種の親レイヤーと言えます。このノード、エッジ、フェイスを総称して「**プリミティブ**」と呼びます。

レイヤー2の親レイヤーはレイヤー1なので、レイヤー1のトポジオメトリーを参照します。ここで、プリミティブからの参照はできません。直接の親となるレイヤーからしか参照できません。

## 親レイヤー/子レイヤーと市区町村レイヤー/都道府県レイヤー

|親レイヤー|子レイヤー|
|---------|---------|
|市区町村  |都道府県  |

なんとなく都道府県が「親」っぽいと思うかもしれませんが、**トポロジープリミティブに近い方が親**です。

もしくは、**レイヤーを生成する順番が古い方が親**です。市区町村レイヤーから都道府県レイヤーを作るというのに、市区町村レイヤーが存在しないと都道府県レイヤーが作れませんよね。

# 前準備

前回の市区町村レイヤー (=トポジオメトリー) に3市を追加します。

```
INSERT INTO mncpl(mid, name, topogeo)
SELECT 2001, '左上市', topology.toTopoGeom(
  'POLYGON((0 10, 10 10, 5 15, 0 15, 0 10))', 'topo1', 1
);

INSERT INTO mncpl(mid, name, topogeo)
SELECT 2002, '中上市', topology.toTopoGeom(
  'POLYGON((10 10, 15 10, 15 15, 5 15, 10 10))', 'topo1', 1
);

INSERT INTO mncpl(mid, name, topogeo)
SELECT 3001, '右上市', topology.toTopoGeom(
  'POLYGON((15 10, 20 10, 20 15, 15 15, 15 10))', 'topo1', 1
);
```

これで、次のような構成になったと思います。

![市区町村レイヤーの様子](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/layer/mncpl.png)

``relation``は次のようになっています。

```
SELECT * FROM topo1.relation;

 topogeo_id | layer_id | element_id | element_type 
------------+----------+------------+--------------
          1 |        1 |          1 |            3
          2 |        1 |          2 |            3
          3 |        1 |          3 |            3
          4 |        1 |          4 |            3
          5 |        1 |          5 |            3
(5 rows)
```

# 都道府県レイヤーを作成する

## テーブル、トポジオメトリーカラムの生成

テーブルとトポジオメトリーカラムを生成します。市区町村レイヤーと同じ要領で行います。

```
-- トポジオメトリーカラムを除いたテーブルを生成
CREATE TABLE pref (
    gid SERIAL PRIMARY KEY,
    pid INT UNIQUE,
    name TEXT
);
-- トポジオメトリーカラムを pref に追加
SELECT topology.AddTopoGeometryColumn('topo1', 'public', 'pref', 'topogeo', 'POLYGON', 1);
```

``AddTopoGeometryColumn()`` の 第6引数が ``1`` となっています。これは ``child_layer=1`` を意味していてい、``mncpl.topogeom``のレイヤーIDを指定しています。

## 県データを作る

県トポジオメトリーを生成するには [CreateTopoGeom()](https://postgis.net/docs/ja/CreateTopoGeom.html) を使用します。

そのためには、[TopoElementArray](https://postgis.net/docs/ja/topoelementarray.html) 型の値が必要です。

これを生成するには、[TopoElement](https://postgis.net/docs/ja/topoelement.html) 型の集合を [TopologyElementArray_Agg() ](https://postgis.net/docs/ja/topoelementarray_agg.html) に与える必要があります。

なお、``TopoElement`` は、トポジオメトリーからのキャストで得られます。

```
SELECT mid, topogeo::TopoElement ta FROM mncpl ORDER BY mid;

 mid  |  ta   
------+-------
 1001 | {1,1}
 1002 | {2,1}
 2001 | {3,1}
 2002 | {4,1}
 3001 | {5,1}
```

``GROUP BY`` でまとめて ``TopologyElementArray_Agg()`` に渡すと ``TopoElementArray`` が生成されます。

```
WITH Q1 AS (
  SELECT mid/1000 pid, topology.TopoElementArray_Agg(topogeo::topoelement) tae
  FROM mncpl GROUP BY mid/1000
)
SELECT pid, tae FROM Q1 ORDER BY pid;

pid |      tae      
-----+---------------
   1 | {{1,1},{2,1}}
   2 | {{3,1},{4,1}}
   3 | {{5,1}}
(3 rows)
```

そして ``Q1.tae`` (``layer_id=1``) はそのまま ``CreateTopoGeom()`` に渡します。
``CreateTopoGeom()`` の第3引数は ``pref.topogeo`` のレイヤーID (ここでは ``2``) を指定します。

```
WITH Q1 AS (
  SELECT mid/1000 pid, topology.TopoElementArray_Agg(topogeo::topoelement) tae
  FROM mncpl GROUP BY mid/1000
)
INSERT INTO pref(pid, topogeo)
SELECT pid, topology.CreateTopoGeom('topo1', 3, 2, tae)
FROM Q1 ORDER BY pid;
```

これで、次のようなレイヤーができあがります。

![都道府県レイヤーの様子](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/layer/pref.png)

## relation を見てみる

``topo1.relation``を見てみましょう。

```
SELECT * FROM topo1.relation ORDER BY layer_id, topogeo_id;

 topogeo_id | layer_id | element_id | element_type 
------------+----------+------------+--------------
          1 |        1 |          1 |            3
          2 |        1 |          2 |            3
          3 |        1 |          3 |            3
          4 |        1 |          4 |            3
          5 |        1 |          5 |            3
          1 |        2 |          1 |            1
          1 |        2 |          2 |            1
          2 |        2 |          3 |            1
          2 |        2 |          4 |            1
          3 |        2 |          5 |            1
(10 rows)
```

前よりもずいぶんとデータが増えていると思います。

ただし今回は、``layer_id``が``2``になっているタプルだけに注目してください。これが都道府県レイヤーです。

- ``layer_id=2`` のトポジオメトリーの ``element_type`` は全部 ``1`` になっています。ノードを指しているわけではありません。**子レイヤー**の場合には、**親レイヤーのID**が入ります。
- ``element_id``は、子レイヤーのIDです。当然ですが、ノードのIDではありません。

そのうえで ``layer_id=2`` かつ ``topogeo_id=1`` のトポジオメトリーを見てみましょう。

```
 topogeo_id | layer_id | element_id | element_type 
------------+----------+------------+--------------
...
          1 |        2 |          1 |            1
          1 |        2 |          2 |            1
...
```

このトポジオメトリーは、市区町村レイヤー (``layer_id=1``) の1番 (``mcode=1001``) と2番 (``mcode=1002``) からできていることが分かります。

市区町村レイヤーと都道府県レイヤーを併せて見てみてください。

# エッジを変更する

エッジを変更すると一部のフェイスが変更されますし、当然ポリゴン系のトポジオメトリーも変更されます。その様子を見てみましょう。

## エッジを変更すると市区町村レイヤーも都道府県レイヤーも影響を受けることがある

エッジをちょっと変更してみましょう。始端、終端は変更せず、途中の頂点だけ変えています。

```
UPDATE topo1.edge SET geom='LINESTRING(5 15, 0 10)' WHERE edge_id=8;
```

変更後の市区町村レイヤーは次のようになります。変更されてますね。

![市区町村トポジオメトリーが変更された様子](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/layer/layer_update_1m.png)

都道府県レイヤーも次のようになります。こちらも変更されてます。

![都道府県トポジオメトリーが変更された様子](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/layer/layer_update_1p.png)

## エッジを変更すると市区町村レイヤーは影響を受けるけど都道府県レイヤーは影響を受けないことがある

別のエッジを変更してみましょう。

```
UPDATE topo1.edge SET geom='LINESTRING(10 0, 15 2, 15 8 , 10 10)' WHERE edge_id=3;
```

そうすると、市区町村レイヤーは変更されます。

![市区町村トポジオメトリーが変更された様子](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/layer/layer_update_2m.png)

しかし、都道府県レイヤーは変更されていません。

![都道府県トポジオメトリーは変更されていない様子](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/layer/layer_update_2p.png)

当然と言えば当然ですが、子レイヤーで影響が出ない場合には、ちゃんと影響が出ないようになっています。

# おわりに

ここでは、トポジオメトリーのレイヤー構造について説明しました。階層構造が取れるのは「きっちり」の利点だろうと思います。大字・町丁目レイヤーなどの細かいフェイスを導入したり、逆に地域レイヤーのような大きなレイヤーを導入したりすることもできます (トポロジーにできるいいデータがあれば)。
---
title: "トポジオメトリ"
---

# はじめに

[PostGISを入れたらやってみよう](../../b1de0a18073af70946e0)の[シェープファイルのデータをインポートしてみよう コマンドライン編](../../b1de0a18073af70946e0/viewer/import-cli)で作った``t1``を元にしてみましょう。

# やっていく

まず、PostGISトポロジのエクステンションを追加します。

```
db=# CREATE EXTENSION postgis_topology;
CREATE EXTENSION
```

続いて、トポロジを追加します。空間参照系を``EPSG:3857``（Webメルカトル）にし、名前を``'topo_t1_3857``としました。

```
db=# SELECT topology.CreateTopology('topo_t1_3857', 3857);
 createtopology
----------------
              1
(1 行)
SELECT topology.CreateTopology('topo_t1_3857', 3857);
```

続いてトポジオメトリを持つテーブルを用意します。

まず、トポジオメトリ以外のカラムを持つテーブルを作ります。今回は自治体コードを持たせておきます。

```
db=# CREATE TABLE topogeom_t1_3857 (
  gid serial primary key,
  city_code TEXT
);
CREATE TABLE
```

テーブルが出来上がった後で、トポジオメトリカラムを追加します。

```
db=# SELECT topology.AddTopoGeometryColumn(
  'topo_t1_3857',
  'public', 'topogeom_t1_3857', 'topo',
  'POLYGON'
);
 addtopogeometrycolumn
-----------------------
                     1
(1 行)
```

``AddTopoGeometryColumn``の引数は次の通りです。

1. 使用するトポロジ
2. トポジオメトリのスキーマ
3. トポジオメトリのテーブル名
4. トポジオメトリのカラム名（カラムは新規に追加されます）
5. トポジオメトリのジオメトリタイプ（``POINT``, ``LINE``, ``POLYGON``, ``COLLECTION``のいずれか）


現時点でのトポロジのメタデータを見てみましょう。

```
db=# SELECT * FROM topology.layer;
 topology_id | layer_id | schema_name |    table_name    | feature_column | feature_type | level | child_id
-------------+----------+-------------+------------------+----------------+--------------+-------+----------
           1 |        1 | public      | topogeom_t1_3857 | topo           |            3 |     0 |
(1 行)
```

# トポジオメトリとしてジオメトリを叩き込む


普通にジオメトリを叩き込みます、が、``t1``は``EPSG:4612``で空間参照系が違うので、``ST_Transform()``で投影変換を施したジオメトリを叩き込みます。

```
db=# EXPLAIN ANALYZE
INSERT INTO topogeom_t1_3857(city_code, topo)
  SELECT n03_007::INT, topology.toTopoGeom(ST_Transform(geom, 3857), 'topo_t1_3857', 1)
  FROM t1;

                                                         QUERY PLAN                                                     
----------------------------------------------------------------------------------------------------------------------------
 Insert on topogeom_t1_3857  (cost=0.00..1191.28 rows=408 width=68) (actual time=40279.809..40279.809 rows=0 loops=1)
   ->  Subquery Scan on "*SELECT*"  (cost=0.00..1191.28 rows=408 width=68) (actual time=42.877..40274.282 rows=408 loops=1)
         ->  Seq Scan on t1  (cost=0.00..1183.12 rows=408 width=36) (actual time=42.869..40272.670 rows=408 loops=1)
 Planning Time: 0.102 ms
 Execution Time: 40279.902 ms
(5 行)
```

手持ちのパソコンでは40秒超かかりました。

普通にジオメトリを挿入するだけなら瞬殺なのですが、フェイスが共有するエッジの確認、エッジが共有するノードの確認とかで、結構時間がかかります。

# ここでQGISで見てみましょう

QGISでレイヤを追加して見ましょう。まず、PostGISレイヤの追加ダイアログを開きます。

![publicスキームレイヤ選択ダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/poweroftopo/01-layerdialog.png)

publicスキーマに``topogeom_t1_3857.topo``があります。これがトポジオメトリカラムです。なお、トポジオメトリとしても、ジオメトリとしても、候補に挙がっていますが、一方だけでいいです。

![topogeom_t1_3857.topoの表示例](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/poweroftopo/02-topo.png)


続いて、``topo_t1_3857``スキームを見てみます。

![topo_t1_3857スキームレイヤ選択ダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/poweroftopo/03-layerdialog-topology.png)

テーブル名``edge_data``を追加した後で``node``を追加します。両方ともカラム名は``geom``です。

![ノードとエッジの表示例](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/poweroftopo/04-node_edge.png)

こんな感じで、エッジとノードが出現します。ノードは、エッジが交わる/分岐する点にだけ出ているのが分かると思います。

ついでにフェイスも見てみましょう。こちらは、カラム名が``mbr``ということで、長方形になるだろうと想像できます。

![フェイスの表示例](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/poweroftopo/05-face.png)

長方形でした。


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



# 出典
本記事では、国土交通省国土政策局が発行する国土数値情報（行政区域）を利用しました。


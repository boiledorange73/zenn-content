---
title: "トポロジをいじってみよう"
---

# はじめに

ちょっとトポロジをいじってみて、どういう動きをするのか見てみましょう。

たぶん、トポジオメトリを多用することになると思いますが、その場合には、ここでの説明はほぼ役に立ちません。

# 準備

## データベースがあるものとします

topologydbという名前のデータベースがあり、PostGIS、PostGISトポロジが導入されているとします。まだ作っていないなら、[そもそもトポロジとは何やねん](whatistopology)を参考に作って下さい。

# 新しいトポロジを用意する

まず、新しいトポロジを用意します。

```
topologydb=# SELECT topology.CreateTopology('topo1', 3857, 0.1);
 createtopology
----------------
              1
(1 行)
```

これでトポロジが準備できました。

現在の``topo1``トポロジの状況を見てみます。

```
topologydb=# SELECT topology.TopologySummary('topo1');
                  topologysummary
----------------------------------------------------
 Topology topo1 (id 1, SRID 3857, precision 0.1)   +
 0 nodes, 0 edges, 0 faces, 0 topogeoms in 0 layers+

(1 行)
```

レイヤがありませんし、その中にはノードもエッジもフェイスもトポジオメトリもありません、と言っています。

# 初期のテーブル構成

```
topologydb=# SELECT * FROM pg_tables WHERE schemaname='topo1';
 schemaname | tablename | tableowner | tablespace | hasindexes | hasrules | hastriggers | rowsecurity
------------+-----------+------------+------------+------------+----------+-------------+-------------
 topo1      | face      | postgres   |            | t          | f        | t           | f
 topo1      | node      | postgres   |            | t          | f        | t           | f
 topo1      | edge_data | postgres   |            | t          | f        | t           | f
 topo1      | relation  | postgres   |            | t          | f        | t           | f
(4 行)
```

ほぼ空っぽですが、フェイスに１件存在します。

```
topologydb=# SELECT * FROM topo1.face;
 face_id | mbr
---------+-----
       0 |
(1 行)
```

これは、ユニバースフェイスと呼ばれる、無限の広さを持つフェイスです。

# さっそくやってみよう

## 孤立ノードの追加

```
topologydb=# SELECT topology.ST_AddIsoNode(
  'topo1',
  0,
  'SRID=3857;POINT(15540700 4254000)'
);

 st_addisonode
---------------
             1
(1 行)
```

返り値は、作成されたノードのノードIDです。覚えておいて下さい。

この時点でのノードの格納状況は次のようになっています。

```
topologydb=# SELECT * FROM topo1.node;
 node_id | containing_face |                        geom
---------+-----------------+----------------------------------------------------
       1 |               0 | 0101000020110F0000000000803BA46D41000000004C3A5041
(1 行)
```

QGISで見ると次のようになっています。

![1ノードを追加した様子](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/editing/1.png)


同じように、二つ孤立ノードを追加します。

```
topologydb=# SELECT topology.ST_AddIsoNode(
  'topo1',
  0,
  'SRID=3857;POINT(15541700 4254000)'
);

 st_addisonode
---------------
             2
(1 行)
```

```
topologydb=# SELECT topology.ST_AddIsoNode(
  'topo1',
  0,
  'SRID=3857;POINT(15540700 4253000)'
);

 st_addisonode
---------------
             3
(1 行)
```

ノードの格納状況を確認すると、次のようになっていました。

```
topologydb=# select * from topo1.node;
 node_id | containing_face |                        geom
---------+-----------------+----------------------------------------------------
       1 |                 | 0101000020110F0000000000803BA46D41000000004C3A5041
       2 |                 | 0101000020110F000000000080B8A46D41000000004C3A5041
       3 |                 | 0101000020110F0000000000803BA46D410000000052395041
(3 行)
```

QGISで見ると次のようになっています。

![3ノードを追加した様子](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/editing/2.png)

## 孤立ノードをつなぐエッジを作る

```
topologydb=# SELECT topology.ST_AddEdgeModFace(
  'topo1',
  1, 2,
  'SRID=3857;LINESTRING(15540700 4254000, 15541700 4254000)'
);

 st_addedgemodface
-------------------
                 1
(1 行)
```

```
topologydb=# SELECT * FROM topo1.node;
 node_id | containing_face |                        geom
---------+-----------------+----------------------------------------------------
       3 |               0 | 0101000020110F0000000000803BA46D410000000052395041
       1 |                 | 0101000020110F0000000000803BA46D41000000004C3A5041
       2 |                 | 0101000020110F000000000080B8A46D41000000004C3A5041
(3 行)


topologydb=# SELECT * FROM topo1.edge_data;
 edge_id | start_node | end_node | next_left_edge | abs_next_left_edge | next_right_edge | abs_next_right_edge | left_face | right_face |                                            geom
---------+------------+----------+----------------+--------------------+-----------------+---------------------+-----------+------------+--------------------------------------------------------------------------------------------
       1 |          1 |        2 |             -1 |                  1 |               1 |                   1 |         0 |          0 | 0102000020110F000002000000000000803BA46D41000000004C3A504100000080B8A46D41000000004C3A5041
(1 行)
```

エッジデータが追加されています。
1番ノードから2番ノードに向かうエッジで、左側のフェイスも右側のフェイスもユニバースフェイスです。

QGISで見ると次のようになっています。

![1エッジを追加した様子](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/editing/3.png)


同様に1番ノードから3番ノードにも作ります。

```
topologydb=# SELECT topology.ST_AddEdgeModFace(
  'topo1',
  1, 3,
  'SRID=3857;LINESTRING(15540700 4254000, 15540700 4253000)'
);

 st_addedgemodface
-------------------
                 2
(1 行)
```

```
topologydb=# SELECT * FROM topo1.node;
 node_id | containing_face |                        geom
---------+-----------------+----------------------------------------------------
       1 |                 | 0101000020110F0000000000803BA46D41000000004C3A5041
       2 |                 | 0101000020110F000000000080B8A46D41000000004C3A5041
       3 |                 | 0101000020110F0000000000803BA46D410000000052395041
(3 行)


topologydb=# SELECT * FROM topo1.edge_data;
 edge_id | start_node | end_node | next_left_edge | abs_next_left_edge | next_right_edge | abs_next_right_edge | left_face | right_face |                                            geom
---------+------------+----------+----------------+--------------------+-----------------+---------------------+-----------+------------+--------------------------------------------------------------------------------------------
       2 |          1 |        3 |             -2 |                  2 |               1 |                   1 |         0 |          0 | 0102000020110F000002000000000000803BA46D41000000004C3A5041000000803BA46D410000000052395041
       1 |          1 |        2 |             -1 |                  1 |               2 |                   2 |         0 |          0 | 0102000020110F000002000000000000803BA46D41000000004C3A504100000080B8A46D41000000004C3A5041
(2 行)
```

QGISで見ると次のようになっています。

![2エッジを追加した様子](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/editing/4.png)

## 三つ目のエッジで四角形を完成させる

```
topologydb=# SELECT topology.ST_AddEdgeModFace(
  'topo1',
  3, 2,
  'SRID=3857;LINESTRING(15540700 4253000, 15541700 4253000, 15541700 4254000)'
);

 st_addedgemodface
-------------------
                 3
(1 行)
```

```
topologydb=# SELECT * FROM topo1.edge_data;
 edge_id | start_node | end_node | next_left_edge | abs_next_left_edge | next_right_edge | abs_next_right_edge | left_face | right_face |                                                            geom                                       
---------+------------+----------+----------------+--------------------+-----------------+---------------------+-----------+------------+----------------------------------------------------------------------------------------------------------------------------
       2 |          1 |        3 |              3 |                  3 |               1 |                   1 |         1 |          0 | 0102000020110F000002000000000000803BA46D41000000004C3A5041000000803BA46D410000000052395041
       3 |          3 |        2 |             -1 |                  1 |              -2 |                   2 |         1 |          0 | 0102000020110F000003000000000000803BA46D41000000005239504100000080B8A46D41000000005239504100000080B8A46D41000000004C3A5041
       1 |          1 |        2 |             -3 |                  3 |               2 |                   2 |         0 |          1 | 0102000020110F000002000000000000803BA46D41000000004C3A504100000080B8A46D41000000004C3A5041
(3 行)
```

エッジが一つ追加されています。

QGISで見ると次のようになっています。

![3エッジを追加した様子](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/editing/5.png)

このエッジは3番ノードから2番ノードに向かっていますが、ラインストリングは南西から北西に向かう最短の線ではなく、まず東に向かったあと北に向かっています。

このようなエッジを追加することも可能です。

そもそも、ノードは、あくまで複数の**エッジで共有される**べきものであり、エッジの**構成するポイント全てに存在するものではない**のです。ここが、ノードとポイントの決定的に違う点です。

よって、3番エッジの途中にある角にノードは存在すべきではありません。もちろん、3番ノードを横エッジと縦エッジに分ける際には、ノードは必要となります。

# 四角形フェイスができているのを確認する

続いて、フェイスを確認します。

```
topologydb=# SELECT * FROM topo1.face;
 face_id |                                                                                                mbr           
---------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
       0 |
       1 | 0103000020110F00000100000005000000000000803BA46D410000000052395041000000803BA46D41000000004C3A504100000080B8A46D41000000004C3A504100000080B8A46D410000000052395041000000803BA46D410000000052395041
(2 行)
```

フェイスが一つ追加されました。フェイスを追加する操作なしに、です。

このように、エッジで閉じるルートができたなら、そのつどフェイスが自動的に追加されます。

# おわりに

今回は、孤立ノードを追加して、ノードを繋げてエッジを作成して、フェイスができあがる様子を見て頂きました。

エッジとノード、エッジとフェイスがかかわりあっていて、ポイント、ラインストリング、ポリゴンのように相互に独立しているわけではないことが分かって頂けたかと思います。

次回は、各テーブルの中を、もう少ししっかりと見てみます。

# 出典

QGISの背景図に[歴史的農業環境閲覧システム](https://habs.dc.affrc.go.jp/)が提供する関東平野迅速測図を使用しました、思い付きで。



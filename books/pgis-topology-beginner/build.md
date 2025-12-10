---
title: "ノード、エッジ、フェイスを生成しちゃおう！"
---

# はじめに

トポロジーの構築を試しにやってみましょう。エクステンションをインストールするところから、ノードを打ち、エッジを引き、フェイスを生成してみます。

# 準備

使用するデータベースにPostGISエクステンションとPostGISトポロジエクステンションとを導入して下さい。

```
postgres=# create database topologydb;
CREATE DATABASE
postgres=# \c topologydb
データベース"topologydb"にユーザ"postgres"として接続しました。
topologydb=# create extension postgis;
CREATE EXTENSION
topologydb=# create extension postgis_topology;
CREATE EXTENSION
topologydb=# 
```

## topologyスキーマを見てみよう

エクステンションがインストールされた直後のスキーマ、テーブルの状況を見てみましょう。

デフォルト(public)でなく、topologyという名前のスキーマを作り、そこに関数や内部使用データを格納します。

```
topologydb=# SELECT * FROM pg_tables WHERE schemaname='topology';
 schemaname | tablename | tableowner | tablespace | hasindexes | hasrules | hastriggers | rowsecurity
------------+-----------+------------+------------+------------+----------+-------------+-------------
 topology   | topology  | postgres   |            | t          | f        | t           | f
 topology   | layer     | postgres   |            | t          | f        | t           | f
(2 行)
```

空っぽなのですが、それぞれのテーブルを見てみましょう。

```
topologydb=# SELECT * FROM topology.topology;
 id | name | srid | precision | hasz
----+------+------+-----------+------
(0 行)


topologydb=# SELECT * FROM topology.layer;
 topology_id | layer_id | schema_name | table_name | feature_column | feature_type | level | child_id
-------------+----------+-------------+------------+----------------+--------------+-------+----------
(0 行)
```

## トポロジーの生成

トポロジーを生成します。[CreateTopology()](https://postgis.net/docs/ja/CreateTopology.html) を使います。ここでは``topo0``という名前のトポロジーを作ります。

```
SELECT topology.CreateTopology('topo0');
```

実際に実行すると、次のように返り値 (トポロジーID) が得られます。

```
topologydb=# SELECT topology.CreateTopology('topo0');
 createtopology 
----------------
              1
(1 row)
```

``topology.topology``にトポロジーが登録されているのを確認しましょう。

```
topologydb=# SELECT * FROM topology.topology;
 id | name  | srid | precision | hasz 
----+-------+------+-----------+------
  1 | topo0 |    0 |         0 | f
(1 row)
```

また、ノード、エッジ、フェイスの各データ用のテーブルが用意されます。スキーマが新たに作られますが、**スキーマ名はトポロジー名**です。``topo0``に作られたテーブル一覧を見てみましょう。

```
topologydb=# SELECT * FROM pg_tables WHERE schemaname='topo0';
 schemaname | tablename | tableowner | tablespace | hasindexes | hasrules | hastriggers | rowsecurity 
------------+-----------+------------+------------+------------+----------+-------------+-------------
 topo0      | face      | yellow     |            | t          | f        | t           | f
 topo0      | node      | yellow     |            | t          | f        | t           | f
 topo0      | edge_data | yellow     |            | t          | f        | t           | f
 topo0      | relation  | yellow     |            | t          | f        | t           | f
(4 rows)
```

## edge_dataテーブルとedgeビュー

ここで、``node``、``face``はいいのですが、エッジについては``edge``でなく``edge_data``となっています。

なぜでしょう?

``edge``は実は作られていますが実はビューです。ここではテーブルのみ見ているため``edge``は出てきません。そして、**``edge``は SQL/MM 互換性確保のために作られているにすぎません**。エッジデータを格納するために別途テーブルを作らないといけなくなったため、``edge_data``という名前のテーブルが作られることになっています。

# 順に実行していきましょう

## 最初はユニバーサルフェイスだけがある

最初の状態では、ノードなし、エッジなしです。フェイスは「ユニバーサルフェイス」という名前の仮想フェイスがひとつあります。

![初期状態](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/build/initial.png)

## ノードとエッジを追加する

エッジを追加するためには、始端と終端のノードを追加する必要があります。

まずはノードを追加してみましょう。[ST_AddIsoNode()](https://postgis.net/docs/ja/ST_AddIsoNode.html) を使います。

```
SELECT topology. ST_AddIsoNode('topo0', 0, 'POINT(0 0)');
SELECT topology. ST_AddIsoNode('topo0', 0, 'POINT(10 0)');
```

さらにエッジを追加してみましょう。[ST_AddEdgeModFace()](https://postgis.net/docs/ja/ST_AddEdgeModFace.html) を使います。

```
SELECT topology.ST_AddEdgeModFace('topo0', 1, 2, 'LINESTRING(0 0, 10 0)');
```

![2ノード1エッジの図](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/build/edge1.png)

## エッジを追加してフェイスを作る

フェイスはエッジで閉じた時点で生成されます。1本エッジを追加してみましょう。

```
SELECT topology.ST_AddEdgeModFace('topo0', 2, 1, 'LINESTRING(10 0, 15 5, 10 10, 0 10, 0 0)');
```

これでフェイスができあがります。

![1フェイス生成の図](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/build/face1.png)

## インターセクトするエッジを追加しようとしたらエラーが出た

次に、右側の5角形を作ろうと思います。ということでエッジを追加してみます。

```
SELECT topology.ST_AddEdgeModFace('topo0', 2, 2,'LINESTRING(10 0, 20 0, 20 10, 10 10, 15 5, 10 0)');
```

```
ERROR:  Spatial exception – geometry intersects edge 2
```

エラーが発生してしまいました。

![エラーが発生する図](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/build/intersects.png)

新規に追加しようとしたエッジと2番エッジがインタセクトする (共通部分を持つ) とエラーが発生します。

## エッジを分割してインターセクトしないようにエッジを追加する

2番エッジ途中に新規ノード (3番ノード) を追加したいですね。
その場合、「エッジを分割して、分割位置にノードを追加する」といいです。

次のクエリーで、2番エッジを``(10 10)``の位置で分割して、ノードを追加します。[ST_ModEdgeSplit()](https://postgis.net/docs/ja/ST_ModEdgeSplit.html) を使います。

```
SELECT topology.ST_ModEdgeSplit('topo0', 2, 'POINT(10 10)');
```

このとき、次のような操作が行われます。

+ ``(10 10)`` に3番ノードが新設されます。
+ 旧2番エッジが短縮され新2番エッジになります。旧2番エッジの始端を始端とし ``(10 10)`` を終端とします。
+ ``(10 10)`` を始端として、旧2番エッジの終端を終端とする新3番エッジが生成されますで分割されます。

2番ノードから新たにできた 3番ノードにエッジを渡すと、問題なくフェイスが出来上がります。

```
SELECT topology.ST_AddEdgeModFace('topo0', 2, 3, 'LINESTRING(10 0, 20 0, 20 10, 10 10)');
```

![左側凸5角形と右側凹5角形が接している図](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/build/all.png)

# QGISで見てみよう

QGISで見ることもできます。

PostGISレイヤーの追加をしようとすると、次のようなジオメトリーカラム選択ダイアログが表示されます。ここにトポロジーのジオメトリーカラムも見えます。

![QGISでのカラム選択ダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/build/qgis_dialog.png)

ここで``edge``にエクスクラメーションマークが出ていますが、``edge``は SQL/MM 互換性確保のためですので無視してください。

``node.geom``と``edge_data.geom``は、ジオメトリーのポイント、ラインストリングなので、問題なく表示されます。``face``は``mbr``となっている通り最小境界矩形なので、正確なポリゴンが表示されません。

表示すると次のようになります。

![QGISでのノード、エッジ、フェイスを表示している図](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/build/qgis_map.png)

ただし、``face.mbr``の塗りつぶし色の透過度は 50% にしていて、重複するところは濃くなっています。MBRベースなので、こういった重複がそこここで発生することになりますが、これは別にポリゴンではないで気にしないでください。

# おわりに

ここでは、ノードを打ってエッジを引くとフェイスができる様子を見ていただきました。あくまで概念的なものですので、クエリーを走らせてもあまり面白くないかもしれません。ただ、トポロジーはこういう動きをするんだよというところだけ見ていただけたらいいかなと思います。

QGISで見ることができて、ノード、エッジは正しく見ることができますが、フェイスはMBRなので正確なポリゴンは見ることができません。ただしこれは``edge_data.geom``を見れば形状の把握は可能です。

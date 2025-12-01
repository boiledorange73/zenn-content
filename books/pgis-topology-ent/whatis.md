---
title: "そもそもトポロジとは何やねん"
---

# トポロジとジオメトリの違い

|            |点|線|面|
|------------|--|--|--|
|図|![点の図、ひとつだけ点がある](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-ent/whatis/point.png)|![線の図、折れ線と両端に点がある](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-ent/whatis/line.png)|![面の図、ポリゴンと二つに点がある](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-ent/whatis/area.png)|
|ジオメトリー|POINT|LINESTRING|POLYGON|
|トポロジー|NODE (ノード)|EDGE (エッジ)|FACE (フェイス)|

空間次元が2次元までの場合には、図形には点、線、面があります。
ジオメトリでは、点はポイント、線はラインストリング、面はポリゴン、という語がそれぞれ使われていました。
トポロジでは、点はノード、線はエッジ、面はフェイスという語を使います。

どちらも点、線、面を表すのですが、全く違う言葉を使います。トポロジとジオメトリとは概念が異なるので、言い分けています。

違いは他の要素から独立しているかそうでないかです。

## ジオメトリは互いに独立している

PostGISで扱うジオメトリは、データ構造上は互いに独立しています。二つのポリゴンが辺を共有しているかどうかはテスト関数で得られますし、共有部分は幾何演算関数で得られます。ただしこれは、それらの関数を実行した時だけ有効です。

## ノード、エッジ、フェイスという単語

ノードは、エッジの境界が共有するポイントです。エッジを追加した際に他のエッジとのクロスが発生した場合には、クロスした位置にノードが追加され、エッジが分割されます。このため、エッジの途中にノードが存在することはありません。なお、ノードは二つ以上のエッジの端点となるか、一つのエッジの場合は、始点と終点が同じ場合の2端点となります。

エッジは、フェイスの境界が共有するラインストリングです。他のエッジと境界でノードを共有します。エッジを追加した際に、フェイスの分割が発生しますが、新たに発生した二つのフェイスの境界は、どちらも、このエッジを含みます。

フェイスはエッジに分割されたポリゴンです。他のフェイスと境界でエッジを共有しています。

## ノード、エッジ、フェイスは独立していない

フェイスはエッジを共有し、エッジはノードを共有していて、その共有関係は記録されています。このため、ノードとエッジ、エッジとフェイスが互いに直接的な関係を持ち、ノード同士、エッジ同士、フェイス同士は、間接的な関係を持ちます。

このため、エッジを削除すると、そのエッジを共有している複数のフェイスが一つのフェイスにまとまります。また、エッジの形状を変更すると、そのエッジを共有している全てのフェイスの形状が変更されます。

この、独立していない点が、ジオメトリと大きく異なる点です。

なお、ノードについては状況が異なり、複数エッジで共有されているノードは削除できません。また、PostGISの機能として、ノードの位置を変更してもエッジが変更されることがありません。

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


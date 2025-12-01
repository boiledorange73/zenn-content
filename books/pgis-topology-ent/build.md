---
title: "トポロジーを構築してみよう"
---

# はじめに


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

トポロジーを生成します。次の通りです。

```
SELECT topology.CreateTopology('topo0');
```

これで、``topo0``という名前のトポロジーが作られます。

# まだ


#

![](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-ent/build/initial.png)

```
SELECT topology. ST_AddIsoNode('topo0', 0, 'POINT(0 0)');
SELECT topology. ST_AddIsoNode('topo0', 0, 'POINT(10 0)');
SELECT topology.ST_AddEdgeModFace('topo0', 1, 2, 'LINESTRING(0 0, 10 0)');
```

![](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-ent/build/edge1.png)

```
SELECT topology.ST_AddEdgeModFace('topo0', 2, 1, 'LINESTRING(10 0, 15 5, 10 10, 0 10, 0 0)');
```

![](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-ent/build/face1.png)

```
SELECT topology.ST_AddEdgeModFace('topo0', 2, 2,'LINESTRING(10 0, 20 0, 20 10, 10 10, 15 5, 10 0)');
```
```
ERROR:  Spatial exception – geometry intersects edge 2
```
![](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-ent/build/intersects.png)

```
SELECT topology.ST_ModEdgeSplit('topo0', 2, 'POINT(10 10)');
SELECT topology.ST_AddEdgeModFace('topo0', 2, 3, 'LINESTRING(10 0, 20 0, 20 10, 10 10)');
```
![](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-ent/build/all.png)


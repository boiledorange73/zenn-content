---
title: "そもそもトポロジとは何やねん"
---

# ジオメトリは互いに独立している

PostGISで扱うジオメトリは、構造上は互いに独立しています。
二つのポリゴンが辺を共有しているかどうかはテスト関数で得られますし、共有部分は幾何演算関数で得られます。
ただしこれは、それらの関数を実行した時だけ有効です。

# ノード、エッジ、フェイスという単語

ジオメトリでは、ポイント、ラインストリング（またはライン）、ポリゴン、という語がつかわれていました。
トポロジでは、点はノード、線はエッジ、面はフェイスと言います。
ジオメトリと異なる概念となるので、言い分けているものと思います。

## ノード

ノードは、どのエッジに

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


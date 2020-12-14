---
title: "トポジオメトリで属性を持つ"
---

# はじめに

ノード、エッジ、フェイスたちは、属性を持っていません。

ジオメトリ自体も属性を持っていませんが、テーブルの任意のカラムにジオメトリを指定できるので、結局はジオメトリとそれ以外のカラムとの組み合わせで、ジオメトリに属性を持たせることができます。しかし、トポロジの場合には、個々のノードやエッジやフェイスのデータは``node``、``edge``、``face``の各テーブルに存在します（フェイスに至っては``face``だけでは図形としての情報が不足しています）が、各テーブルはスキーマは固定しています。このため、属性を示すカラムと図形との組み合わせを作ることができません。

これは結構困ったものです。たとえば、市区町村ポリゴンからフェイスを生成しても、フェイス自体には市町村名や自治体コードを持つことができないということになります。

ここで、トポジオメトリというものを使います。トポジオメトリは、トポロジのテーブルとは別に、ジオメトリのように見せるカラムを任意に作ることができます。これにより、ジオメトリと同じように、トポジオメトリと属性の組み合わせが生成でき、トポロジをジオメトリのように扱うことができ、ちょうどトポロジの要素をジオメトリカラムと同じような使い勝手にできるようになります。

ここでは、トポジオメトリ使用して、属性を持つフェイスを生成してみます。

# トポジオメトリを作る

## dbデータベースを使います

[PostGISを入れたらやってみよう](../../b1de0a18073af70946e0)の[シェープファイルのデータをインポートしてみよう コマンドライン編](../../b1de0a18073af70946e0/viewer/import-cli)で作った``t1``をもとにしてみましょう。

以前はPostGISエクステンションしか入っていないはずですので、PostGISトポロジのエクステンションを追加します。

```
db=# CREATE EXTENSION postgis_topology;
CREATE EXTENSION
```

## トポロジとトポジオメトリの生成

まず、トポロジを生成します。空間参照系を``EPSG:3857``（Webメルカトル）にし、名前を``'topo_t1_3857``としました。

```
db=# SELECT topology.CreateTopology('topo_t1_3857', 3857);
 createtopology
----------------
              1
(1 行)
SELECT topology.CreateTopology('topo_t1_3857', 3857);
```

さらに、トポジオメトリを持つテーブルを用意します。

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

![publicスキームレイヤ選択ダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/topogeom/01-layerdialog.png)

publicスキーマに``topogeom_t1_3857.topo``があります。これがトポジオメトリカラムです。なお、トポジオメトリとしても、ジオメトリとしても、候補に挙がっていますが、一方だけでいいです。

![topogeom_t1_3857.topoの表示例](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/topogeom/02-topo.png)

普通に表示されました。この「普通」というのがポイントで、トポロジのノード、エッジ等の共有状況とは全く関係ないようにジオメトリだけが見えています。

さらに、地物の属性情報を見てみましょう。

![topogeom_t1_3857.topoと選択地物の属性の表示例](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/topogeom/03-attr.png)

このように、地物ごとの属性情報も問題なく見ることができます。

トポロジには属性の概念が無かったのですが、トポジオメトリの場合には、属性を持たせられることがお分かりいただけたと思います。

# トポロジも見てみる

続いて、トポロジの本体である``topo_t1_3857``スキームを見てみます。

![topo_t1_3857スキームレイヤ選択ダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/topogeom/04-layerdialog-topology.png)

テーブル名``edge_data``を追加した後で``node``を追加します。両方ともカラム名は``geom``です。

![ノードとエッジの表示例](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/topogeom/05-node_edge.png)

こんな感じで、エッジとノードが出現します。ノードは、エッジが交わる/分岐する点にだけ出ているのが分かると思います。

ついでにフェイスも見てみましょう。こちらは、カラム名が``mbr``ということで、長方形になるだろうと想像できます。

![フェイスの表示例](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/topogeom/06-face.png)

長方形でした。

# おわりに

トポジオメトリを作る方法についてご紹介しました。

トポジオメトリを使うと、トポロジのノード、エッジ、フェイスの関係を構築したうえで、属性を持つジオメトリ（ポリゴン）としても見ることができることが確認できたと思います。

次回には、トポロジの利点を見て頂けると思います。


# 出典
本記事では、国土交通省国土政策局が発行する国土数値情報（行政区域）を利用しました。


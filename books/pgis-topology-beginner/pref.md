---
title: "全国の市区町村ポリゴンから都道府県レイヤーを作っちゃおう！"
---

# はじめに

これまでの総仕上げとして、地区町村レイヤー(トポジオメトリー)を作成し、都道府県レイヤーを子レイヤーとした階層構造を作ります。

# 市区町村ポリゴンの入手と格納

https://nlftp.mlit.go.jp/ksj/ から取得 
2025年版から N03-20250101.geojson を取り出しPGDUMPに変換して psql から叩き込みます。

(``%``は tcsh のプロンプトです)

```
% ogr2ogr -f PGDUMP --config PG_USE_COPY YES \
  -lco SPATIAL_INDEX=GIST -lco GEOMETRY_NAME=geom -lco FID=gid \
  -nln raw_mncpl raw_mncpl.sql N03-20250101.geojson

% createdb mncpl
% psql -d mncpl -c "CREATE EXTENSION postgis_topology cascade"
% psql -d mncpl -f raw_mncpl.sql
```

# レイヤーの作成

ここからは ``psql`` を使います。

## 市区町村レイヤー

まず、市区町村レイヤーを作成します。最初にテーブルとトポジオメトリーカラムの作成です。

```
SELECT topology.CreateTopology('tp_mncpl', 3395);

CREATE TABLE tg_mncpl (
  gid SERIAL PRIMARY KEY,
  mcode INT
);
CREATE INDEX ON tg_mncpl(mcode);
SELECT topology.AddTopoGeometryColumn(
        'tp_mncpl', 'public', 'tg_mncpl', 'tg', 'POLYGON');
```

次に全国市区町村トポジオメトリーを挿入します。

```
EXPLAIN ANALYZE
INSERT INTO tg_mncpl (mcode, tg)
SELECT n03_007::INT,
  topology.toTopoGeom(ST_Transform(geom,3395), 'tp_mncpl', 1, 0)
FROM raw_mncpl;
```

結果は次の通りです。1時間20分かかりました。

```
                                                              QUERY PLAN                                                               
---------------------------------------------------------------------------------------------------------------------------------------
 Insert on tg_mncpl  (cost=0.00..1593352.38 rows=0 width=0) (actual time=4702389.713..4702390.042 rows=0 loops=1)
   ->  Subquery Scan on "*SELECT*"  (cost=0.00..1593352.38 rows=124094 width=40) (actual time=56.401..4631827.037 rows=124094 loops=1)
         ->  Seq Scan on raw_mncpl  (cost=0.00..1592731.91 rows=124094 width=36) (actual time=56.237..4411531.278 rows=124094 loops=1)
 Planning Time: 0.216 ms
 Trigger for constraint tg_mncpl_mcode_fkey: time=1502.695 calls=124094
 JIT:
   Functions: 3
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 1.771 ms (Deform 1.148 ms), Inlining 10.347 ms, Optimization 21.288 ms, Emission 18.166 ms, Total 51.571 ms
 Execution Time: 4704780.248 ms
(10 rows)
```

## 都道府県レイヤー

続いて、都道府県レイヤーを作成します。市区町村レイヤーと同じく、テーブルとトポジオメトリーカラムを作成します。

```
CREATE TABLE tg_pref (
  gid SERIAL PRIMARY KEY,
  pcode INT
);
CREATE INDEX ON tg_pref(pcode);

SELECT topology.AddTopoGeometryColumn(
  'tp_mncpl', 'public', 'tg_pref', 'tg', 'POLYGON',1
);
```

市区町村レイヤーのトポジオメトリーから全国都道府県トポジオメトリーを作成します。

```
EXPLAIN ANALYZE
INSERT INTO tg_pref (pcode, tg)
SELECT mcode/1000 pcode, topology.CreateTopoGeom('tp_mncpl', 3, 2, topology.TopoElementArray_Agg(TopoElement(tg)))
FROM tg_mncpl
GROUP BY mcode/1000;
```

結果は次の通りです。8秒程度で終わりました。

都道府県レイヤーのトポジオメトリーはただ単に「〇〇県は××市と△△市と〇×町と…のマルチポリトンを使っている」ことを格納しているにすぎません。

これをもう少し掘り下げると、「〇〇県はフェイスIDでいうと〇番と△番と…を使っている」情報を収めているにすぎません。

それだけなので、市区町村ポリゴンから都道府県ポリゴンを作る際にはポリゴンの結合を行う必要があるけれども、ここでは使っていないので、ジオメトリー演算が一切ないのが高速になっている理由だろうと思います。

```
                                                              QUERY PLAN                                                               
-------------------------------------------------------------------------------------------------------------------------------------
 Insert on tg_pref  (cost=34355.15..34600.98 rows=0 width=0) (actual time=8215.503..8215.932 rows=0 loops=1)
   ->  Subquery Scan on "*SELECT*"  (cost=34355.15..34600.98 rows=919 width=40) (actual time=3691.731..8208.851 rows=47 loops=1)
         ->  HashAggregate  (cost=34355.15..34596.38 rows=919 width=36) (actual time=3691.490..8191.960 rows=47 loops=1)
               Group Key: (tg_mncpl.mcode / 1000)
               Batches: 1  Memory Usage: 1096kB
               ->  Seq Scan on tg_mncpl  (cost=0.00..2711.18 rows=124094 width=44) (actual time=0.084..1297.039 rows=124094 loops=1)
 Planning Time: 0.247 ms
 Trigger for constraint tg_pref_pcode_fkey: time=0.983 calls=47
 Execution Time: 8217.666 ms
(9 rows)
```

## インデックスが効かない

トポジオメトリーはインデックスが効きません。次でお示しする QGIS による表示は、とんでもなく時間がかかります。

# QGISで見たけど見ない方がいい

QGISで見てみましょう。

PostGISレイヤーの追加をしようとすると、次のようなダイアログが見えます。トポジオメトリーカラムに対応してくれています。やったぜ。

![QGISで見た mncpl のカラム一覧](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/pref/qgis_dialog.png)

ということで市区町村レイヤー見てみましょう。

…**とんでもなく遅い**。表示に**2分20秒**かかりました。インデックスが効かないので、拡大表示しても同じだけ時間がかかります。そして、**少しスクロールしただけでも**、レイヤー全体をなめるクエリーを出すので、**再表示までに時間がかかります**。でも見れるのは見れるので、一度ぐらい見てみてもいいかも知れません。

![市区町村レイヤー](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/pref/mncpl.png)

続いて、戸津府県レイヤーも見てみましょう。**1分44秒**かかりました。ましになりましたね…と言うには微妙でした。

![都道府県レイヤー](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/pref/pref.png)

もう少し拡大して (ありえんぐらいに表示に時間がかかるけど) 地物をクリックして属性データが表示されるか確認してみましょう。

まずは市区町村です。``mcode``しかなくて豪華さは全くないですけど、いろいろ付け足せます。

![市区町村レイヤー](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/pref/mncpl_large.png)

次に都道府県です。``pcode``しかないですが以下同文。

![都道府県レイヤー](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/pref/pref_large.png)

# ジオメトリーにする

最後に、都道府県ジオメトリーテーブルを作成してジオメトリーデータを挿入してみましょう。インデックスが効きます。

```
CREATE TABLE g_pref (
  gid SERIAL PRIMARY KEY,
  pcode INT,
  geom GEOMETRY(POLYGON, 6668)
);
CREATE INDEX ON g_pref (pcode);
CREATE INDEX ON g_pref USING GiST (geom);

EXPLAIN ANALYZE
INSERT INTO g_pref(pcode, geom)
SELECT pcode, ST_Transform((ST_Dump(tg::GEOMETRY)).geom, 6668)
FROM tg_pref;
```

トポジオメトリーからジオメトリーに変換しているのが ``tg::GEOMETRY``のところです。キャストでジオメトリーにできるので、気軽に使ってみてください。

# おわりに

市区町村ポリゴンから市区町村レイヤーを作って、さらに市区町村レイヤーから都道府県レイヤーを作りました。

対象が違っても、おそらくこういう手続きを踏んでいくのだろうと思います。

トポロジーを使わないのでしたらそれに越したことはないですが、トポロジーを使うのでしたら参考になると思います。

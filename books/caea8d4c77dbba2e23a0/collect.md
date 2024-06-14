---
title: "結合せずまとめる"
---
# はじめに

[QGISでPostGISのデータを見てみよう](https://zenn.dev/boiledorange73/books/b1de0a18073af70946e0/viewer/qgis)の最後の画像では、ひとつの市区町村名が複数出現しているのが見えます。

もう少し見てみましょう。

![インポートしたデータで福山市の島を選択した様子](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/collect/1.png)

「福山市」がたくさん出ているうえ、ひとつの島を選択しています（福山市内海町に属します）が、本土側も仙酔島も何もかも選択されていません。

これは、ひとつの市区町村に対して複数のポリゴンから成っているためです。これを市区町村ごとにひとつのマルチポリゴンにしたいとします。

## ST_Unionは使わない

[ジオメトリの足し算、引き算をする](https://zenn.dev/boiledorange73/books/caea8d4c77dbba2e23a0/viewer/editor)をご覧頂いた方は、``ST_Union``という関数を思い出すことでしょう。

しかし、ジオメトリ演算は結構なコストがかかります。

国土数値情報（行政区域）は非常に「きれい」なポリゴンで、複数のポリゴンから成っていますが、離島や飛び地を形成するものだけで、ひとつのポリゴンで表現できる領域を不自然に分割するといったことがありません。

こういった場合に限っては、複数のポリゴンを単純にひとつのマルチポリゴンに「まとめる」だけの関数を使った方が計算コストを省くことができます。

ここでは、``ST_Collect``を使って「まとめる」方法を紹介します。

## 使用する関数

``ST_Collect``

# ST_Collect

``ST_Collect``は、単純にまとめるだけです。``ST_Union``の場合は、インタセクションがあればひとつのポリゴンに作り直してくれます。

``ST_Union``の挙動を確認してみましょう。

```psql
db=# SELECT ST_AsText(
  ST_Union(
    'POLYGON((0 0, 2 0, 2 2, 0 2, 0 0))'::GEOMETRY,
    'POLYGON((1 1, 3 1, 3 3, 1 3, 1 1))'::GEOMETRY
  )
);
                   st_astext                    
------------------------------------------------
 POLYGON((2 1,2 0,0 0,0 2,1 2,1 3,3 3,3 1,2 1))
(1 行)
```

続いて``ST_Collect``を見てみましょう。

```psql
db=# SELECT ST_AsText(
  ST_Collect(
    'POLYGON((0 0, 2 0, 2 2, 0 2, 0 0))'::GEOMETRY,
    'POLYGON((1 1, 3 1, 3 3, 1 3, 1 1))'::GEOMETRY
  )
);
                           st_astext                           
---------------------------------------------------------------
 MULTIPOLYGON(((0 0,2 0,2 2,0 2,0 0)),((1 1,3 1,3 3,1 3,1 1)))
(1 行)
```

単に引数になっているポリゴンをまとめてマルチポリゴンにしているだけなのが見えるかと思います。

## 返り値のジオメトリタイプに注意

```psql
db=# SELECT ST_AsText(
  ST_Collect(
    'MULTIPOLYGON(((0 0, 2 0, 2 2, 0 2, 0 0)))'::GEOMETRY,
    'MULTIPOLYGON(((1 1, 3 1, 3 3, 1 3, 1 1)))'::GEOMETRY
  )
);
                                            st_astext                                            
-------------------------------------------------------------------------------------------------
 GEOMETRYCOLLECTION(MULTIPOLYGON(((0 0,2 0,2 2,0 2,0 0))),MULTIPOLYGON(((1 1,3 1,3 3,1 3,1 1))))
(1 行)
```

``ST_Collect``は、先ほどから述べていますが、引数として渡されたジオメトリを単純にまとめることしかしません。複数ある単一ジオメトリをまとめてひとつのマルチジオメトリにするのはごく簡単なのでやってくれますが、複数のマルチジオメトリをまとめる際に、いちど単一ジオメトリにバラしてひとつのマルチジオメトリにまとめなおす、といったことはしません。引数で渡されたマルチジオメトリを要素とした、ひとつのジオメトリコレクションを返してきます。

ジオメトリコレクションは扱いにくいジオメトリタイプです。PostGISのマニュアルを見ていると、「この関数はジオメトリコレクションを引数にすることができません」というようなことばを頻繁に目にします。ジオメトリコレクションはできるだけ避けるべきです。

この場合、``ST_Dump``を使って単一ポリゴンを返すサブクエリから得たジオメトリを``ST_Collect``に渡すと、返り値はジオメトリコレクションではなくなります。

```psql
db=# SELECT ST_AsText(ST_Collect(dumped)) FROM (
  SELECT (ST_Dump(geom)).geom AS dumped FROM (
    SELECT 'MULTIPOLYGON(((0 0, 2 0, 2 2, 0 2, 0 0)))'::GEOMETRY AS geom
    UNION
    SELECT 'MULTIPOLYGON(((1 1, 3 1, 3 3, 1 3, 1 1)))'::GEOMETRY AS geom
  ) AS Q1
) AS Q2;
                           st_astext                           
---------------------------------------------------------------
 MULTIPOLYGON(((0 0,2 0,2 2,0 2,0 0)),((1 1,3 1,3 3,1 3,1 1)))
(1 行)
```


# 国土数値情報（行政区域）の広島県データに対してやってみる

[マルチポリゴンを単一ポリゴンに変換する](https://zenn.dev/boiledorange73/books/caea8d4c77dbba2e23a0/viewer/dump)でt1dを生成したので、これを市区町村ごとに集め、t1dcに叩き込みます。

なお、テーブル名が``t1u``の場合には、``alter table t1u rename to t1d;``で、``t1d``に変更しておいて下さい。

```psql
CREATE TABLE t1dc (
  gid SERIAL PRIMARY KEY,
  n03_001 TEXT,
  n03_002 TEXT,
  n03_003 TEXT,
  n03_004 TEXT,
  n03_007 TEXT,
  geom GEOMETRY(MULTIPOLYGON, 4612)
);
CREATE INDEX ix_t1dc_geom ON t1dc USING GiST (geom);

db=# INSERT INTO t1dc(n03_001, n03_002, n03_003, n03_004, n03_007, geom)
  SELECT n03_001, n03_002, n03_003, n03_004, n03_007, ST_Collect(geom)
  FROM t1d
  GROUP BY n03_001, n03_002, n03_003, n03_004, n03_007;
```

QGISに出すと、次のようになりました。

![ST_Collectを実行した結果](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/collect/2.png)

ひとつのジオメトリを選択していますが、福山市全域が選択されました。これで福山市全域がひとつのマルチポリゴンになっていることが分かります。

## マルチポリゴンからST_Collectを実行する例

[シェープファイルのデータをインポートしてみよう コマンドライン編](https://zenn.dev/boiledorange73/books/b1de0a18073af70946e0/viewer/import-cli)で``t1``を生成しましたが、``t1``はマルチポリゴンですから、``t1``から直接``ST_Collect``を実行すると、上述のジオメトリコレクションの問題に直面します。

試してみましょう。まず、テーブルを作ります。

```psql
CREATE TABLE t1_c (
  gid SERIAL PRIMARY KEY,
  n03_001 TEXT,
  n03_002 TEXT,
  n03_003 TEXT,
  n03_004 TEXT,
  n03_007 TEXT,
  geom GEOMETRY(MULTIPOLYGON, 4612)
);
CREATE INDEX ix_t1_c_geom ON t1_c USING GiST (geom);
```

エラーを出してみましょう。

```psql
db=# INSERT INTO t1_c(n03_001, n03_002, n03_003, n03_004, n03_007, geom)
  SELECT n03_001, n03_002, n03_003, n03_004, n03_007, ST_Collect(geom)
  FROM t1
  GROUP BY n03_001, n03_002, n03_003, n03_004, n03_007;
ERROR:  Geometry type (GeometryCollection) does not match column type (MultiPolygon)
```

``ST_Dump``で単一ポリゴンにバラしたうえで``ST_Collect``に渡すとマルチジオメトリを返してくれます。次のようになります。

```psql
db=# INSERT INTO t1_c(n03_001, n03_002, n03_003, n03_004, n03_007, geom)
  SELECT n03_001, n03_002, n03_003, n03_004, n03_007, ST_Collect(dumped) FROM (
    SELECT n03_001, n03_002, n03_003, n03_004, n03_007, (ST_Dump(geom)).geom AS dumped
    FROM t1
  ) AS Q
  GROUP BY n03_001, n03_002, n03_003, n03_004, n03_007;
```

# ST_CollectとST_Unionとの速度差

``ST_Collect``と``ST_Union``との速度を順に見ていきましょう。

ただし、この実験では、全国の市区町村データを使用しています。広島県だけだともっと短縮されると思います。

```psql
db=# TRUNCATE t1dc;
TRUNCATE TABLE
db=# EXPLAIN ANALYZE INSERT INTO t1dc(n03_001, n03_002, n03_003, n03_004, n03_007, geom)
  SELECT n03_001, n03_002, n03_003, n03_004, n03_007, ST_Collect(geom)
  FROM t1d
  GROUP BY n03_001, n03_002, n03_003, n03_004, n03_007;
                                                           QUERY PLAN                                                            
---------------------------------------------------------------------------------------------------------------------------------
 Insert on t1dc  (cost=5670.83..5890.78 rows=7332 width=82) (actual time=24356.407..24356.407 rows=0 loops=1)
   ->  Subquery Scan on "*SELECT*"  (cost=5670.83..5890.78 rows=7332 width=82) (actual time=394.384..2608.734 rows=1911 loops=1)
         ->  HashAggregate  (cost=5670.83..5762.48 rows=7332 width=2080) (actual time=394.339..512.687 rows=1911 loops=1)
               ->  Seq Scan on t1d  (cost=0.00..4571.13 rows=73313 width=2080) (actual time=0.014..26.897 rows=73313 loops=1)
 Total runtime: 24380.198 ms
(5 行)

db=# TRUNCATE t1dc;
TRUNCATE TABLE
db=# EXPLAIN ANALYZE INSERT INTO t1dc(n03_001, n03_002, n03_003, n03_004, n03_007, geom)
  SELECT n03_001, n03_002, n03_003, n03_004, n03_007, ST_Multi(ST_Union(geom))
  FROM t1d
  GROUP BY n03_001, n03_002, n03_003, n03_004, n03_007;
                                                            QUERY PLAN                                                            
----------------------------------------------------------------------------------------------------------------------------------
 Insert on t1dc  (cost=5670.83..5909.11 rows=7332 width=82) (actual time=88274.766..88274.766 rows=0 loops=1)
   ->  Subquery Scan on "*SELECT*"  (cost=5670.83..5909.11 rows=7332 width=82) (actual time=265.584..77405.919 rows=1911 loops=1)
         ->  HashAggregate  (cost=5670.83..5780.81 rows=7332 width=2080) (actual time=265.566..77373.000 rows=1911 loops=1)
               ->  Seq Scan on t1d  (cost=0.00..4571.13 rows=73313 width=2080) (actual time=0.006..17.999 rows=73313 loops=1)
 Total runtime: 88296.701 ms
(5 行)
```

``ST_Collect``の方が圧倒的に早く、全体で3.6倍、サブクエリ分をのぞくと4.4倍程度の速度差がありました。

# ST_Unionが要らないだなんて言ってはいけない

最初にあげた``ST_Collect``の実行例で、インタセクションがあろうともそのままマルチポリゴンにしているのを示しました。多くの場合、マルチポリゴンの要素間でインタセクションがあるのは許せないかと思います。元データがきれいでないなら、ST_Unionを使うべきです。

また、市区町村ポリゴンを結合して都道府県ポリゴンを生成する場合などの、本当に加工が必要な場合には、``ST_Collect``は使えません。

# おわりに

いくつかのマルチ/単一ポリゴンをまとめてマルチポリゴンを生成する場合で、``ST_Union``のようなジオメトリ演算が不要な場合には、``ST_Collect``を使えることを紹介しました。また、``ST_Collect``はジオメトリのインタセクションに全く配慮せず、ジオメトリコレクションを返そうとするのですが、その代わり、速度は圧倒的に``ST_Collect``の方が早くなることも示しました。

状況に応じて使い分けてください。

# 出典

地図の作成にあたっては、国土数値情報（行政区域）を使用しました。
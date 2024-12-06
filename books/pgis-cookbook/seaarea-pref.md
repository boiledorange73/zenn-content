---
title: "ボロノイ分割で「海域それっぽい都道府県界ポリゴン」をでっち上げる"
---
# はじめに

「国土数値情報（行政区域）」などで、都道府県・市区町村ポリゴンが取得できます（境界未確定については暫定的なものです）が、海上での境界を示すデータは存在しません。

あくまで私の想像ですが、陸地のように登記は行わないので管轄都道府県・市町村が存在しないのではないかと思います。埋め立てが始まる段階になって帰属が問題になる事例はよくあります、某都特別区間とか、某県の県庁所在市と南隣の火発を持っていた市との間とか。

一般的な地図では不要（場合によっては害悪）なものとは思いますが、それでも、かなり前から海域における市区町村の境界を表現するポリゴンを作りたいと思っていました。

かつて、某サーバで、緯度経度からその位置はどの市区町村に入るかを判定する計算していました（過去形）。PostGIS を使うとすごく簡単で、市区町村ポリゴンがあれば、``ST_WithIn()``を使った問合せ一つだけで済みました。また、空間インデックスの威力が絶大で、計算速度も非常に速いものでした。

しかし、GNSS単独測位の場合には、位置が多少ずれることを想定しないといけません。海岸付近にいる場合には、少しずれると海域に出たことになってしまいます。都道府県ポリゴンは陸域に限定されるので、GNSSによって得る位置が**海域に少しでもずれると、とたんに計算ができなくなります**。

![位置がずれると困ること](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/19-part-pref.png)

これの対策としては、海岸線から適切な距離でバッファを取るのが考えられますが、こんなかんじになります。

![10mでバッファを作って、一部重なっている箇所をQGISで表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/20-part-buff10m.png)

上の図では、バッファレイヤを #F00、透過度50% として表示していますが、重なっている箇所があると、ピンク色でなく、より濃い色になります。で、実際に濃い色が出ている通り、重なっている部分ができてしまっています。

運悪く、重なっている地域にGNSS測位結果が入り込むと、どちらの県に属するかが不定となります（上の図で言うと「広島」なのか「岡山」なのか分かりません）。

それでも問題になる条件限定的ですので、わざわざする必要は無いと思います。それでもほら、**寝覚めが悪い**じゃないですか。

前述の某サーバをやっている頃は、考えては試して失敗して長期間放置、また思い出しては試して失敗、というのを 10年ぐらいやってきていると思います。少なくとも ``ST_VoronoiPolygons()`` が導入される 2016年9月26日 より前から考えてはいました。

ということは、「**構想8年から10年ほど**」なのです（範囲が広すぎる）。

もうその計算をする必要も無くなったのですが、ふと先日、「PostGIS 料理帳」のネタになるかと思い立って、試しに陸域の都道府県ポリゴン（市区町村ではありません）から「海域それっぽい都道府県界ポリゴン」をでっち上げてみたところ、それなりにでっち上げることができました。まだ良くないですけど、構想8年から10年（**しつこい**）かかったことを記念して、また、思ったより大作になってしまったので自分用メモの意味としても、ここに記します。

## ご注意

このポリゴンはあくまで「それっぽい」ものとして作成したもので、都道府県、市区町村の管轄に関する取り決めを一切考慮に入れていませんし、そもそもどういう取り決めがされているのかも存じません。

いちおう念のため申しますが、このデータには権威は一切なく、参考程度に扱って下さい。

# 必要なデータ

国土交通省国土政策局が提供する「国土数値情報 (行政区域)」を使います。最近は、市区町村ポリゴンと都道府県ポリゴンが同梱されてて、ちょっとビックリしました。

  * PostGIS料理帳 / [国土数値情報のダウンロード](dl-seaarea-border)

# PGDUMPデータ作成とインポート

[PostGISをはじめてみよう](../../b1de0a18073af70946e0) / [シェープファイルのデータをインポートしてみよう コマンドライン編](../../b1de0a18073af70946e0/viewer/import-cli) を参照して下さい。

ここでは ``shp2pgsql`` を使っていますが、次のようなかんじに、PostgreSQLの ``bin`` フォルダに入っています。``ogr2ogr`` でも、もちろん構いません。

```
"c:\Program Files\PostgreSQL\16\bin\"shp2pgsql -s 6668 -S -D -I N03-20240101.shp raw_mncpl > raw_mncpl.sql
"c:\Program Files\PostgreSQL\16\bin\"shp2pgsql -s 6668 -S -D -I N03-20240101_prefecture.shp raw_pref > raw_pref.sql
```

作業データベースは、今回は``voro``というデータベースを専用に作成することにしました。

データベース作成とともに ``postgis``と``postgis_topology``のエクステンション作成も忘れないでください。

```
"c:\Program Files\PostgreSQL\16\bin\"createdb -U postgres voro
"c:\Program Files\PostgreSQL\16\bin\"psql -U postgres -d voro -c "CREATE EXTENSION postgis"
"c:\Program Files\PostgreSQL\16\bin\"psql -U postgres -d voro -c "CREATE EXTENSION postgis_topology"
```

``psql``からインポートする場合は、次のようにします。

```
"c:\Program Files\PostgreSQL\16\bin\"psql -U postgres voro
voro=# \i raw_mncpl.sql
voro=# \i raw_pref.sql
```

## geometry_columns で確認

ジオメトリタイプと座標参照系(空間参照系とも)を確認しておきましょう。``geometry_columns``というビューで確認できます。

座標参照系については [PostGIS入門](../../caea8d4c77dbba2e23a0) / [空間参照系の概要](../../caea8d4c77dbba2e23a0/viewer/srs) などをご覧下さい。

```
voro=# select * from geometry_columns;
 f_table_catalog | f_table_schema |       f_table_name       | f_geometry_column | coord_dimension | srid |     type
-----------------+----------------+--------------------------+-------------------+-----------------+------+--------------
 voro            | public         | raw_mncpl                | geom              |               2 | 6668 | POLYGON
 voro            | public         | raw_pref                 | geom              |               2 | 6668 | POLYGON
...
```

EPSG:6668 (JGD 2011 地理座標系)のポリゴンであることが確認できました。

## QGISで見てみる

都道府県ポリゴンを QGIS で見てみましょう。

操作方法については [PostGISをはじめてみよう](../../b1de0a18073af70946e0) / [空間参照系の概要](../../b1de0a18073af70946e0/viewer/qgis) などをご覧下さい。

![QGISでraw_mncplの地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/01-pref.png)

ここまでは問題ないです。

# 方針

海域での境界を求めるには、二つの都道府県が相対する海岸線の中線を引いていくといいはずですが、その中線をどうやって引くかが問題です。

中線の定義がどうなるかはハッキリとは分からないですが、相対する海岸線から等距離に当たる点の集合と言えるのではないでしょうか。そうすると、ボロノイ分割が近いのではないかと考えました。

## ボロノイ分割とは

ボロノイ分割は、複数のポイント（複数の母点）から、どのポイントに最も近いかで領域を分けることを指します。また、分割した結果の図をボロノイ図といいます。

母点間で等距離になる点の集合（線）で面を分割することです。ボロノイ分割で生成される細胞状のポリゴンは、必ずその中に母点を一つ包含し、その母点が全ての母点の中で最も近い点の集合となります。

PostGISでの利用法の例として PostGIS料理帳 / [ボロノイ分割でおおざっぱな消防署の管轄範囲を作ろう](fire-voronoi) をご覧下さい。

``ST_VoronoiPolygon()``等のPostGISでのボロノイ分割は、マルチポイントを引数に与えると、各要素のポイントを母点として、ボロノイ分割を行ってくれます。

## 海岸線でなく海岸線を構成する点ならどうだろう

ボロノイ分割は、母点に対して行うので、海岸線を引数に与えることはできません。そこで、海岸線自体ではなく、海岸線を構成する点を与えてボロノイ分割を送るといいのではないかと考えました。

図で示すとこんな感じになります。

![ボロノイポリゴンから「海域それっぽい都道府県界ポリゴン」をでっち上げた例](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/21-voronoi.png)

上図で、オレンジ色が広島県、緑色が愛媛県だとします。海岸線にある多数の点からボロノイ分割を行い、それぞれのボロノイポリゴンには一つの母点を持つので、その母点の都道府県コードをポリゴンの都道府県コードとします。

そうしてできた薄いオレンジと薄い緑色が広島県と愛媛県の海域のポリゴンです。

これを都道府県コードで ``GROUP BY`` して、集約関数 ``ST_Union()`` にかけると、「海域それっぽい都道府県界ポリゴン」が完成、かも知れません。


## 海岸線構成点を減らしたうえでボロノイ分割にする

他に考えつかなかったので（えらい消極的やな）、海岸線を構成する点でボロノイ分割を行うこととしました。ただし、海岸線を構成しない都道府県境界は使用しません。

あと、途中のことは書いていませんが、国土数値情報（行政区域）をそのまま ``ST_VoronoiPolygons()`` に適用すると、``GEOSError``が出てきました。
じゃあ減らそうと思ったのですが、そのまま ``ST_Simplify()`` 等を使うと、隙間ができたり、重複ができたりと大変なので、トポロジを使いました。

## 「やってみました」程度にとらえて下さい

この手法が正解かどうかは分かりません。それっぽいものができたとは思いますが、問題もありました。あくまで、ひとつのやり方を考えてやってみました、程度にとらえて下さい。

# 都道府県コードと都道府県名のテーブルの作成

作業中は都道府県コードだけを付けておいて、最後に都道府県名を追加するようにするので、都道府県コードから都道府県名を検索できるテーブルを用意しておきます。

後で使うので、今でなくていいですが、とりあえず作っておきます。

また、都道府県名を最終成果物に入れるつもりがないのでしたら、不要です。

```
CREATE TABLE t_pref (
  pcode TEXT PRIMARY KEY,
  pname TEXT
);
CREATE INDEX ON t_pref(pname);
INSERT INTO t_pref SELECT DISTINCT n03_007, n03_001 FROM raw_pref ORDER BY n03_007;
```

# 日本ポリゴン作成

都道府県ポリゴンを結合して日本ポリゴン（都道府県ごとには分けない）を作ります。
後で使うので、今でなくていいですが、とりあえず作っておきます。

```
CREATE TABLE jp (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(POLYGON, 6668)
);
CREATE INDEX ON jp USING GiST (geom);

WITH
  mlt AS (SELECT ST_Union(geom) AS geom FROM raw_pref)
INSERT INTO jp(geom)
SELECT (ST_Dump(geom)).geom FROM mlt;
```

``ST_Union()`` でまとめると ``GROUP BY`` が指定されていないので、都道府県ポリゴンの全てを結合して、重ならないものは別個のポリゴン要素にして、マルチポリゴンを生成します。``mlt`` がそれです。続いて、``ST_Dump()`` でマルチポリゴンをバラして、(シングル)ポリゴンにします。


# 領海(よりやや広い領域)ポリゴンを作る

海域の都道府県境界は、かなり遠方の、たとえば200海里ぐらいになってくると、どの国の管轄かは重要だとしても、どの県に最も近いかは、まあどうでも良くなってきます。

今回は領海ぐらいに留めた都道府県境界を作ろうと思います。

そのためには、領海ポリゴンを作り必要があります。ただ、少し大きめに取っても問題はないだろうと考え、今後は「領海(よりやや広い領域)」と書くこととします。

``jp`` テーブルを使用します。``pref_simplified_3395``テーブルでは、非常に小さい島が消えています。


## 単純なバッファを作る

領海(よりやや広い領域)の生成には、外側にバッファを作りました。

距離は、12海里=22,224mよりやや広い25,000mとしました。

```
CREATE TABLE buffered_1 (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(POLYGON, 6668)
);
CREATE INDEX ON buffered_1 USING GiST (geom);
-- ST_Bufferでマルチポリゴンが現れたので Dump した
INSERT INTO buffered_1 (geom)
SELECT (ST_Dump(ST_Buffer(geom::GEOGRAPHY, 25000)::GEOMETRY)).geom AS geom FROM jp;
```

## 結合、ダンプする

```
CREATE TABLE buffered_2 (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(POLYGON, 6668)
);
CREATE INDEX ON buffered_2 USING GiST (geom);

INSERT INTO buffered_2 (geom)
SELECT (ST_Dump(ST_CollectionExtract(ST_Multi(ST_Union(geom)), 3))).geom FROM buffered_1;
```

### QGISで見る

![QGISで buffered_2 の地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/02-buffered_2.png)

これ(``buffered_2``)、鹿児島の薩摩半島と甑島諸島との間ぐらいに穴が開いてしまっています。

## 穴をなくして1個のマルチポリゴンにしてEPSG:3395にする

``buffered_2`` を見るとの穴がは、外した方が都合がよさそうなので、穴をなくします。

[PostGIS入門](../../caea8d4c77dbba2e23a0) / [結合でできた「隙間」を排除する](../../caea8d4c77dbba2e23a0/viewer/delcrack) でご紹介した通り、ポリゴンごとに外側リングだけを抽出できます。

また、今回はマスキングとして使うので、ひとつのマルチポリゴンにしてしまいました。

1行だけになるので、インデックスは作りませんでした。

```
CREATE TABLE buffered_3395 (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(MULTIPOLYGON, 3395)
);

INSERT INTO buffered_3395 (geom)
SELECT ST_Collect(ST_Transform(ST_MakePolygon(ST_ExteriorRing(geom)), 3395)) FROM buffered_2;
```

### QGISで見る

![QGISで buffered_3395 の地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/03-buffered_3395.png)

いい眺めですね。


# すげー時間がかかるけど隙間とか切れ込みのない簡略化された都道府県ポリゴンを作成

隙間とか切れ込みのない簡略化された都道府県ポリゴンを作成するために、トポロジを使用しました。

[PostGIS トポロジ 入門](../../pgis-topology-beginner) をご覧下さい。

## トポロジとトポジオメトリテーブルを作る

```
voro=# SELECT topology.CreateTopology('topo_pref_3395', 3395);

 createtopology
----------------
              1
(1 row)
```

```
voro=# CREATE TABLE topogeom_pref_3395 (
  gid serial primary key,
  n03_007 TEXT
);

voro=# SELECT topology.AddTopoGeometryColumn(
  'topo_pref_3395',
  'public', 'topogeom_pref_3395', 'topo',
  'POLYGON'
);

 addtopogeometrycolumn
-----------------------
                     1
(1 row)
```

トポジオメトリテーブルは次のように確認できます。

```
voro=# SELECT * FROM topology.layer;

 topology_id | layer_id | schema_name |     table_name     | feature_column | feature_type | level | child_id
-------------+----------+-------------+--------------------+----------------+--------------+-------+----------
           1 |        1 | public      | topogeom_pref_3395 | topo           |            3 |     0 |
(1 row)
```

## データを叩き込む

これは、**約1日かかる大仕事でした**。

```
INSERT INTO topogeom_pref_3395(n03_007, topo)
  SELECT n03_007, topology.toTopoGeom(ST_Transform(geom, 3395), 'topo_pref_3395', 1)
  FROM raw_pref;
```

## 単純化したポリゴンを生成する

トポロジを1日かけて作ったら、単純化したポリゴンを生成します。

```
-- Simplifyする 数分程度
-- INSERT 0 10139
--
CREATE TABLE pref_simplified_3395 (
  gid SERIAL PRIMARY KEY,
  n03_007 TEXT,
  geom GEOMETRY(POLYGON, 3395)
);
CREATE INDEX ON pref_simplified_3395 (n03_007);
CREATE INDEX ON pref_simplified_3395 USING GiST (geom);

INSERT INTO pref_simplified_3395(n03_007, geom)
SELECT n03_007, (ST_Dump(ST_Simplify(topo,30))).geom
FROM topogeom_pref_3395 ORDER BY gid;
```

```
-- 単純化後のポイント個数
voro=# SELECT COUNT(*) FROM (SELECT ST_DumpPoints(geom) FROM pref_simplified_3395) AS Q1;
 count
--------
 366423
(1 row)

-- 単純化前のポイント個数
voro=# SELECT COUNT(*) FROM (SELECT ST_DumpPoints(geom) FROM raw_pref) AS Q1;
  count
----------
 10697558
(1 row)
```

``ST_Simplify(topo, 30)`` の第2引数 (``tolerance``) の値を ``30`` としています。

``tolerance`` の値が大きい方が簡略化の度合いが大きくなり、ポイントは減ります。その後の計算量は減りますし、体感ですが GEOSエラー 遭遇率が減ります。

しかし、``tolerance``が大きすぎると問題もあります。小さな島などは消えます。たとえば、愛媛県上島町生名の亀島が残らないと、亀島あたりが因島の領内に入ることになります。また、東京都沖ノ鳥島が消えたりする場合もあります。

``tolerance`` の値をアレコレしていたところ、次のような結果になりました。

  * 50 では 亀島も沖ノ鳥島もダメ
  * 40 だと 沖ノ鳥島がダメ
  * 30 では どちらも残りました。
  * 10 だと今度は ``ST_VoronoiPolygons()`` で GEOSエラー が出ました。

## QGISで単純化した図を見てみる

![QGISでpref_simplified_3395の地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/04_pref_simplified.png)

しれっと出していますが、コレ、県境界を簡略化しているのですが、両県のポリゴンで重複部分ができて、一方の県がもう一方の県にささってたり、逆に両県ポリゴンの間にスキマができたり、といった問題が発生していないのです。

トポロジはデータ格納に時間がかかりますが、トポロジでしかできないことがあるのです。

# ポイントをダンプしてボロノイ分割を行う (非細分化版)

## ポイントをダンプする (非細分化版)

この際、使用するポイントと使用しないポイントを分けるので、``used``フィールドを作りました。

```
-- ポイントをダンプ (INSERT 0 366423)
CREATE TABLE nonseg_points_3395 (
  gid SERIAL PRIMARY KEY,
  n03_007 TEXT,
  used INT,
  geom GEOMETRY(POINT, 3395)
);
CREATE INDEX ON nonseg_points_3395 USING GiST (geom);
CREATE INDEX ON nonseg_points_3395 (n03_007);
CREATE INDEX ON nonseg_points_3395 (used);

INSERT INTO nonseg_points_3395(n03_007, used, geom) SELECT n03_007, 1, (ST_DumpPoints(geom)).geom FROM pref_simplified_3395;
```

他県のポイントと重なる箇所は、陸域で接触している箇所ですが、今回は海域のみを考えるので、使わないようにします。

```
UPDATE nonseg_points_3395 SET used=0 WHERE used=1 AND EXISTS (
  SELECT * FROM nonseg_points_3395 AS p1
  WHERE nonseg_points_3395.n03_007 <> p1.n03_007 AND ST_DWithIn(nonseg_points_3395.geom,p1.geom, 0.001)
);
```

## ボロノイ分割を行う (非細分化版)

```
--
-- INSERT 0 91367 (100m)
--
CREATE TABLE nonseg_vororaw_3395 (
  gid SERIAL PRIMARY KEY,
  n03_007 TEXT,
  geom GEOMETRY(POLYGON, 3395)
);
CREATE INDEX ON nonseg_vororaw_3395 (n03_007);
CREATE INDEX ON nonseg_vororaw_3395 USING GiST (geom);

-- INSERT 0 280265 (100mは不明)
-- insert nonseg_vororaw_3395.n03_007 は null のまま行追加)
INSERT INTO nonseg_vororaw_3395 (geom)
SELECT (ST_Dump(ST_CollectionExtract(ST_VoronoiPolygons(ST_Collect(geom),3)))).geom AS geom
FROM nonseg_points_3395 WHERE used=1;
```

```
-- nonseg_vororaw_3395.n03_007 を設定
-- UPDATE 280265 (100m不明)
UPDATE nonseg_vororaw_3395 SET n03_007=(
  SELECT n03_007 FROM nonseg_points_3395
  WHERE ST_WithIn(nonseg_points_3395.geom, nonseg_vororaw_3395.geom) AND used=1 LIMIT 1
);
```

### QGISで見る

ボロノイ分割の結果は次の通りです。

![QGISで nonseg_vororaw_3395 の地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/05-nonseg_vororaw_3395.png)

分かりにくいですが、なんとなく日本列島っぽいものが見えてると思います。

## ボロノイ分割の結果ポリゴンを都道府県ごとにまとめる (非細分化版)

```
CREATE TABLE nonseg_sealandarea_pref_raw_3395 (
  gid SERIAL PRIMARY KEY,
  n03_007 TEXT,
  geom GEOMETRY(POLYGON, 3395)
);
CREATE INDEX ON nonseg_sealandarea_pref_raw_3395 (n03_007);
CREATE INDEX ON nonseg_sealandarea_pref_raw_3395 USING GiST (geom);

WITH mlt AS (
  SELECT n03_007, ST_CollectionExtract(ST_Multi(ST_Union(geom)),3) AS geom
  FROM nonseg_vororaw_3395 GROUP BY n03_007
)
INSERT INTO nonseg_sealandarea_pref_raw_3395(n03_007, geom)
SELECT n03_007, (ST_Dump(mlt.geom)).geom FROM mlt;
```

### QGISで見る

![QGISで nonseg_sealandarea_pref_raw_3395 の地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/06-nonseg_sealandarea_pref_raw_3395.png)

正直、これでもよく分からないですね。

# おやなにかがおかしいです

塗りつぶしをなくして、境界線だけ表示するようにして、アップにしてみましょう。

![QGISで 地理院地図 と nonseg_seaarea_pref_3395 を重ねて、因島土生町付近を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/07-nonseg_innoshima_gsimap.png)

瀬戸内の島々は所属が意外と複雑で、亀島（上図の上端付近）がどうみても因島（広島県）に属するものと見えますが、ここは愛媛県に属しています。それをしっかり反映させられています。土生港付近（上図の中心付近）が何か怪しいですけど。

# 「土生港付近が何か怪しい」理由

土生港付近で境界がフラフラしてるのは、母点の数が適切でないためだろうと考えられます。

![QGISで 地理院地図 と nonseg_seaarea_pref_3395 とボロノイ分割の図と母点とを重ねて、因島土生町付近を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/08-nonseg_voronoi_and_points.png)

右側の陸地（広島県・因島）のうち、土生港付近の208006点と208007点とが、かなり離れています。そして、左側の陸地（愛媛県・生名島）の246268点がちょうど、土生港付近の2点の中間に入り込んでいける位置にあります。これで、208006点と208007点と246268点が等間隔になるポイントがほぼ因島の海岸線上になってしまうことになりました。

これを解決するには、208006点と208007点との間に、``ST_Segmentize()``で、いくつか点を挿入しておけばいいんじゃないかなと思いました。

# リングを細分化したうえでボロノイ分割をしてみる

細分化版から見た非細分化版との違いは、ラングを 50m (EPSG:3395) で細分化させたうえでポイントをダンプさせていることだけです。

リング/ラインストリングの細分化については https://postgis.net/docs/ja/ST_Segmentize.html を参照して下さい。リング/ラインストリングの隣同士の点間の距離が指定した距離を超えないよう点を挿入する処理です。

テーブルも、テーブル名はさすがに変えていますが、フィールド構成は``nonseg_points_3395`` 以外は変わりません。

```
CREATE TABLE seg_points_3395 (
  gid SERIAL PRIMARY KEY,
  n03_007 TEXT,
  used INT,
  geom GEOMETRY(POINT, 3395)
);
CREATE INDEX ON seg_points_3395 USING GiST (geom);
CREATE INDEX ON seg_points_3395 (n03_007);
CREATE INDEX ON seg_points_3395 (used);
```

```
-- 50mで切る INSERT 0 1593375
INSERT INTO seg_points_3395(n03_007, used, geom)
SELECT n03_007, 1, (ST_DumpPoints(ST_Segmentize(geom, 50))).geom FROM pref_simplified_3395;
```

```
-- 重なってるのは使わない UPDATE 748455
UPDATE seg_points_3395 SET used=0 WHERE used=1 AND EXISTS (
  SELECT * FROM seg_points_3395 AS p1 WHERE seg_points_3395.n03_007 <> p1.n03_007 AND ST_DWithIn(seg_points_3395.geom,p1.geom, 0.001)
);
```

## ボロノイ分割を行う

```
CREATE TABLE seg_vororaw_3395 (
  gid SERIAL PRIMARY KEY,
  n03_007 TEXT,
  geom GEOMETRY(POLYGON, 3395)
);
CREATE INDEX ON seg_vororaw_3395 (n03_007);
CREATE INDEX ON seg_vororaw_3395 USING GiST (geom);

INSERT INTO seg_vororaw_3395 (geom)
SELECT (ST_Dump(ST_CollectionExtract(ST_VoronoiPolygons(ST_Collect(geom),3)))).geom AS geom
FROM seg_points_3395 WHERE used=1;

UPDATE seg_vororaw_3395 SET n03_007=(
  SELECT n03_007 FROM seg_points_3395 WHERE ST_WithIn(seg_points_3395.geom, seg_vororaw_3395.geom) AND used=1 LIMIT 1
);
```

## 都道府県ごとにまとめる

```
CREATE TABLE seg_sealandarea_pref_raw_3395 (
  gid SERIAL PRIMARY KEY,
  n03_007 TEXT,
  geom GEOMETRY(POLYGON, 3395)
);
CREATE INDEX ON seg_sealandarea_pref_raw_3395 (n03_007);
CREATE INDEX ON seg_sealandarea_pref_raw_3395 USING GiST (geom);

WITH mlt AS (
  SELECT n03_007, ST_CollectionExtract(ST_Multi(ST_Union(geom)),3) AS geom
  FROM seg_vororaw_3395 GROUP BY n03_007
)
INSERT INTO seg_sealandarea_pref_raw_3395(n03_007, geom)
SELECT n03_007, (ST_Dump(mlt.geom)).geom FROM mlt;
```

## 改めて因島南西部を見てみる

改めて、リングを細分化した後の因島南西部を見てみましょう。

![QGISで 地理院地図 と nonseg_seaarea_pref_3395 を重ねて、因島土生町付近を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/09-innoshima_gsimap.png)

まだフラフラしているところもありますが、土生港の海岸線付近まで切り込んでくることはなくなりました。

## 改めて母点も見てみる

母点も見直してみましょう。

![QGISで 地理院地図 と nonseg_seaarea_pref_3395 とボロノイ分割の図と母点とを重ねて、因島土生町付近を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/10-voronoi_and_points.png)

ご覧の通り、ビッシリとポイントが出ていますね。

## ボロノイまとめポリゴンから領海外を除去する

地図表示から CLI に戻ります。先ほど作成したボロノイ分割ポリゴン``seg_sealandarea_pref_raw_3395``をフルに使うのではなく、``buffered_3395``の領海(よりやや広い領域)だけに適用してみます。

```
CREATE TABLE sealandarea_pref_3395 (
  gid SERIAL PRIMARY KEY,
  pcode TEXT, -- n03_007
  geom GEOMETRY(POLYGON, 3395)
);
CREATE INDEX ON sealandarea_pref_3395 (pcode);
CREATE INDEX ON sealandarea_pref_3395 USING GiST (geom);

INSERT INTO sealandarea_pref_3395 (pcode, geom)
SELECT n03_007,
  (ST_Dump(
    ST_CollectionExtract(
      ST_Multi(
        ST_Intersection(geom, (SELECT geom FROM buffered_3395))
      ),3
    )
  )).geom AS geom
FROM seg_sealandarea_pref_raw_3395
ORDER BY n03_007;
```

## QGISで見る

![QGISで sealandarea_pref_3395 の地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/11-sealandarea_pref.png)

なんとなく都道府県界が出てるようにみえますね。ただし、陸域の境界としては誤りを含んでいます。

というのも、ボロノイ分割に使ったのは海岸線の母点だけで、陸域の都道府県境界は捨てました。このため、栃木県、群馬県、埼玉県、山梨県、長野県、岐阜県、滋賀県、奈良県が存在しません。

余談ですが、栃木県は茨城に編入、群馬県は東京と新潟で分割、埼玉県は東京に編入、山梨県は神奈川と静岡で分割、長野県は新潟、富山、静岡、愛知の4県で分割、岐阜県は富山、石川、福井、愛知の4県で分割、滋賀県は福井、三重、大阪で分割、奈良県は三重、大阪、和歌山の3県で分割、となっています (**余談にもほどがあるやないの**)。

あくまで陸域部分については、不適切なのは知ってるけど放置しているだけです。陸域部分は使わないようにして下さい。

# 陸域も除去する

上記ではボロノイまとめポリゴンから領海外を除去しましたが、陸域は使えないので除去したいと思います。この際、小さい島は除去対象から除外することとします。

## 日本の大き目の島だけを抜き出す

QGISで``jp``テーブルをつっついて、面積を確認していきました。

だいたい、次のようになっていました(単位は平方メートル、小数点切り上げ)

* 沖縄本島 - 1,208,429,382
* 福江島 (五島列島) - 326,357,654
* 島後 (隠岐諸島) - 241,533,716

これらから、240,000,000平方メートルとしました。

```
CREATE TABLE jpmask_1 (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(POLYGON, 6668)
);
CREATE INDEX ON jpmask_1 USING GiST (geom);

INSERT INTO jpmask_1 (geom)
SELECT geom FROM jp WHERE ST_Area(geom::GEOGRAPHY) >= 240000000;
```


## 200m後退した日本ポリゴンを作る

``sealandarea_pref_3395`` は内陸県が存在していないため、これを直接使うのではなく、陸域は ``raw_pref`` にまかせて``sealandarea_pref_3395`` は``raw_pref``から外れた領域で使うことにします。そのため、``sealandarea_pref_3395`` をくり抜きます。

ただし ``raw_pref`` で厳密にくり抜く必要は無いので、ややノードが減るのを期待して、``raw_pref``の海岸線を内陸に200m (エイヤで決めました)ほど後退させたポリゴンでくり抜きます。

まず、200m後退した日本ポリゴンを作成します。``ST_Buffer()``はジオグラフィ版を使っていて、``-200``と指定することで、後退させています。
この手続きは7分程度でした。他の大多数の手続きが1分以内だったのに比べたら時間がかかりますが、トポロジよりはずっとマシです。


```
CREATE TABLE jpmask_2 (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(POLYGON, 6668)
);
CREATE INDEX ON jpmask_2 USING GiST (geom);

INSERT INTO jpmask_2 (geom)
SELECT (ST_Dump(ST_Buffer(geom::GEOGRAPHY, -200)::GEOMETRY)).geom AS geom FROM jpmask_1;
```

## 島と化した半島を除いて、ひとつのマルチポリゴンにする

次にマスクとして使用するためのポリゴンにします。

ここで、``jpmask_1``生成時に小さい島を除外していますが、さらに小さいポリゴンを除外しています。これは、200m後退させたがために、一部の半島が島になってしまい、これを除外するためです。

除外されなかったポリゴンは``EPSG:3395``に座標変換したうえで、``ST_Collect()``で一つのマルチポリゴンにまとめます。なお、``jpmask_2``では、既にポリゴン同士がインタセクトしていることはないので、``ST_Union()``を使う必要はありません。

```
CREATE TABLE jpmask_3395 (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(MULTIPOLYGON, 3395)
);

INSERT INTO jpmask_3395 (geom)
SELECT ST_Collect(ST_Transform(geom,3395)) FROM jpmask_2
WHERE ST_Area(geom::GEOGRAPHY) >= 200000000;
```

## QGISで見る

![QGISで jpmask_3395 の地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/12-jpmask_3395.png)

ここでは、島が減ったことだけ見えればいいです。本当は ``raw_pref`` より少しだけ痩せています。

## 200m後退した日本ポリゴンでくり抜く

``sealandarea_pref_3395``から、ついさっき作った``jpmask_3395``でくり抜きます

```
CREATE TABLE seaarea_pref_3395 (
  gid SERIAL PRIMARY KEY,
  pcode TEXT, -- n03_007
  geom GEOMETRY(POLYGON, 3395)
);
CREATE INDEX ON seaarea_pref_3395 (pcode);
CREATE INDEX ON seaarea_pref_3395 USING GiST (geom);

INSERT INTO seaarea_pref_3395 (pcode, geom)
SELECT pcode,
  (ST_Dump(
    ST_CollectionExtract(
      ST_Multi(
        ST_Difference(geom, (SELECT geom FROM jpmask_3395))
      ),3
    )
  )).geom AS geom
FROM sealandarea_pref_3395
ORDER BY pcode;
```

### QGISで見る

![QGISで seaarea_pref の地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/13-seaarea_pref.png)

陸域（200m後退した日本ポリゴン）を除去しました。日本列島の海岸線の縁取りを施した絵にしかなっていないように見えますが、このピンクの領域内に、ちゃんと度道府県境界線が引かれています。

![QGISで raw_pref と seaarea_pref を重ねて表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/14-land_and_sea.png)

ピンクが、領海よりやや広い海域に限ってでっち上げた「海域それっぽい都道府県界ポリゴン」で、テーブル名は seaarea_pref です。緑が 国土数値情報をインポートしただけの raw_pref です。

# 最後に地理座標系にして都道府県名を付加したテーブルを作る

地理座標系にして都道府県名を付加したテーブルを作ってみました。これをもとに各種フォーマットにエクスポートしようかと思います。

これが本当必要かどうかはよく分かりませんが、書くだけ書いておきます。

まず、地理座標系にします。

```
CREATE TABLE seaarea_pref (
  gid SERIAL PRIMARY KEY,
  pcode TEXT, -- n03_007
  pname TEXT, -- n03_001
  geom GEOMETRY(POLYGON, 6668)
);
CREATE INDEX ON seaarea_pref (pcode);
CREATE INDEX ON seaarea_pref USING GiST (geom);

INSERT INTO seaarea_pref (pcode, geom)
SELECT  pcode, ST_Transform(geom, 6668) AS geom
FROM seaarea_pref_3395
ORDER BY gid;
```

その後で、都道府県名を補充します。

```
UPDATE seaarea_pref
SET pname=(SELECT t_pref.pname FROM t_pref WHERE t_pref.pcode=seaarea_pref.pcode);
```

念のため``pname``が``NULL``になってないか確認しておきます。

```
voro=# SELECT * FROM seaarea_pref WHERE pname IS NULL;
 gid | pcode | pname | geom
-----+-------+-------+------
(0 行)
```

## 確認する

不正なポリゴンが発生していないかを確認します。

```
SELECT gid FROM seaarea_pref WHERE NOT ST_IsValid(geom);
 gid
-----
(0 行)
```

なかったですね。これで OK とします。

# ギャラリー

さきほどと違い、「海域それっぽい都道府県界ポリゴン」は、塗りつぶしせずに、境界線をオレンジ色で描画しています。

![北陸地域や山陰地域の県境を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/15-hokuriku-sannin.png)

ざっと見て、いい具合に仕上がってるように見えます。

![福島県と茨城県の県境を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/16-fukushima-ibaraki.png)

やや入り組んでいますが、いい感じに境界が描けたと思います。

![イシマの県境を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/18-ishima.png)

有人の離島で唯一の県境（岡山県と香川県）を持つ島であるイシマ（石島/井島）の周辺もしっかりと分割できています。

![羽田空港D滑走路付近の県境を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/seaarea-pref/17-haneda-rwd.png)

羽田空港沖合滑走路に県境がかかっていますが、羽田空港D滑走路の人工島について、国土数値情報と地理院地図とで違うので、仕方ないかと思います。

# おわりに

結構うまくいったように見えたら幸いです。**どの局面で使うか分からない**けどな。

ここまでの説明では完全に無かったことにしてますが、GEOSError が時々出てきました。このエラーが出ると、どうすればいいか分からなくなります。単純化しすぎてポリゴンが不正になったのかなとか、単純化が足りずに逆に他のポリゴンとのインタセクション等で不正ポリゴンが生成されたんじゃないのかとか、いろいろぐるぐる考えは巡るのですが、マジで分かりませんでした。

結局、簡略化等のパラメータについては、理論的な背景なんか一片も考えずに、適当にいじってなんとかする方法で解決しました。

**それでもうまくいくもんだな**と思いました（パッとしないまとめ）。

今回、意外だったのが、これまで全く使ってなかった``ST_Segmentize()``が、いい仕事をしてくれたことです。今回のもひとつの使い方なのかも知れません。

それと、最初にも書きましたが、このデータは参考程度に使うべきものです。たとえば**自治紛争委員に**「海域それっぽい都道府県界ポリゴン」を作成して**提出しないで**ください（おらんわ）。

# 参照

* PostGIS料理帳 / [国土数値情報のダウンロード](dl-seaarea-border)
* [PostGISをはじめてみよう](../../b1de0a18073af70946e0) / [シェープファイルのデータをインポートしてみよう コマンドライン編](../../b1de0a18073af70946e0/viewer/import-cli)
* [PostGISをはじめてみよう](../../b1de0a18073af70946e0) / [空間参照系の概要](../../b1de0a18073af70946e0/viewer/qgis)
* PostGIS料理帳 / [ボロノイ分割でおおざっぱな消防署の管轄範囲を作ろう](fire-voronoi)
* [PostGIS入門](../../caea8d4c77dbba2e23a0) / [結合でできた「隙間」を排除する](../../caea8d4c77dbba2e23a0/viewer/delcrack)
* https://postgis.net/docs/ja/ST_Segmentize.html

# 出典

本章の作成にあたり、次のデータを使用しました。

* 「国土数値情報 (行政区域)」 (国土交通省) https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-A09.html (2024年11月2日取得)
* 「地理院タイル」 (国土地理院) https://maps.gsi.go.jp/development/ichiran.html (2024年11月11日取得)


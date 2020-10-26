---
title: "コンビニを近い順に5つだけ抽出する"
emoji: "😀"
type: "tech"
topics: [PostGIS, OSM]
published: false
---
# はじめに

[OverPass APIからデータを落としてPostGISに入れてみた](https://boiledorange73.qrunch.io/entries/809aiJzINuKMCopY)では、得たデータから「何かする」とだけ宣言しました。

ありがちなものですが、指定した位置から近いPOI (Point Of Interest)を探してみたいと思います。

この際、KNN GiSTを使って、高速な検索を行い、``ST_Distance``を使う場合と比べてみます。

## 前回のデータは使いません

コンビニエンスストアを対象にしていますが、実は前回の実行例では、抽出範囲内にコンビニはあまり無く、ありがたみを感じないものとなったので、もう一度取得しなおします。

# 準備

## データを引っ張りなおしてくる

[OverPass APIからデータを落としてPostGISに入れてみた](https://boiledorange73.qrunch.io/entries/809aiJzINuKMCopY)でも紹介しましたが、OverPass APIを使います。

http://overpass-api.de/query_form.html にアクセスして、"Overpass API Query Form"の見出しのあるところの直下のテキストエリアに次のXMLをコピペして、"Query"をクリックするとデータを生成してくれます。5秒程度かかるので、あまりサーバに負荷をかけないようにし、頻繁にアクセスするのは控えましょう。

```xml
<query type="node">
  <!-- ボックスを指定 -->
  <bbox-query s="35.572" n="35.786" w="139.663" e="139.883" />
  <!-- コンビニエンスストアのみ抜き出す -->
  <has-kv k="shop" v="convenience" />
</query>
<print/>
```

## 前のデータを消して新たなデータを格納する

先ほど取得したデータを、".osm"というサフィックスを付けたファイル名にします。

それから``osm2pgsql``でデータベースに叩き込みます。今回は前に作ったものを消すために``-c``オプションを指定しています。

```csh
% osm2pgsql -d db -c -l convenience_stores.osm
```

## ジオグラフィテーブルに叩き込む

次のとおり、テーブルを作成して、全データを複製しておきます。

```psql
CREATE TABLE cs_geog (
  gid SERIAL PRIMARY KEY,
  osm_id BIGINT,
  name TEXT,
  geog GEOGRAPHY(POINT, 4326)
);

CREATE INDEX ix_cs_geog_geog ON cs_geog USING GiST (geog);

INSERT INTO cs_geog(osm_id, name, geog)
SELECT osm_id, name, ST_Transform(way, 4326)::GEOGRAPHY
FROM planet_osm_point;
```

なお、``ST_Transform``をかませているのは、EPSG:3857でplanet_osm_point

# ST_Distanceを使用する方法

通常、距離順にソートするには、距離を計測する式を``ORDER BY``句に与えると思います。次のようになるでしょう。

```psql
db=# SELECT
  osm_id,
  name,
  ST_Distance('SRID=4326;POINT(139.77 35.68)'::GEOGRAPHY, geog)
FROM cs_geog
ORDER BY
  ST_Distance('SRID=4326;POINT(139.77 35.68)'::GEOGRAPHY, geog)
LIMIT 5;
   osm_id   |       name       | st_distance  
------------+------------------+--------------
 1423728720 | セブン-イレブン  |  95.74956557
 2365555224 | サンクス         | 209.25617877
 2365111053 | ファミリーマート | 231.60629173
 3756803582 | セブン-イレブン  | 249.71026452
 2365110377 | セブン-イレブン  | 251.34838716
(5 行)
```

今回は``WHERE``句で``ST_DWithin``を使うことで、候補を絞り込んだうえでソートを実行します。この場合だと ``WHERE ST_DWithin(geog, 'SRID=4326;POINT(139.77 35.68)'::GEOGRAPHY, 1000)``といったふうにします。

ただし、地物データの分布や指定する位置によっては距離上限を1000mで切ったら候補が0になる場合があったり、調整が難しくなります。

# KNN GiST演算子を使う

## とりあえずやってみる

``<->``演算子を使うと距離順のソートができます。

```psql
db=# SELECT
  osm_id,
  name,
  ST_Distance('SRID=4326;POINT(139.77 35.68)'::GEOGRAPHY, geog)
FROM cs_geog
ORDER BY
  'SRID=4326;POINT(139.77 35.68)'::GEOGRAPHY <-> geog
LIMIT 5;
   osm_id   |       name       | st_distance  
------------+------------------+--------------
 1423728720 | セブン-イレブン  |  95.74956557
 2365555224 | サンクス         | 209.25617877
 2365111053 | ファミリーマート | 231.60629173
 3756803582 | セブン-イレブン  | 249.71026452
 2365110377 | セブン-イレブン  | 251.34838716
(5 行)
```

``ORDER BY``句には``ST_Distance``が無いにもかかわらず、``ST_Distance``を使った場合と同じ結果になりました。

## 実行速度を比較してみる

先ほど示した通り、``ST_Distance``と``<->``とを結果で比較したら、同じ結果が出ました。

まったく面白くないですね。「だからなんなんだ」と。

では、結果はひとまずおいておいて、実行速度を比較してみましょう。


まずは``ST_Distance``を実行してみます。

```psql
db=# EXPLAIN ANALYZE SELECT
  osm_id,
  name,
  ST_Distance('SRID=4326;POINT(139.77 35.68)'::GEOGRAPHY, geog)
FROM cs_geog
ORDER BY
  ST_Distance('SRID=4326;POINT(139.77 35.68)'::GEOGRAPHY, geog)
LIMIT 5;
                                                              QUERY PLAN                                                              
--------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=798.25..798.26 rows=5 width=85) (actual time=12.208..12.209 rows=5 loops=1)
   ->  Sort  (cost=798.25..805.09 rows=2734 width=85) (actual time=12.206..12.207 rows=5 loops=1)
         Sort Key: (_st_distance('0101000020E6100000713D0AD7A3786140D7A3703D0AD74140'::geography, geog, '0'::double precision, true))
         Sort Method: top-N heapsort  Memory: 25kB
         ->  Seq Scan on cs_geog  (cost=0.00..752.84 rows=2734 width=85) (actual time=0.199..10.911 rows=2734 loops=1)
 Planning time: 0.131 ms
 Execution time: 12.237 ms
(7 行)
```

続いて``<->``です。


```psql
db=# EXPLAIN ANALYZE SELECT
  osm_id,
  name,
  ST_Distance('SRID=4326;POINT(139.77 35.68)'::GEOGRAPHY, geog)
FROM cs_geog
ORDER BY
  'SRID=4326;POINT(139.77 35.68)'::GEOGRAPHY <-> geog
LIMIT 5;
                                                              QUERY PLAN                                                              
--------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.14..3.22 rows=5 width=85) (actual time=0.617..0.653 rows=5 loops=1)
   ->  Index Scan using ix_cs_geog_geog on cs_geog  (cost=0.14..1678.99 rows=2734 width=85) (actual time=0.616..0.650 rows=5 loops=1)
         Order By: (geog <-> '0101000020E6100000713D0AD7A3786140D7A3703D0AD74140'::geography)
 Planning time: 0.137 ms
 Execution time: 0.695 ms
(5 行)
```

見比べてみて下さい。"Execution time"が、``ST_Distance``だと 12.237ミリ秒で、``<->``だと 0.695ミリ秒です。

ダブルスコアなんてものではありません。しかも、``ST_Distance``だとシーケンシャルスキャンを行うので、データが増えれば増えるほど不利になります。

## KNN GiST

GiSTは、PostgreSQLが用意する一般化されたインデックス機能で、いくつかの演算子と関数を実装すれば、好きなインデックスを作ることができます。PostGISはジオメトリ/ジオグラフィのインデックスを構築するための演算子実装を持っています。
地物カラムに対してインデックスを付ける時に``USING GiST``という句を入れていると思います。これがGiSTです。

KNNは、PostgreSQL 9.1で追加されました。評価演算子``<->``に、何らかの評価関数を与えると、その評価関数から得られるスカラ値でソートする際にインデックスを効かせることができるようになる機能です (理解が足りないので間違った表現かも知れません)。

## KNN GiSTの注意点

* PostgreSQL 9.1以上、PostGIS 2.0以上でなければなりません (新たに導入された方は問題ないと思います)。
* ジオグラフィの場合、球面上の距離を計算します。回転楕円体面上ではありません。このため、「概ね」の評価しかできません (ジオメトリの場合はユークリッド距離を計算します)。

# おわりに

今回はOverPassを使ってコンビニエンスストアの位置データを得て (前回の復習)、指定した地点から近い順に並べる方法を紹介しました。

KNN GiSTによって高速検索が可能になることを示し、ジオグラフィで使う場合の注意点も少し触れました。

距離の評価の精度をあまり気にしない場合には、KNN GiSTの方が高速であり、かつ、``WHERE``句で絞り込む必要もないので、扱いやすいので、使ってみてください。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

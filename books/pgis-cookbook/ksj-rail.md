---
title: "全国の鉄道データ"
---
# はじめに

# 必要なデータ

全国の鉄道データは、国土交通省国土政策局が提供する「国土数値情報 (鉄道)」を使います。

  * [国土数値情報のダウンロード](dl-ksj-ua)

# PGDUMPデータ作成とインポート

[PostGISを入れるところからやってみよう](../../b1de0a18073af70946e0)にある[各種データを ogr2ogr でインポートしてみよう](../../b1de0a18073af70946e0/viewer/import-ogr2ogr)を参照して下さい。

国土数値情報 (鉄道) には、路線(ラインストリング)と駅(ポイント)があります。

## 路線

```batch
ogr2ogr -f PGDUMP ^
  --config PG_USE_COPY YES ^
  -lco SPATIAL_INDEX=GIST ^
  -lco GEOMETRY_NAME=geom ^
  -nln ksj_rail_road ^
  ksj_rail_road.sql N02-22_RailroadSection.shp
```

```batch
> set PGCLIENTENCODING=UTF8
> psql -d db -f ksj_rail_road.sql
```

## 駅

```batch
ogr2ogr -f PGDUMP ^
  --config PG_USE_COPY YES ^
  -lco SPATIAL_INDEX=GIST ^
  -lco GEOMETRY_NAME=geom ^
  -nln ksj_rail_station ^
  ksj_rail_station.sql N02-22_Station.shp
```

```batch
> set PGCLIENTENCODING=UTF8
> psql -d db -f ksj_rail_station.sql
```

## geometry_columns で確認

ジオメトリタイプと座標参照系(空間参照系とも)を確認しておきましょう。``geometry_columns``というビューで確認できます。

```
db=# select * from geometry_columns;
 f_table_catalog | f_table_schema |       f_table_name       | f_geometry_column | coord_dimension | srid |    type
-----------------+----------------+--------------------------+-------------------+-----------------+------+------------
...
 db              | public         | ksj_rail_road            | geom              |               2 | 6668 | LINESTRING
 db              | public         | ksj_rail_station         | geom              |               2 | 6668 | LINESTRING
...
```

EPSG:6668 のラインストリングであることが確認できました。

駅もラインストリング? その通りです。

# QGISで見てみる

まず、``ksj_rail_road``テーブル (路線)を見てみましょう。

![QGISでksj_rail_roadの地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/ksj-rail/01-ksj-rail.png)

次に ``ksj_rail_station``テーブル (駅)を見てみましょう。

![QGISでksj_rail_stationの地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/ksj-rail/02-ksj-station-wide.png)

あまりいい眺めでもないですね。小縮尺過ぎて、駅がひっつきあってしまっています。

ズームして見ましょう。

![QGISでksj_rail_roadとksj_rail_stationの東京駅近辺の地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/ksj-rail/03-ksj-station.png)

ああいい眺めですね。

# いろいろ聞いてみる

## 新幹線の総延長は?

https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N02-v3_1.html を見ると、路線の ``n02_002`` (事業車種別) が ``1`` だと新幹線だそうです。

```
db=# SELECT Sum(ST_Length(geom::GEOGRAPHY)) FROM ksj_rail_road WHERE n02_002=1;
ERROR:  演算子が存在しません: character varying = integer
LINE 1: ...Length(geom::GEOGRAPHY)) FROM ksj_rail_road WHERE n02_002=1;
                                                                    ^
HINT:  指定した名称と引数の型に合う演算子がありません。明示的な型キャストが必要かもしれません。
```

何が起こったと思ったのですが、よく見ると、文字列型と整数型とで比較しようとしてのエラーでした。

```
SELECT Sum(ST_Length(geom::GEOGRAPHY)) FROM ksj_rail_road WHERE n02_002='1';
```

```
        sum
--------------------
 2838077.8721216745
(1 row)
```

https://ja.wikipedia.org/wiki/%E6%96%B0%E5%B9%B9%E7%B7%9A を見ると、フル規格は 2,830km だそうです。思ったよりも近い値が得られました。

## 無軌条鉄道を見る

無軌条鉄道っていうのが若干気になりませんか? 気にならないですか。でも気になったことにして下さい。

総延長を見てみましょう。

```
db=# SELECT Sum(ST_Length(geom::GEOGRAPHY)) FROM ksj_rail_road WHERE n02_001='17';
        sum
--------------------
 3749.6225432374376
(1 row)
```

3kmですって。おそらく、1路線しかなさそうです。

## 軌道の総延長

```
db=# SELECT Sum(ST_Length(geom::GEOGRAPHY)) FROM ksj_rail_road WHERE n02_001='21';
        sum
-------------------
 350428.9463133114
(1 row)
```

約 350km だそうです。

ここで QGIS で確認してみましょう。

```
SELECT * INTO ksj_rail_road_tram FROM ksj_rail_road WHERE n02_001='21';
CREATE INDEX ON ksj_rail_road_tram USING GiST(geom);
```

QGISで見てみましょう。

![QGISでksj_rail_road_tramを表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/ksj-rail/04-ksj-tram.png)

分かりにくいですね。

地理院地図を使わせてもらいましょう。淡色地図を使います。

![QGISでksj_rail_road_tramと地理院地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/ksj-rail/05-tram-gsimap.png)

だいたい見えてきました。

おそらく宇都宮ライトレールはまだ入ってないのでしょう。そこはいいのですが、大阪って、こんもりするぐらい軌道があったっけ? もしくは、阪堺電車がすごい路線網を持ってるとか?

拡大してみます。

![QGISで大阪市付近のksj_rail_road_tramと地理院地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/ksj-rail/06-tram-gsimap-zoom.png)

ああ、大阪市を南北に貫いて淀川を渡って新大阪駅に至るだけでなく、たぶん千里中央まで行く路面電車…ないわ。

## 大阪メトロは軌道っぽくないので除く

https://ja.wikipedia.org/wiki/%E5%A4%A7%E9%98%AA%E5%B8%82%E9%AB%98%E9%80%9F%E9%9B%BB%E6%B0%97%E8%BB%8C%E9%81%93 には、大阪メトロは軌道法によるとのこと。それだと軌道で引っかかってくるのは正しい。

とはいえ、路面電車を想定しているので、御堂筋線を爆走する10両編成は、さすがに外したい。

ということで、外そう。

```SQL
DELETE FROM ksj_rail_road_tram WHERE n02_004='大阪市高速電気軌道';
```

```
DELETE 248
```

![QGISで大阪市付近のksj_rail_road_tramと地理院地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/ksj-rail/07-tram-gsimap-keihanna.png)

…けいはんな線も外そう。

```SQL
DELETE FROM ksj_rail_road_tram WHERE n02_003='けいはんな線';
```

```
DELETE 8
```

![QGISで大阪市付近のksj_rail_road_tramと地理院地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/ksj-rail/08-tram-gsimap-cleaned.png)

# 参照

* [PostGISを入れるところからやってみよう](../../b1de0a18073af70946e0) / [各種データを ogr2ogr でインポートしてみよう](../../b1de0a18073af70946e0/viewer/import-ogr2ogr)
* [PostGIS入門](../../caea8d4c77dbba2e23a0) / [空間参照系の概要](../../caea8d4c77dbba2e23a0/viewer/srs)
* [PostGIS入門](../../caea8d4c77dbba2e23a0) / [ジオグラフィ型というものがある](../../caea8d4c77dbba2e23a0/viewer/geog)
* [PostGIS入門](../../caea8d4c77dbba2e23a0) / [PostGISの空間関係をテストする関数たち](../../caea8d4c77dbba2e23a0/viewer/testing)

# 出典

本章の作成にあたり、次のデータを使用しました。

* 「筆ポリゴンデータ」 (農林水産省) https://open.fude.maff.go.jp/ (2023年11月28日取得)
* 「国土数値情報 (都市地域データ)」 (国土交通省) https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-A09.html (2023年12月1日取得)


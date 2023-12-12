---
title: "全国の鉄道データ"
---
# はじめに

全国の鉄道データを使って、新幹線の総延長を計測してみたり、路面電車地図を作ったりしてみましょう。

# 必要なデータ

全国の鉄道データは、国土交通省国土政策局が提供する「国土数値情報 (鉄道)」を使います。

  * [国土数値情報のダウンロード](dl-ksj-ua)

# PGDUMPデータ作成とインポート

[PostGISを入れるところからやってみよう](../../b1de0a18073af70946e0) / [各種データを ogr2ogr でインポートしてみよう](../../b1de0a18073af70946e0/viewer/import-ogr2ogr) を参照して下さい。

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

座標参照系については [PostGIS入門](../../caea8d4c77dbba2e23a0) / [空間参照系の概要](../../caea8d4c77dbba2e23a0/viewer/srs) などをご覧下さい。


```
db=# select * from geometry_columns;
 f_table_catalog | f_table_schema |       f_table_name       | f_geometry_column | coord_dimension | srid |    type
-----------------+----------------+--------------------------+-------------------+-----------------+------+------------
...
 db              | public         | ksj_rail_road            | geom              |               2 | 6668 | LINESTRING
 db              | public         | ksj_rail_station         | geom              |               2 | 6668 | LINESTRING
...
```

EPSG:6668 (JGD 2011 地理座標系)のラインストリングであることが確認できました。

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

新幹線の総延長を知りたくなったとします。

新幹線に該当するラインストリングの長さを計算して、それらの合計を計算すると、答が出ます。

答を求めるには、新幹線に該当するラインストリングを抽出することと、長さを計測することと、合計すること、に分かれると思います。

まず、新幹線に該当するラインストリングの抽出を考えます。

https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N02-v3_1.html を見ると、路線の ``n02_002`` (事業車種別) が ``1`` だと新幹線だそうです。

ということは ``WHERE n02_002=1`` でよさそうです。いやよくないです。

```
db=# SELECT Sum(ST_Length(geom::GEOGRAPHY)) FROM ksj_rail_road WHERE n02_002=1;
ERROR:  演算子が存在しません: character varying = integer
LINE 1: ...Length(geom::GEOGRAPHY)) FROM ksj_rail_road WHERE n02_002=1;
                                                                    ^
HINT:  指定した名称と引数の型に合う演算子がありません。明示的な型キャストが必要かもしれません。
```

何が起こったと思ったのですが、よく見ると、文字列型と整数型とで比較しようとしてのエラーでした。ということで ``WHERE n02_002='1'`` にする必要があります。

次いで、長さの計測。これは ``ST_Length()``を使いますが、地理座標系では、単位が「度」という謎の距離しか出ません。

地理座標系の長さや面積などの計測にはジオグラフィ型を使って関数に渡すといいことがあります。[PostGIS入門](../../caea8d4c77dbba2e23a0) / [ジオグラフィ型というものがある](../../caea8d4c77dbba2e23a0/viewer/geog) を参照して下さい。

長さの計測は ``ST_Length(geom::GEOGRAPHY)`` でいけそうです。

最後に合計ですが ``Sum()`` で充分ですね。

まとめると次のようなクエリになります。

```SQL
SELECT Sum(ST_Length(geom::GEOGRAPHY)) FROM ksj_rail_road WHERE n02_002='1';
```

```
        sum
--------------------
 2838077.8721216745
(1 row)
```

https://ja.wikipedia.org/wiki/%E6%96%B0%E5%B9%B9%E7%B7%9A を見ると、フル規格は 2,830km だそうです。思ったよりも近い値が得られました。

## 無軌条鉄道の総延長は?

無軌条鉄道っていうのが若干気になりませんか? 気にならないですか。でも気になったことにして下さい。

無軌条鉄道の総延長を見てみましょう。先ほどは ``n02_002='1'`` でしたが、今度は ``n02_001``が``'17'``のラインストリングを抽出して計算します。あとは先ほどと同じです。

```SQL
SELECT Sum(ST_Length(geom::GEOGRAPHY)) FROM ksj_rail_road WHERE n02_001='17';
```

```
        sum
--------------------
 3749.6225432374376
(1 row)
```

3.7kmですって。1路線しかなさそうです。

調べたら、本当に今は 3.7km の 1路線 だけです。https://ja.wikipedia.org/wiki/%E7%AB%8B%E5%B1%B1%E9%BB%92%E9%83%A8%E8%B2%AB%E5%85%89%E7%84%A1%E8%BB%8C%E6%9D%A1%E9%9B%BB%E8%BB%8A%E7%B7%9A そしてこれも 2024年12月1日 で廃止になるようです。

# 路面電車地図を作ってみよう

今度は「軌道」の話です。総延長を求めずに、地図を作っていきます。

## 軌道のみ抽出する

``n02_001``が``'21'``のラインストリングを抽出します。

```
SELECT * INTO ksj_rail_road_tram FROM ksj_rail_road WHERE n02_001='21';
CREATE INDEX ON ksj_rail_road_tram USING GiST(geom);
```

## QGISで見てみる

QGISで見てみましょう。

![QGISでksj_rail_road_tramを表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/ksj-rail/04-tram.png)

分かりにくいですね。

地理院地図を使わせてもらいましょう。淡色地図を使います。

[PostGISを入れるところからやってみよう](../../b1de0a18073af70946e0) / [WMTSで地域を絞り込んだうえでPostGISレイヤを表示する](../../b1de0a18073af70946e0/viewer/qgis-wmts) を参照して、追加してみて下さい。

![QGISでksj_rail_road_tramと地理院地図を逆順で表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/ksj-rail/05-gsimap-tram.png)

軌道が見えんがな。

これは「レイヤ」ペイン (画面左端の「ksj_rail_road_tram」と「淡色地図」の文字が表示されているところ) で、ドラッグすると、描画順が変わります。「淡色地図」を下にドラッグしてみましょう。

![QGISでksj_rail_road_tramと地理院地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/ksj-rail/06-tram-gsimap.png)

だいたい見えてきました。

だいたい路面電車地図になってそうです。ただ宇都宮ライトレールはまだ入ってないようです。そこは仕方ないです。

でも大阪って、こんもりするぐらい軌道があったっけ? もしくは、阪堺電車がすごい路線網を持ってるとか?

拡大してみます。

![QGISで大阪市付近のksj_rail_road_tramと地理院地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/ksj-rail/07-tram-gsimap-zoom.png)

ああ、大阪市を南北に貫いて淀川を渡って新大阪駅に至るだけでなく、たぶん千里中央まで行く路面電車…ないわ。

## 大阪メトロとけいはんな線は軌道っぽくないので除く

https://ja.wikipedia.org/wiki/%E5%A4%A7%E9%98%AA%E5%B8%82%E9%AB%98%E9%80%9F%E9%9B%BB%E6%B0%97%E8%BB%8C%E9%81%93 には、大阪メトロは軌道法によるとのこと。それだと軌道で引っかかってくるのは正しい。

とはいえ、路面電車を想定しているので、御堂筋線を爆走する10両編成は、さすがに外したい。

ということで、外そう。

```SQL
DELETE FROM ksj_rail_road_tram WHERE n02_004='大阪市高速電気軌道';
```

```
DELETE 248
```

![QGISで大阪市付近のksj_rail_road_tramと地理院地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/ksj-rail/08-tram-gsimap-keihanna.png)

…けいはんな線も外そう。

```SQL
DELETE FROM ksj_rail_road_tram WHERE n02_003='けいはんな線';
```

```
DELETE 8
```

![QGISで大阪市付近のksj_rail_road_tramと地理院地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/ksj-rail/09-tram-gsimap-cleaned.png)

大阪市近辺は阪堺電車だけになりました。これは軌道っぽいです。

## 路面電車地図は実は不完全

路面電車地図から大阪メトロの削除等はできましたが、これではまだ足りません。

たとえば、広電宮島線 (西広島以西) は鉄道営業法に基づく「鉄道」であって、軌道ではありません。このため、広島駅から宮島口まで「路面電車」が走っているのですが、途中までしかこの地図には出てきません。

本気で検討していくと大変になるので、本章はこれにて終了して、あんまり深く考えないようにします。

# おわりに

いかがだったでしょうか。国土数値情報 (鉄道データ) で、新幹線等の総延長の計測を通して、仕様書を見ながら適切な条件を選択したり、ジオグラフィを使った線長計測を行いました。

また、軌道の抽出と一部の削除を通して、データの妥当性の判断は目で見て判断しながら削除して、求めるデータセット (この場合は路面電車地図) に加工する様子をご覧頂きました。この辺は GIS 出身の方よりも PostgreSQL 出身の方の方が慣れているかも知れません。

# 参照

* [PostGISを入れるところからやってみよう](../../b1de0a18073af70946e0) / [各種データを ogr2ogr でインポートしてみよう](../../b1de0a18073af70946e0/viewer/import-ogr2ogr)
* [PostGIS入門](../../caea8d4c77dbba2e23a0) / [空間参照系の概要](../../caea8d4c77dbba2e23a0/viewer/srs)
* [PostGISを入れるところからやってみよう](../../b1de0a18073af70946e0) / [WMTSで地域を絞り込んだうえでPostGISレイヤを表示する](../../b1de0a18073af70946e0/viewer/qgis-wmts)
* [PostGIS入門](../../caea8d4c77dbba2e23a0) / [ジオグラフィ型というものがある](../../caea8d4c77dbba2e23a0/viewer/geog)

# 出典

本章の作成にあたり、次のデータを使用しました。

* 「国土数値情報 (鉄道データ)」 (国土交通省) https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-A09.html (2023年12月8日取得)
* 「地理院タイル」 (国土地理院) https://maps.gsi.go.jp/development/ichiran.html (2023年12月11日取得)

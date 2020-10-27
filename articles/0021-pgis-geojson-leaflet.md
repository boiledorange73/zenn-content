---
title: "GeoJSONのエクスポートとWeb地図アプリケーションの作成"
emoji: "😀"
type: "tech"
topics: [PostGIS, Leaflet, GeoJSON]
published: true
---
# はじめに

PostGISに貯めてあるデータを地図アプリで見れるようにしてみます。地図ライブラリにはLeafletを使い、背景地図には地理院タイルを使います。

PostGISに貯めているデータはGeoJSON形式でエクスポートし、jQueryを使ってロードして、地図上に表示します。

# データ

今回は、国土数値情報（道の駅データ）を使います。[国土数値情報ダウンロードサービス](http://nlftp.mlit.go.jp/ksj/index.html)からダウンロードし、[シェープファイルのデータをインポートしてみよう コマンドライン編](https://zenn.dev/boiledorange73/books/b1de0a18073af70946e0/viewer/import-cli)または[シェープファイルのデータをインポートしてみよう GUI編](https://zenn.dev/boiledorange73/books/b1de0a18073af70946e0/viewer/import-gui)を参考に、PostGISデータベースにインポートして下さい。

テーブル名は、GUIでインポートしたら``p35-18_roadside_station``となったので、そのまま使っています。インポートして生成されたテーブルのテーブル名が違う場合には、ご自身のテーブル名を優先して下さい。

# GeoJSONデータの作成

## GeoJSON ?

たとえば[GeoJSON フォーマット仕様](https://s.kitazaki.name/docs/geojson-spec-ja.html)の、最初の「例」を見て下さい。

まず最上位を見てみます。

```json
{
  "type": "FeatureCollection",
  "features": [(地物1), (地物2), ...]
}
```

``FeatureCollection``という単一のオブジェクトがあり、``features``プロパティに地物の配列が入っています。

地物は次のようになっています。

```json
{
  "type": "Feature",
  "geometry": (ジオメトリ),
  "properties": {(key1): (value1), ...}
}
```

``geometry``プロパティにはジオメトリを、``properties``プロパティには属性を表現するハッシュを、それぞれ入れます。

ジオメトリは``ST_AsGeoJSON()``で得られる文字列にあたります。

## 広島県だけにします

どうも全国データをそのまま出すと読み切れないようなので、広島県だけ抽出するようにします。このため、``WHERE``節で``p35_005 LIKE '34%'``という条件を設定し、市区町村コードの上2桁(=県コード)が34のものだけを抽出するようにしています。

## Featureの部品を表示する

JSON化せずに、type, properties, geometry を出してみます。

```sql
SELECT
  'Feature' AS type,
  json_object(ARRAY['name', p35_006, 'mcode', p35_005]) as properties,
  ST_AsGeoJSON(geom)::json AS geometry
FROM "p35-18_roadside_station"
WHERE  p35_005 LIKE '34%'

  type   |                      properties                       |                        geometry                      
---------+-------------------------------------------------------+---------------------------------------------------------
 Feature | {"name" : "リストアステーション", "mcode" : "34210"}  | {"type":"Point","coordinates":[133.0713142,34.784591]}
...
```

``type``は文字列型です。``properties``と``geometry``は文字列型でなくJSON型です。

## FeatureCollectionの部品を表示する

```sql
SELECT
  'FeatureCollection' AS type,
  json_agg(Q1)::JSON AS features
FROM (
  SELECT
    'Feature' AS type,
    json_object(ARRAY['name', p35_006, 'mcode', p35_005]) as properties,
    ST_AsGeoJSON(geom)::json AS geometry
  FROM "p35-18_roadside_station"
  WHERE  p35_005 LIKE '34%'
) AS Q1

       type        |                                                                          features                  
-------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------
 FeatureCollection | [{"type":"Feature","properties":{"name" : "リストアステーション", "mcode" : "34210"},"geometry":{"type":"Point","coordinates":[133.0713142,34.784591]}},   +
                   |  {"type":"Feature","properties":{"name" : "遊YOUさろん東城", "mcode" : "34210"},"geometry":{"type":"Point","coordinates":[133.2757124,34.88967]}},         +
                   |  {"type":"Feature","properties":{"name" : "さんわ182ステーション", "mcode" : "34545"},"geometry":{"type":"Point","coordinates":[133.2998686,34.6734859]}}, +
                   |  {"type":"Feature","properties":{"name" : "豊平どんぐり村", "mcode" : "34369"},"geometry":{"type":"Point","coordinates":[132.4446982,34.6468579]}},        +
                   |  {"type":"Feature","properties":{"name" : "来夢とごうち", "mcode" : "34368"},"geometry":{"type":"Point","coordinates":[132.2698295,34.5744153]}},          +
                   |  {"type":"Feature","properties":{"name" : "よがんす白竜", "mcode" : "34204"},"geometry":{"type":"Point","coordinates":[132.913945,34.5064709]}},           +
                   |  {"type":"Feature","properties":{"name" : "アリストぬまくま", "mcode" : "34207"},"geometry":{"type":"Point","coordinates":[133.3173083,34.394684]}},       +
                   |  {"type":"Feature","properties":{"name" : "スパ羅漢", "mcode" : "34213"},"geometry":{"type":"Point","coordinates":[132.1042513,34.389119]}},               +
                   |  {"type":"Feature","properties":{"name" : "ゆめランド布野", "mcode" : "34209"},"geometry":{"type":"Point","coordinates":[132.7970555,34.8557897]}},        +
                   |  {"type":"Feature","properties":{"name" : "ふぉレスト君田", "mcode" : "34209"},"geometry":{"type":"Point","coordinates":[132.8494852,34.8990665]}},        +
                   |  {"type":"Feature","properties":{"name" : "クロスロードみつぎ", "mcode" : "34205"},"geometry":{"type":"Point","coordinates":[133.1446622,34.5115157]}},    +
                   |  {"type":"Feature","properties":{"name" : "舞ロードIC千代田", "mcode" : "34369"},"geometry":{"type":"Point","coordinates":[132.5412326,34.6745412]}},      +
                   |  {"type":"Feature","properties":{"name" : "北の関宿安芸高田", "mcode" : "34214"},"geometry":{"type":"Point","coordinates":[132.6810513,34.7210499]}},      +
                   |  {"type":"Feature","properties":{"name" : "湖畔の里福富", "mcode" : "34212"},"geometry":{"type":"Point","coordinates":[132.7759856,34.5322334]}},          +
                   |  {"type":"Feature","properties":{"name" : "たけはら", "mcode" : "34203"},"geometry":{"type":"Point","coordinates":[132.9121681,34.3439384]}},              +
                   |  {"type":"Feature","properties":{"name" : "みはら神明の里", "mcode" : "34204"},"geometry":{"type":"Point","coordinates":[133.1038862,34.3968282]}},        +
                   |  {"type":"Feature","properties":{"name" : "たかの", "mcode" : "34210"},"geometry":{"type":"Point","coordinates":[132.8815981,35.024058]}},                 +
                   |  {"type":"Feature","properties":{"name" : "世羅", "mcode" : "34462"},"geometry":{"type":"Point","coordinates":[133.0841,34.59638]}},                       +
                   |  {"type":"Feature","properties":{"name" : "びんご府中", "mcode" : "34208"},"geometry":{"type":"Point","coordinates":[133.235633,34.56977]}}]
```

``type``は文字列型です。``features``は文字列型でなくJSON型です。

``json_agg()``の引数には、``Q1``を指定しています。これで、``type``、``properties``、``geometry``からなる行を個別のJSONオブジェクトにし、全ての行をまとめてJSON配列にしています。


## FeatureCollectionを表示する

最終的には、次のクエリでGeoJSONが出力できます。

```sql
SELECT to_json(Q) FROM (
  SELECT 'FeatureCollection' AS type,
    json_agg(Q1)::JSON AS features
  FROM (
    SELECT
      'Feature' AS type,
      json_object(ARRAY['name', p35_006, 'mcode', p35_005]) as properties,
      ST_AsGeoJSON(geom)::json AS geometry
    FROM "p35-18_roadside_station"
    WHERE  p35_005 LIKE '34%'
  ) AS Q1
) AS Q;
```

``to_json()``の引数に``Q``を指定していますが、上述の``json_agg()``のような集約関数でなく、Qの1行をひとつのJSONオブジェクトにします。


# 結果をUTF-8でファイルに出力する

```
% psql -t -A -d (データベース名)
db=# \encoding UTF-8
db=# \o (出力ファイル名)
db=# SELECT ...
db=# \q
```

``psql -t -A``の``-t``は結果行だけを表示するオプションで、``-A``は桁そろえ無しで表示するオプションです。

``\encoding UTF-8``で出力エンコーディングを、Shift JISではなく、UTF-8にしています。

``\o data.json``等とすると、data.jsonというファイルが作られます。その直後に``SELECT``を実行すると、結果が、結果行だけ、桁そろえ無しで、UTF-8でファイルに保存されます。

``\q``で終了します。

# スクリプト

Leafletを使って地図表示するスクリプトの例を示します。このスクリプトは、同じフォルダに``data.json``が用意できていなければなりません。

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Leaflet 地図</title>
<script src="https://code.jquery.com/jquery-3.4.1.min.js" integrity="sha256-CSXorXvZcTkaix6Yvo6HppcZGetbYMGWSFlBw8HfCJo=" crossorigin="anonymous"></script>
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.2.0/dist/leaflet.css">
<script src="https://unpkg.com/leaflet@1.2.0/dist/leaflet.js"></script>
<style>
html, body, #MAP {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  position: relative;
}
</style>
<script src="index.js">
var map;
// クリックしたときにproperties.nameを表示する
// https://leafletjs.com/examples/geojson/
function onEachFeature(feature, layer) {
  var str = "";
  if (feature.properties && feature.properties.name) {
    str = feature.properties.name;
  }
  layer.bindPopup(str);
}

// $.getJSON()のイベントハンドラ (JSONが読み込まれたら呼ばれる)
// 引数 data にはオブジェクトが入る
function Data_OnLoad(data) {
  L.geoJson(
    data,
    {onEachFeature: onEachFeature}
  ).addTo(map);
}

// jQuery読み込み完了イベントハンドラ
$(document).ready(
  function() {
    // Leafletのセットアップ
    // https://github.com/gsi-cyberjapan/gsitiles-leaflet/blob/gh-pages/index.html
    map = L.map("MAP");
    L.tileLayer(
      "https://cyberjapandata.gsi.go.jp/xyz/std/{z}/{x}/{y}.png",
      {
        attribution: "<a href=\"https://maps.gsi.go.jp/development/ichiran.html\" target=\"_blank\">地理院タイル</a>"
      }
    ).addTo(map);
    map.setView([35.3622222, 138.7313889], 5);
    // JSONのロード
    $.getJSON("data.json", Data_OnLoad)
  }
);
</script>
</head>

<body>
<div id="MAP"></div>
</body>
</html>
```

# $.getJSON()はローカルも読めるがChromeは起動オプションが必要

jQueryの``$.getJSON()``は、CORSで引っかかります。CORSは"Cross-Origin Resource Sharing"の略です。の説明を参照して下さい。


jQueryの``$.getJSON()``は、FirefoxやEdge上においては、ローカルファイルを取得できますが、Chrome上においては、CORSで引っかかります。
CORSについてはhttps://developer.mozilla.org/ja/docs/Web/HTTP/CORS 等を参照して下さい。

回避するには、オプションに``--allow-file-access-from-files``を指定して起動します。
https://qiita.com/takahiro_itazuri/items/a80a0b3f285d5ada4af7 を参照して下さい。

# できた!

こんなかんじになります。

![POI表示地図アプリ実行画面](https://storage.googleapis.com/zenn-user-upload/uacdmw7x3h40fhe9khfaaqtkv02y)

# おわりに

PostGISを使ってGeoJSON (FeatureCollection)を生成しようとして、``ST_AsGeoJSON()``だけでなく、PostgreSQLのJSON関数も併用したクエリを作り、Leafletを使って地図Webアプリとして表示する方法を示しました。

とりあえず動くはずですので、お試しください。

GeoJSON生成クエリのところは、説明が行き届いておりませんが、具体的にどう行き届いていないのか、よく分かっていないという状況です。

# 出典

この記事では国土政策局「国土数値情報（道の駅データ）」 ( http://nlftp.mlit.go.jp/ksj/ ) を使用しました。

この記事では国土地理院「地理院タイル」 ( https://maps.gsi.go.jp/development/ichiran.html )を使用しました。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

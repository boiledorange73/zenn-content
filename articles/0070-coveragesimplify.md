---
title: "ST_CoverageSimplifyで単純化してみよう"
emoji: "😀"
type: "tech"
topics: [PostGIS,GIS]
published: false
---

# はじめに

これまで ``ST_CoverageSimplify`` の記事を書こうと思いつつサボりまくっていましたが、ここに来てなんとなく実行すると、重なりも隙間もできないうえにめちゃくちゃ早かったので、ここに記す。

# 全国市区町村ポリゴンを叩き込む

[各種データを ogr2ogr でインポートしてみよう](https://zenn.dev/boiledorange73/books/b1de0a18073af70946e0/viewer/import-ogr2ogr) などで使ってきました「国土数値情報（行政区域）」に、またお世話になりましょう。

今回は GeoJSON 形式の方を使います。

```
ogr2ogr -f PGDUMP \
  --config PG_USE_COPY YES \
  -lco SPATIAL_INDEX=GIST \
  -lco FID=gid \
  -lco GEOMETRY_NAME=geom \
  -nln admarea6668 \
  admarea6668.sql N03-20250101.geojson
```

シェープファイルの場合にはエンコーディング指定が可能でしたが、GeoJSONでは ``-oo ENCODING=(文字コードセット)`` には対応していません。[RFC7946 11.1.](https://datatracker.ietf.org/doc/html/rfc7946#section-11.1) から I-JSON に従うべき（必須ではない）で、[RFC7493 2.1](https://datatracker.ietf.org/doc/html/rfc7493#section-2.1) によると UTF-8 でないといけないとなっていますので、まあ基本的に UTF-8 と見ていいだろうということなのだろうと思います。

次にデータベースを作って PostGIS をインストールして、出来上がった SQL ファイルを叩き込みます。

```
createdb testdb
psql -d testdb -c "create extension postgis"
psql -d testdb -f admarea6668.sql
```

これで ``admarea6668`` というテーブルが出来上がります。

# 投影座標系に変換

簡略化の距離をしていするために、投影座標系に変換します。ここでは EPSG:3395 を使うこととします。

``admarea3395`` を作って、ここに投影返還したジオメトリと属性とを入れます。あと、属性名が数字だとわかりにくいので、文字にしました。

```
CREATE TABLE admarea3395(
  gid SERIAL PRIMARY KEY,
  mcode TEXT, -- n03_007
  pname TEXT, -- n03_001
  sname TEXT, -- n03_002 subprefecture
  gname TEXT, -- n03_003 gun
  mname TEXT, -- n03_004 municipality
  wname TEXT, -- n03_005 ward
  geom GEOMETRY(MULTIPOLYGON, 3395)
);
CREATE INDEX ON admarea3395 USING GiST(geom);

INSERT INTO admarea3395(mcode, pname,sname,gname,mname,wname,geom)
SELECT n03_007,n03_001,n03_002,n03_003,n03_004,n03_005, ST_Transform(geom, 3395) geom
FROM admarea6668
ORDER BY gid;
```

ここで、マルチポリゴン数は ``124094`` となっていることを確認してください。

```
INSERT 0 124094
```

# 簡略化

```
CREATE TABLE simplified3395(
  gid SERIAL PRIMARY KEY,
  mcode TEXT, -- n03_007
  pname TEXT, -- n03_001
  sname TEXT, -- n03_002 subprefecture
  gname TEXT, -- n03_003 gun
  mname TEXT, -- n03_004 municipality
  wname TEXT, -- n03_005 ward
  geom GEOMETRY(MULTIPOLYGON, 3395)
);
CREATE INDEX ON simplified3395 USING GiST(geom);
```

こんかんじになります。

```
INSERT INTO simplified3395(gid, mcode, pname, sname, gname, mname, wname,geom)
SELECT gid, mcode, pname, sname, gname, mname, wname, ST_CoverageSimplify(geom, 100) OVER()
FROM admarea3395
ORDER BY gid;
```

だいたい **70秒** ぐらいかかりました。めっちゃくちゃ早いじゃないか!?
全国の市区町村でトポロジーを構築すると1日はかかりました。今のパソコンと CPU パワーは違うのですが、さすがに桁が違いすぎる。


# じゃあちゃんとオーバーラップや隙間が無いかを確認

## ST_CoverageInvalidEdges でチェック

https://postgis.net/docs/ja/ST_CoverageInvalidEdges.html を見ながら、不正なエッジを持つ gid を抽出するクエリ例を作ってみました。

```
WITH coverage(gid, geom) AS (VALUES
  (1, 'POLYGON ((10 190, 30 160, 40 110, 100 70, 120 10, 10 10, 10 190))'::geometry),
  (2, 'POLYGON ((100 190, 10 190, 30 160, 40 110, 50 80, 74 110.5, 100 130, 140 120, 140 160, 100 190))'::geometry),
  (3, 'POLYGON ((140 190, 190 190, 190 80, 140 80, 140 190))'::geometry),
  (4, 'POLYGON ((180 40, 120 10, 100 70, 140 80, 190 80, 180 40))'::geometry)
)
SELECT gid FROM (
  SELECT gid, ST_CoverageInvalidEdges(geom) OVER () result
  FROM coverage
) Q1
WHERE result IS NOT NULL;
```

今度は ``simplified3395`` で不正なエッジが無いか確認します。

```
SELECT gid FROM (
  SELECT gid, ST_CoverageInvalidEdges(geom) OVER () result
  FROM simplified3395 order by gid
) Q1
WHERE result IS NOT NULL;
```

```
 gid 
-----
(0 rows)
```

無いようですね。よかった。``simplified3395``のチェックで10秒程度かかりました。

なお、``admarea3395``では3分半ぐらいかかりました。不正なエッジはありませんでした。

## 最後に画像を見る

QGISで透明度50%のレイヤを作ります。こうすると重複部分の色が濃くなります。

では見てみましょう。

![重なっていない証拠の図](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0070/01-result.png)

いいかんじですね。


# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

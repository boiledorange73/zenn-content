---
title: "市街化区域内にある農地"
---
# はじめに

ある市について、市街化区域内にある農地の筆数と面積を集計したいと思いました。

というのも、農地法5条にある通り、農地を農地以外のものにする(以下「転用」といいます）には、都道府県知事の許可が必要です。しかし農地法5条1項但書および同項6号から、都市計画法7上市街化区域内にある農地は農業委員会に届け出れば、都道府県知事の許可なく転用できます。このため、市街化区域内は、そうでない区域よりも農地が減りやすいと考えられます。

ただし、農地のGISデータはあるけれども、そこに市街化区域に入っているかどうかを示す属性データは無いものとします。

# 必要なデータ

農地のGISデータは、ここでは、農林水産省大臣官房統計部が提供する「筆ポリゴン」を使います。

また、都市計画区域のGISデータは、国土交通省国土政策局が提供する「国土数値情報 (都市地域)」を使います。

  * [筆ポリゴンのダウンロード](dl-fpoly)
  * [国土数値情報のダウンロード](dl-ksj-ua)

# PGDUMPデータ作成とインポート

[PostGISを入れるところからやってみよう](../../b1de0a18073af70946e0)にある[各種データを ogr2ogr でインポートしてみよう](../../b1de0a18073af70946e0/viewer/import-ogr2ogr)を参照して下さい。

## 筆ポリゴン

```batch
ogr2ogr -f PGDUMP ^
  --config PG_USE_COPY YES ^
  -lco SPATIAL_INDEX=GIST ^
  -lco GEOMETRY_NAME=geom ^
  -nln fude ^
  fude.sql 2023_342076.json
```

```batch
> set PGCLIENTENCODING=UTF8
> psql -d db -f fude.sql
```

## 国土数値情報 (都市地域)

```batch
ogr2ogr -f PGDUMP ^
  --config PG_USE_COPY YES ^
  -lco SPATIAL_INDEX=GIST ^
  -lco GEOMETRY_NAME=geom ^
  -oo ENCODING=CP932 ^
  -nln planing  ^
  planing.sql 34207_福山市.shp
```

```batch
> set PGCLIENTENCODING=UTF8
> psql -d db -f planing.sql
```

# QGISで見てみる

まず、``planing``テーブル (都市計画区域)を見てみましょう。

![QGISでplaningの地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook-editing/fpoly-in-uarea_01-planing.png)

いい眺めですね。

次に、``fude``テーブル (筆ポリゴン)を表示して見ましょう。

![QGISでfudeの地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook-editing/fpoly-in-uarea_02-fude.png)

いい眺めですね。

最後に、重ね合わせてみましょう。

![QGISでplaningとfudeの地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook-editing/fpoly-in-uarea_03-planing_fude.png)

素晴らしい眺めですね。

# 対象となるポリゴンの抽出

今回は、筆ポリゴンの農地データのうち市街化区域内にあるものの面積を求める、というものです。

その前段階(かつ中核部分)である、対象となるポリゴンの抽出を行いましょう。

まず、そのままでは動かないクエリを提示します。やりたいことは、これだろうと思います。

```SQL
SELECT fude.* FROM fude WHERE 市街化区域内にある(fude.geom);
```

## 市街化区域はどこだ?

まず「市街化区域内にある」かどうかをテストするには、市街化区域がどこかを知る必要があります。

市街化区域はどこだ? と考えて、なんとなく``planing``テーブルの地図で、赤色で塗りつぶした領域でないかと思われた方がいらっしゃったかも知れません。あたりです。

``layer_no`` という整数型のカラムで ``1`` だと「市街化区域」、``2``だと「市街化調整区域」、``3``は「その他用途地域」、``4``は「用途未設定」、とされています。詳しくは https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-A09.html を参照して下さい。

市街化区域を抽出するSQLは次の通りです。

```SQL
SELECT planing.* FROM planing WHERE layer_no=1;
```

## 市街化区域内にあるポリゴンの抽出に失敗した

次のクエリを実行してみましょう。

```SQL
SELECT fude.* FROM planing, fude
  WHERE planing.layer_no = 1
    AND ST_Contains(planing.geom, fude.geom);
```

```
ERROR:  contains: Operation on mixed SRID geometries (Polygon, 6668) != (Polygon, 4326)
```

エラーが出ました!

SRID は "Spatial Reference System Idenfifier" の略（"System"が抜けてる）で、日本語では「空間参照系識別子」となります。また、空間参照系と座標参照系は同じと見て下さい。

そのうえで先ほどのエラーメッセージを見直してみると、ジオメトリの座標参照系が、一方は``6668``で、もう一方は``4326``となって混合していることがエラーの理由のようです。


# 座標参照系が違うと言われたら

座標参照系が違うものを混ぜようとするとエラーが発生します。座標参照系が違うと、座標値の意味が全く異なります。

エラーをなくしたいなら、自分で座標参照系を変更する必要があります。

なお、日本国内でよく使われる座標参照系については、[PostGIS入門](../../caea8d4c77dbba2e23a0)にある[空間参照系の概要](../../caea8d4c77dbba2e23a0/viewer/srs)を参照して下さい。


## 座標参照系を確認する

前の節で「ジオメトリの座標参照系が、一方は``6668``で、もう一方は``4326``となって混合している」と言いましたが、それが合っているかどうか確認しましょう。

``geometry_columns`` を見るといいです。

```SQL
SELECT * FROM geometry_columns;
```

```
 f_table_catalog | f_table_schema | f_table_name | f_geometry_column | coord_dimension | srid |  type
-----------------+----------------+--------------+-------------------+-----------------+------+---------
 db              | public         | fude         | geom              |               2 | 4326 | POLYGON
 db              | public         | planing      | geom              |               2 | 6668 | POLYGON
(2 rows)
```

``fude.geom``は``4326`` (WGS84経度緯度)、``planing.geom``は``6668`` (JGD2011経度緯度) で、確かに違います。

## 空間参照系を変更する方法は2種類ある

座標参照系を変更する方法には2種類あります。

``ST_SetSRID(geom, <変換先SRID>)`` によって、座標参照系識別子を差し替えるだけか、``ST_Transform(geom, <変換先SRID>)``で、座標変換を行うか、です。

座標参照系識別子を差し替えていいのは、座標参照系は確かに違うけれども座標値がほぼ同じで通用する場合ぐらいです。

たとえば、``4326`` (WGS84経度緯度) と ``6668`` (JGD2011経度緯度) とを、``ST_SetSRID()``で相互変換するのは、2011年5月以降に作成されたデータならば、ミリ単位とかのよほど高い精度を求められる場合を除いて許されると思っていいです。また、2003年1月1日から2011年3月11日までなら、``4326`` (WGS84経度緯度) と ``4612`` (JGD2000経度緯度) とを``ST_SetSRID()``で相互変換するのは、これもすごく高い精度でもないなら、許されると思っていいです。

しかし、``4326``と``4301``は、そのままではダメです。400メートルはズレます。この場合は ``ST_Transform()``で座標変換を行う必要があります。

## どちらを使うといいか

``ST_Transform()``の方が、``ST_SetSRID()``が適用できる場合でも、そうでない場合でも、どちらでも適用できるので、とりあえず使うならこちらになると思います。

しかし、座標変換は非常に多くの量の計算が必要ですので、確実に大丈夫そうでしたら、``ST_SetSRID()``を優先した方がいいように思います。

また、座標参照系を``EPSG:4301``としてデータを作っているはずなのに``EPSG:6668``と指定しているるデータが存在する可能性があります。この場合には、``ST_SetSRID()``一択です。

筆ポリゴンは2023年に作られたデータですので、``4326``といっても``6668``とほぼ変わらないので、``ST_SetSRID(geom, 6668)``で済むと判断できます。

## fude6668 を作成する

最も手っ取り早いのが、``SELECT INTO``で全体をコピーして、SRID制約を切って、geomのSRIDを変更して、SRID制約を付け直す方法だろうと思います。

```SQL
-- fude→fude6668 コピー (fude6668.geomのSRID=4326)
SELECT  * INTO fude6668 FROM fude;
-- fude6668.geomのSRID制約を0に設定 (=制限なし)
ALTER TABLE fude6668 ALTER COLUMN geom TYPE GEOMETRY(POLYGON, 0);
-- fude6668.geomのSRIDを6668に付け直し
UPDATE fude6668 SET geom=ST_SetSRID(geom, 6668);
-- fude6668.geomのSRID制約を6668に設定
ALTER TABLE fude6668 ALTER COLUMN geom TYPE GEOMETRY(POLYGON, 6668);
-- インデックス作成
CREATE INDEX ON fude6668 USING GiST(geom);
```

## fude6668とplaningだと座標参照系エラーが出ないのを確認

```SQL
SELECT fude6668.* FROM planing, fude6668
  WHERE planing.layer_no = 1
    AND ST_Contains(planing.geom, fude6668.geom);
```

# テーブルに叩き込んで完成

```SQL
SELECT fude6668.* INTO fude_containedby_planing FROM planing, fude6668
  WHERE planing.layer_no = 1
    AND ST_Contains(planing.geom, fude6668.geom);

CREATE INDEX ON fude_containedby_planing USING GiST(geom);
```

## QGISで見る

結果をQGISで見てみましょう。

![QGISで市街化区域に含まれる農地を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook-editing/fpoly-in-uarea_04-fude_containedby_planing.png)

もうちょっとズームして見ましょう。

![QGISで市街化区域に含まれる農地の一部をズームして表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook-editing/fpoly-in-uarea_05-fude_containedby_planing_zoom.png)


## 面積の合計を見る

面積を得る関数は ``ST_Area()``なのですが、ジオメトリ型の場合には、地理座標系 (経度緯度) の面積が計算できるっぽいのですが、単位が「度の2乗」という、謎な面積が得られます。

でなくて、まともな面積を測りたいですね。その場合は、ジオグラフィ型を使うといいです。[PostGIS入門](../../caea8d4c77dbba2e23a0)の[ジオグラフィ型というものがある](../../caea8d4c77dbba2e23a0/viewer/geog)を参照して下さい。

ジオグラフィ型にキャストして``ST_Area()``に与えるだけで、平方メートル単位の面積計算ができますし、キャスト自体は中身を変えずに型だけ変えているので、キャストについては大して計算コストがかからないので、結構お気軽に使えます。

```SQL
SELECT Sum(ST_Area(geom::GEOGRAPHY)), Count(ST_Area(geom::GEOGRAPHY)), Avg(ST_Area(geom::GEOGRAPHY)) FROM fude6668;
```

```
        sum        | count
-------------------+-------
 42350579.40272802 | 82735
(1 row)
```

単位は平方メートルですので、この市では、市街化区域内に 4235[ha] の農地がある、ということが分かりました。


## テスト関数の選択で結果が変わる

次のクエリを実行して、インタセクトする(=共有部分がある)かどうかをテストする関数を使うと、どうなるでしょう。

次のクエリを実行してみます。

```SQL
SELECT fude6668.* INTO fude_intersects_planing FROM planing, fude6668
  WHERE planing.layer_no = 1
    AND ST_Intersects(planing.geom, fude6668.geom);

CREATE INDEX ON fude_intersects_planing USING GiST(geom);
```

``ST_Contains()``を使った場合はこう見えます。

![QGISで市街化区域に含まれる農地の一部をズームして表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook-editing/fpoly-in-uarea_05-fude_containedby_planing_zoom.png)

``ST_Intersects()``を使った場合はこう見えます。

![QGISで市街化区域とインタセクトする農地の一部をズームして表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook-editing/fpoly-in-uarea_06-fude_intersects_planing_zoom.png)

はみ出した農地もカウントに入ってしまいますね。この場合は、おそらく``ST_Contains()``の方が適切でないかなと思います。適切なテスト関数は求めたいものによって異なりますので、その時その時に検討して下さい。

テスト関数の詳細については、[PostGIS入門](../../caea8d4c77dbba2e23a0)にある[PostGISの空間関係をテストする関数たち](../../caea8d4c77dbba2e23a0/viewer/testing)を参照して下さい。

# 参照

* [PostGISを入れるところからやってみよう](../../b1de0a18073af70946e0) / [各種データを ogr2ogr でインポートしてみよう](../../b1de0a18073af70946e0/viewer/import-ogr2ogr)
* [PostGIS入門](../../caea8d4c77dbba2e23a0) / [空間参照系の概要](../../caea8d4c77dbba2e23a0/viewer/srs)
* [PostGIS入門](../../caea8d4c77dbba2e23a0) / [ジオグラフィ型というものがある](../../caea8d4c77dbba2e23a0/viewer/geog)
* [PostGIS入門](../../caea8d4c77dbba2e23a0) / [PostGISの空間関係をテストする関数たち](../../caea8d4c77dbba2e23a0/viewer/testing)

# 出典

本章の作成にあたり、次のデータを使用しました。

* 「筆ポリゴンデータ」 (農林水産省) https://open.fude.maff.go.jp/ (2023年11月28日取得)
* 「国土数値情報\ (都市地域データ)」 (国土交通省) https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-A09.html (2023年12月1日取得)


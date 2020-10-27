---
title: "空港近くの水平表面をつくる"
emoji: "😀"
type: "tech"
topics: [PostGIS, QGIS]
published: false
---
# はじめに

現在、人口密集地域と空港周辺での無人航空機の使用は許可制です。このうち、人口密集地域は人口集中地区（国勢調査の結果に基づいて定められる）とされ、また、空港周辺は制限表面とされています。

ドローンユーザの方は、ここにきて、DID（人口集中地区）や制限表面が気になってきた方もいらっしゃるのではないでしょうか。

今回は、制限表面のうち、水平表面を、国土数値情報（空港）を使って、作り出してみます。

なお、制限表面はこれだけではありません。たとえば https://ja.wikipedia.org/wiki/%E5%88%B6%E9%99%90%E8%A1%A8%E9%9D%A2 を見てみてください。
また、今回の処理は、人口密集地域かどうかとは全く関係ありませんので、水平表面から外れれば常に無人航空機を飛ばして良いというわけではありません。

なにより、これは制限表面の概要を見るためのものです。実際に無人飛行機を使う際には、違法業務にならないように注意して下さい。あやしい場合には、確実に確認ができるところに問い合わせて下さい。

# 準備

## ダウンロードと格納

国土数値情報から空港GMLをダウンロードしてきます。
unzipをすると、C28-14_Airport.shp と C28-14_AirportReferencePoint.shp が現れます。これらをデータベースに叩き込みます。

```csh
% shp2pgsql -s 4612 -D -i -I -W cp932 C28-14_GML/C28-14_Airport.shp airport | psql -d (データベース名)
% shp2pgsql -s 4612 -D -i -I -W cp932 C28-14_GML/C28-14_AirportReferencePoint.shp airportreference | psql

```

## 多重チェック

多重チェックをしてみましょう。もしかしたらひとつの空港が2つに分割されているかも知れません。

```psql
db=# SELECT COUNT(*) FROM airport;
 count 
-------
    97
(1 行)

db=# SELECT COUNT(*) FROM (SELECT DISTINCT c28_005 FROM airport) AS Q;
 count 
-------
    96
(1 行)
```

ひとつ重複がありました。

```psql
db=# SELECT * FROM (
  SELECT
    c28_005,
    (SELECT COUNT(*) FROM airport WHERE airport.c28_005=Q1.c28_005) AS count
  FROM
    (SELECT DISTINCT c28_005 FROM airport) AS Q1
) AS Q2
WHERE count >= 2;

 c28_005  | count 
----------+-------
 紋別空港 |     2
(1 行)
```

紋別空港でした。

今回はc28_101のみ使うので、これも同じならDISTINCTを使って重複を消し去ってOKです。

```psql
db=# SELECT c28_005, c28_101 FROM airport WHERE c28_005='紋別空港';

 c28_005  |   c28_101   
----------+-------------
 紋別空港 | #cf03_00014
 紋別空港 | #cf03_00014
(2 行)
```

大丈夫そうですね。

# 標点を持つ空港テーブルを作る

水平表面は標点から4000m圏内と決まっているので、標点のポイントデータが必要です。これは標点テーブルにあります。見てみましょう。

```psql
db=# SELECT * FROM airportreference;
 gid |  c28_000   |                                 geom                                 
-----+------------+----------------------------------------------------------------------
   1 | cf03_00001 | 01040000200412000001000000010100000082C8224D3C3D6040B821C66B5E754040
...
(96 行)
```

うーん、空港名が無い。
空港名(airport.c28_005)と標点ID (airportreference.c28_000)とをつなげないといけなくなりました。
airport側に標点IDは、c28_101です。これで、空港名と標点IDのテーブルができますね。

```psql
db=# SELECT DISTINCT c28_005, c28_101 FROM airport;
    c28_005     |   c28_101   
----------------+-------------
 北大東空港     | #cf03_00058
...
(96行)
```

DISTINCTで96行になっているので、紋別空港の重複エントリ問題はクリアされているようです。

ただ、airport.C28_101には先頭に"#"がついていて、そのままでは連結できません。

先頭1文字を削るには、PostgreSQL関数substring()を使います。

```psql
db=# SELECT DISTINCT c28_005, substring(c28_101 from 2) AS refcode FROM airport;
    c28_005     |  refcode   
----------------+------------
 与論空港       | cf03_00057
...
(96行)
```

airportとairportreferenceを結合して、空港名と標点のポイントからなるデータを作りましょう。

```psql
db=# SELECT Q1.c28_005, airportreference.geom
  FROM
  (SELECT DISTINCT c28_005, substring(c28_101 from 2) AS refcode FROM airport) AS Q1
  LEFT JOIN
  airportreference
  ON Q1.refcode=airportreference.c28_000;


    c28_005     |                                 geom                                 
----------------+----------------------------------------------------------------------
 長崎空港       | 01040000200412000001000000010100000082C8224D3C3D6040B821C66B5E754040
...
(96行)
```

ビューにして見ましょう。

```psql
db=# CREATE VIEW v_refpt AS
SELECT Q1.c28_005, airportreference.geom
  FROM
  (SELECT DISTINCT c28_005, substring(c28_101 from 2) AS refcode FROM airport) AS Q1
  LEFT JOIN
  airportreference
  ON Q1.refcode=airportreference.c28_000;
```

# ビューをQGISで見る

これをQGISで見てみましょう。

[QGISでPostGISのデータを見てみよう](https://zenn.dev/boiledorange73/books/b1de0a18073af70946e0/viewer/qgis)を参考に、「テーブル追加」ダイアログまで出して下さい。

いろいろテストしてるビューなりテーブルがあるのですがそれは無視して、v_refptが灰色文字になっていることに注意して下さい。このままではレイヤとして選択できません。

![v_refptがグレーアウトしているところ](https://storage.googleapis.com/zenn-user-upload/6ax51exqyh56jv3o1scaljeg32fq)

これは、QGISからは、ビューの行を識別する値が入っているカラムが分からないためです。QGISは行を識別する値が入っている（ユニークである）カラムが無いとレイヤにできません。QGISはテーブルに対しては主キーとなっているカラムを自動で探してくれるので、テーブルを表示する場合には問題になりませんが、ビューについてはユーザが指し示してやる必要があります。

ここで、v_refptの行の「主キー」列をクリックして下さい。

![v_refptの主キー列をクリックしたところ](https://storage.googleapis.com/zenn-user-upload/msqpsa6ji7c9j2irnolmtoznn21l)

「主キー」候補が現れますので、今回は c28_005 (空港名)を主キーとします。

フォーカスを移すと、灰色文字から黒文字になったことが確認できます。

![v_refptが選択可能になったところ](https://storage.googleapis.com/zenn-user-upload/p25hpm9jfm0os47oq2np0b5pj948)

こうなったら追加可能となります。

![空港地図](https://storage.googleapis.com/zenn-user-upload/y9owp3jfknyeygsy0mgw0dbgiir5)

# 半径4000mの円を作る

次いで、半径4000mの円を作ります。ST_Bufferに標点と距離(4000m)を与えます。

ST_Bufferは、図形と距離を指定されると、指定された図形から指定された距離の範囲内で形成される領域図形を返します。図形に点を与えた場合は円を返します。

## ジオメトリ型の地理座標系ポイントは使ってはいけない

ここで、ジオメトリ型の地理座標系ポイントを中心とした半径4000の円を描いて見ます。

```psql
db=# SELECT ST_Buffer('SRID=4612;POINT(135 35)', 4000);
```

結果は次のようになります。概ねの位置が分かるよう、基盤地図情報WMS配信サービス乗せています。

![巨大な円](https://storage.googleapis.com/zenn-user-upload/1vlvtee55i6635k0qzeyw33dnvwl)

日本どこー？

日本列島全体が見れる程度に縮尺を大きくしてみます。

![日本列島と巨大な円の一部](https://storage.googleapis.com/zenn-user-upload/i4fv5f2oj43qvdpr5bp96yqhgdsa)

ああ、円の中に日本列島がすっぽり入っているわけですね。
…4000mの円なのに？

ジオメトリ型を引数にとる関数で数字が出てくる場合、その数字の単位は、ジオメトリの空間参照系の単位です。第2引数引数の"4000"は、地理座標系のジオメトリについては、4000度となります。地球10周を超える円になっていて、地球上に本来ならあってはならない円になります。

## ジオグラフィ型でやってみる


[ジオグラフィ型というものがある](https://zenn.dev/boiledorange73/books/caea8d4c77dbba2e23a0/viewer/geog)において、距離計測関数``ST_Distance``でジオグラフィ型を引数にとった場合には、返り値の単位がメートルになることを示しました。

ジオグラフィ型を引数としたST_Bufferで同じようにすればどうなるでしょう。第2引数の単位がメートルであることが期待できますね。ただし、実装されていれば、の話ですが。

```psql
db=# SELECT ST_Buffer('SRID=4612;POINT(135 35)'::GEOGRAPHY, 4000);
```

改めて重ねて表示してみましょう。

![地図と4000メートル円](https://storage.googleapis.com/zenn-user-upload/opuvlcy3hh6q9he3dpbzvfey7lu1)

うまくいきましたね。

ジオグラフィ型をとる``ST_Buffer``が実装されていて、第2引数はメートル単位となっていて、座標系の単位（この場合は度）ではないことが分かりました。

# 水平表面のビューを作る

最後のまとめとして、水平表面のビューを作ってみましょう。

```psql
db=# CREATE VIEW v_hs AS
SELECT
  Q1.c28_005,
  ST_SetSRID(
    ST_Buffer(
      airportreference.geom::GEOGRAPHY,
      4000
    )::GEOMETRY,
    4612
  )
  FROM (
    SELECT DISTINCT c28_005, substring(c28_101 from 2) AS refcode
    FROM airport
  ) AS Q1 LEFT JOIN
  airportreference
  ON Q1.refcode=airportreference.c28_000;
```

ST_SetSRIDのあたりが不思議なかんじがすると思います。
この式の内側から見ていきましょう。

まず、airportreference.geom をジオグラフィ型に変換します。

```postgres
      airportreference.geom::GEOGRAPHY,
```

ジオグラフィ型を引数にとるST_Bufferを呼んで、半径4000mの円を生成します。

```postgres
    ST_Buffer(
      airportreference.geom::GEOGRAPHY,
      4000
    ),
```

ST_Bufferの返り値はジオグラフィ型です（なお引数がジオメトリ型なら返り値もジオメトリ型です）。これをジオメトリ型に変換します。

```postgres
    ST_Buffer(
      airportreference.geom::GEOGRAPHY,
      4000
    )::GEOMETRY,
```

ジオメトリ型を引数にとるST_Bufferの返り値の空間参照系IDは4326 (WGS84地理座標系)に固定されています（これは今後の版で変わるかもしれません）。そこで、空間参照系IDを 4612 (JGD2000地理座標系)に変更しています（座標変換は行っていません）。

```postgres
  ST_SetSRID(
    ST_Buffer(
      airportreference.geom::GEOGRAPHY,
      4000
    )::GEOMETRY,
    4612
  )
```

これの結果をQGISで見てみましょう。なお、前述のとおり、主キー相当カラムは手動で指定しなければなりません。

![広島空港近辺の地図と4000メートル円](https://storage.googleapis.com/zenn-user-upload/04lnye56t4d7ebbnorrg2ovc3gt6)

# おわりに

標点データから水平表面を作ってみました。

ビューをQGISで表示する方法を示し、また、ジオグラフィを引数にした``ST_Buffer``を使いました。

手順の分量が多いように見えますが、インポートと重複チェックがあり、かつQGISの使用方法も記述されているためです。

核となる、標点から水平平面を作るのは``ST_Buffer``だけです。しかも、深く考えずに、ジオグラフィにキャストした標点と、数値（今回は``4000``）を引数に与えただけです。
こういうのをさらっとやってくれるのがPostGISの素敵なところです。

しかしながら、水平平面だけで制限表面をカバーできたわけではありません。次は進入表面を無理に作ってみたいと思います。

# ご注意

これは概要を示すためのものです。実際に無人飛行機を使う際には、違法業務にならないように注意して下さい。あやしい場合には、確実に確認ができるところに問い合わせて下さい。

# 出典

本稿では、国土交通省国土政策局発行「国土数値情報 空港データ」を使用しました。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

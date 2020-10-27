---
title: "PostGISユーザがMySQL 8の空間拡張機能を少し触ってみた"
emoji: "😀"
type: "tech"
topics: [GIS, MySQL]
published: true
---
# はじめに

https://www.slideshare.net/yoyamasaki/mysql-57gis-80822317 を見て「どうもMySQLの空間拡張の機能が随分良くなった」ようだと知りました。また、https://www.slideshare.net/NorvaldRyeng/mysql-80-gis-are-you-ready で、もう少し踏み込んだ説明もありました。

これはちょっと触ってみなければいけないと思った次第です。

## 使用したもの

基本的に MySQL 8.0.3-rc Windows x64用 zip版を使いました。インストール方法は http://www.uetyi.com/server-const/entry-1650.html を見ました。

ST_Transform()の記述では、MySQL 8,0.20 Windows x64用 zip版も使用しています(2020年6月18日)。

# 以前のバージョンとの比較

## 空間参照系の導入

空間参照系が導入されました。

```mysql
mysql> SELECT ST_GeomFromText('POINT(35 135)', 4326);
+----------------------------------------+
| ST_GeomFromText('POINT(35 135)', 4326) |
+----------------------------------------+
| （バイナリデータ）                     |
+----------------------------------------+
1 row in set (0.00 sec)
```

座標軸の順序については、後で書きます。

## 地理座標系を前提とした計測関数の導入

```mysql
mysql> SELECT ST_Distance(
    ST_GeomFromText('POINT(35 135)', 4326),
    ST_GeomFromText('POINT(36 136)', 4326)
  ) AS distance;
+--------------------+
| distance           |
+--------------------+
| 143321.89121904946 |
+--------------------+
1 row in set (0.00 sec)
```

通常はユークリッド距離を取るものですが、上記の通り、地理座標系であるかを判定して、地表面に沿った距離を計測しています。


## テスト関数が実体の比較になった

``ST_Intersects()``の第1引数は、10*10の正方形に8*8の正方形の穴を開けた、ドーナツ型です。第2引数は、6*6の正方形です。第1引数のポリゴンの穴の中に第2引数のポリゴンがあるので、``0``を返すのが正しい動作です。


```mysql
mysql> SELECT ST_Intersects(
    ST_GeometryFromText('POLYGON((0 0, 0 10, 10 10, 0 10, 0 0),(1 1, 1 9, 9 9, 1 9, 1 1))'),
    ST_GeometryFromText('POLYGON((2 2, 2 8, 8 8, 8 2, 2 2))')
  ) AS intersects;
+------------+
| intersects |
+------------+
|          0 |
+------------+
1 row in set (0.00 sec)
```

これが、以前は、MBR (Minimum Bounding Rectangle)ベースでした。ジオメトリを包含する長方形で最も小さいもの、と定義されます。この場合には、10*10の正方形がMBR (穴が無い)と6*6の正方形とのインタセクトのテストとなるので、``1``を返していました。

# PostGISとの相違点

## axis orderが違う

2次元の位置を特定するには、X値とY値が必要ですが、X-Yの順の場合と、Y-Xの順の場合とがあります。

「Y-Xの順なんかおかしいやろ」と思ったあなたは、北を上にした標準的な地図と「北緯〇度東経△度」という言い回しを思い出して下さい。Y-Xの順もとりうるのです。

この順序をaxis orderと言います。

で、先ほど出た「北緯〇度東経△度」について、PostGISでは、「△,〇」の順とし、Xを経度(横)、Yを緯度(縦)に取ります。MySQLでは、「〇,△」の順とし、Xを緯度(縦)、Yを経度(横)に取ります。

たいていのGISでは経度-緯度の順です。対象の座標系が何であっても横-縦の順にしています。PostGISはこちら側です。

対して、OGC (Open Geospatial Consortium, 地理空間情報に関する標準化団体)は、axis orderは、その座標系の定義による、としています。EPSG:4326等の地理座標系は、経度-緯度の順ですので、MySQLはこちらです。

先に出てきていた ``ST_GeomFromText('POINT(35 135)', 4326)`` を改めて見てみましょう。MySQLでは「北緯35度東経135度」となり、兵庫県内の位置を指します。PostGISでは「東経35度北緯135度」となり、北極を越えるどこかになってします。

## GEOGRAPHY型が無いが問題無い

地理座標系であるか投影座標系であるかで、計測関数が全く違ってくるので、関数に伝えてやる必要があります。PostGISの場合には、地理座標系にはGEOGRAPHY型を使うようにしていますが、MySQLではGEOGRAPHY型が無く、SRIDで判定しています。

投影座標系のひとつである EPSG:3857 (Webメルカトル)の``POINT(35 135)``と``POINT(36 136)`` (ギニア湾) で``ST_Distance()``を実行してみます。これはユークリッド距離を取り、以下のようになります。

```mysql
mysql> SELECT ST_Distance(
    ->     ST_GeomFromText('POINT(35 135)', 3857),
    ->     ST_GeomFromText('POINT(36 136)', 3857)
    ->   ) AS distance;
+--------------------+
| distance           |
+--------------------+
| 1.4142135623730951 |
+--------------------+
1 row in set (0.00 sec)
```

EPSG:4326では、次のようになります (兵庫県内)。

```mysql
mysql> SELECT ST_Distance(
    ST_GeomFromText('POINT(35 135)', 4326),
    ST_GeomFromText('POINT(36 136)', 4326)
  ) AS distance;
+--------------------+
| distance           |
+--------------------+
| 143321.89121904946 |
+--------------------+
1 row in set (0.00 sec)
```


## ST_Transformが弱い

(2020年6月18日更新)

以前、少なくとも 8.0.3-rc では、``ST_Transform(geometry, int)``がありませんでした。これは困りました。

その後、元記事で @yyamasaki1 さんが 2018年11月10日に

> 8.0.13でST_Transform()が実装されました

とコメントを頂きました。が、8.0.20でも確認しましたが、地理座標系のみ対応のようです。

## 内部表現の差

これは余談というかマニア向けというかアヤしいというか、そういうものです。

とりあえず、``HEX()``で見てみましょう。

まずは、EPSG:4612のポイントデータです。

```mysql
mysql> SELECT HEX(ST_GeometryFromText('POINT(35 135)', 4612));
+----------------------------------------------------+
| HEX(ST_GeometryFromText('POINT(35 135)', 4612))    |
+----------------------------------------------------+
| 0412000001010000000000000000E060400000000000804140 |
+----------------------------------------------------+
1 row in set (0.00 sec)
```

なんとなくEWKBに近いと思いますよね。先頭4バイトが多分SRIDだろう、というのも見えてきます。

つづいて、SRID無しの場合も見てみます。

```mysql
mysql> SELECT HEX(ST_GeomFromText('POINT(35 135)'));
+----------------------------------------------------+
| HEX(ST_GeomFromText('POINT(35 135)'))              |
+----------------------------------------------------+
| 00000000010100000000000000008041400000000000E06040 |
+----------------------------------------------------+
1 row in set (0.00 sec)
```

ここで、2つの相違があります。
- SRIDの4バイトが確保されています。
- XとYの座標値が入れ替わっています。

ここで、PostGISでの結果 (35,135)でなく (135,35) を見てみましょう。

```psql
db=# SELECT ST_GeometryFromText('POINT(135 35)', 4612);
                st_geometryfromtext                 
----------------------------------------------------
 0101000020041200000000000000E060400000000000804140
(1 行)
```

まとめると、mysqlの内部表現は次のようになります。

```
04120000 - SRID (必ず入る)
01 - エンディアンフラグ
01000000 - ジオメトリタイプ (ポイント)
0000000000E06040 - X値 (横)
0000000000804140 - Y値 (縦)
```

また、PostGISの内部表現 (EWKB)は次のようになります。

```
01 - エンディアンフラグ
01000020 - ジオメトリタイプ (ポイント+SRIDあり)
04120000 - SRID
0000000000E06040 - X値 (横)
0000000000804140 - Y値 (縦)
```

### なんで内部表現とWKTとでaxis orderが違うのか

(2018年7月19日更新)

「MySQLの空間拡張機能のaxis orderについて」の元記事のコメントでご紹介頂きました https://mysqlserverteam.com/axis-order-in-spatial-reference-systems/ ("Storage Format"の節)によると

> The storage format in MySQL is unchanged. The few functions that supported geographic coordinates in MySQL 5.7 stored longitude in the first coordinate and latitude in the second coordinate. MySQL 8.0 still does that. 

ということで、以前の版から変更しないためのようです。

## 機能の圧倒的な差

``ST_Transform()``だけでなく、機能面で圧倒的な差があります。

MySQLサイトの「空間関数のリファレンス」 https://dev.mysql.com/doc/refman/5.6/ja/spatial-function-reference.html と PostGIS日本語マニュアルの「PostGIS関数索引」 https://aginfo.cgk.affrc.go.jp/docs/pgisman/2.4.0/PostGIS_Special_Functions_Index.html とを見比べると一目瞭然です。

これは、でも、頑張れば差を詰められるものであることに注意して下さい。現時点ではダメだと思っても、来年の今頃はどうなっているか分かりません。

# おわりに

MySQL 8.0.3-rcの空間拡張機能を少し触ってみて、また、PostGISと若干の比較をしてみたりしました。

「機能の圧倒的な差」は、MySQLの空間拡張に本腰を入れだしたのが最近であり、当然なのですが、ちらっと書いた通り「手練れがいる」ように思いますので、今後に注目していくべきだと思います。

私の場合は、現時点では、``ST_Transform()``が弱い以上、ちょっと使えません。でも、ユーザの目的によって必要な機能が違ってきますので、空間関数のリファレンスを見ながら判断するべきだろうと思います。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

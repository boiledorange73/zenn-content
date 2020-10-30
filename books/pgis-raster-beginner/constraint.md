---
title: "PostGISラスタに追加することができる制約"
---
# はじめに

ここでは、ラスタ型に対する制約について説明します。

## 改めて警告を見直してみる

ここまでで、``gdalinfo``実行結果をコピペする際に付いてきていたけど「無視して下さい」と言い続けてきた警告を、ここで改めて見直してみます。

```
Warning 1: Cannot find (valid) information about public.dem10 table in raster_columns view. The raster table load would take a lot of time. Please, execute AddRasterConstraints PostGIS function to register this table as raster table in raster_columns view. This will save a lot of time.
```

「``AddRasterConstraints``でラスタカラムの情報を``raster_columns``に登録するとロード時間が短縮されます」と言っています。

この警告を消すための話でもあります。

# 制約

データベースでは、カラムごとに型が規定され、型に反するデータは格納できませんが、より格納できる条件を厳しくしたいことが多くあり、その際には、制約を追加します。制約を追加すると、その制約に合ったデータだけが格納されていることが保証され、制約に反するデータが存在しない前提でロジックを書くことができるようになります。

PostGISのベクタデータも、結構フリーダムで、空間参照系もジオメトリタイプも全てバラバラでも大丈夫です。が、たいていはカラムを定義する際に型を``GEOMETRY(POINT, 4326)``等と指定して、制約を付けています。

PostGISラスタもやはり、かなりフリーダムです。たとえば、各カラムが隙間無くしきつめるタイルのひとつをなすことが保証されるような仕様ではありませんし、そもそもラスタごとに別個のSRIDが設定できます。このフリーダムさを抑えるために、ラスタ型に対しても制約は必須です。

特に、1テーブル1ラスタの場合には、制約がないと困ります。行ごとにSRIDやセルサイズが異なる場合には、じゃあ結局そのテーブル=ラスタのSRIDは何なのか、また、セルサイズはどの行のセルサイズを取ればいいのか、といったことが分からなくなります。制約が無い場合には、GDALは先頭10タプルから推定するようです。

極めつけはextent制約(全行のラスタが既定された地理的範囲内に存在する)で、extent制約があればそのテーブル=ラスタの地理的範囲をextent制約から推定できますが、extent制約がなければ、全行のラスタの地理的範囲を調べて集計します。数行程度ならすぐに計算できますが、10万行を超えるような状況(5mメッシュがそんなかんじ)になってくると、地理的範囲が必要なときに毎回計算するのはかなり面倒そうです。

つまり、ラスタの制約は、ベクタデータの場合と違い、制約を制約として使うだけでなく、メタデータとしても使っています。警告に「``AddRasterConstraints``で…時間が短縮されます」と書かれていたことを思い出して下さい。ラスタの制約はパフォーマンス向上のためにも必要なのです。


# 制約を見る

``raster_columns``ビューでラスタについている制約を確認することができます。

```
raster=# select * from raster_columns;
 r_table_catalog | r_table_schema | r_table_name | r_raster_column | srid | scale_x | scale_y | blocksize_x | blocksize_y | same_alignment | regular_blocking | num_bands | pixel_types | nodata_values | out_db | extent | spatial_index
-----------------+----------------+--------------+-----------------+------+---------+---------+-------------+-------------+----------------+------------------+-----------+-------------+---------------+--------+--------+---------------
 raster          | public         | dem          | rast            |    0 |         |         |             |             | f              | f                |           |             |               |        |        | f
 raster          | public         | dem10        | rast            |    0 |         |         |             |             | f              | f                |           |             |               |        |        | t
(2 行)
```

このうち、r_table_catalog, r_table_schema, r_table_name, r_raster_column は、それぞれ、データベース名、スキーマ名、テーブル名、カラム名、を指します。以降のカラムが制約を示しています。カラムごとの意味は次の通りです。

* srid - 全行のラスタについて、同じ空間参照系IDが、制約で決められた値になる。
* scale_x, scale_y - 全行のラスタについて、ピクセル1つあたりの空間参照系の単位でのサイズが、制約で決められた値になる。
* blocksize_x, blocksize_y - 全行のラスタについて、ラスタ1つあたりの横・縦のピクセル数が、制約で決められた値になる。
* same_alignment - 全行のラスタについて、同じアラインメントになる(後述)。
* regular_blocking - 全行のラスタについて、レギュラーブロックになる(後述)。
* num_bands - 全行のラスタについて、バンド数が、制約で決められた値になる。
* pixel_types - 全行のラスタについて、ピクセルタイプが、制約で決められた値になる。
* nodata_values - 全行のラスタについて、NoData値が、制約で決められた値になる。
* out_db - 全行のラスタについて、本体がデータベース外にあるか内部に納められているか(f)外部ファイルに置かれているか(t)の選択が、制約で決められた値になる。
* extent - 全行のラスタについて、ラスタの地理的範囲が、制約で決められた地理的範囲内に収まる。

同じアラインメントになる(same_alignment)、というのは、スキュー、スケール、空間参照系が同じで、かつ、ピクセルを分けるグリッド線が（範囲外まで引くと）重なることを指します。

レギュラーブロックになる(regular_blocking)、というのは、同じアラインメントを持ち(same_alignment)、同じブロックサイズ(blocksize_x, blocksize_y)を持ち、ラスタ集合のエンベロープの左上隅からピクセル単位でblocksize_x, blocksize_y間隔でラスタが置かれていて、かつ重複タイルが無いことを指します。

# 制約を追加する

regular_blockingを除く全ての制約を追加するには、次のようにします。

```
SELECT AddRasterConstraints(<テーブル名>,<カラム名>);
```

既にデータが入っている場合には、そのデータが制約に合うかどうかをチェックし、全行について制約に合う場合に限って制約を追加します。つまり、全ての制約が追加できるとは限りません。

特定の制約を追加する場合には、次のようにします。

```
SELECT AddRasterConstraints('<テーブル名>'::name,'<カラム名>'::name, '<制約名1>', '<制約名2>', ...);
```

第1引数、第2引数で``::name``をつけていることに注意して下さい。これらを文字列型にすると、全引数が文字列型となり、文字列型はたいていのスカラ値にキャスト可能なので、既にあるバリエーションと衝突してしまうためです。

## regular_blockingは個別に追加

``AddRasterConstraints(<テーブル名>,<カラム名>);``では、regular_blocking制約は、そもそもチェックしません。制約名を文字列で指定するバリエーションを使って、別途regular_blocking制約を追加します。

```
SELECT AddRasterConstraints('<テーブル名>'::name,'<カラム名>'::name, 'regular_blocking');
```


## ためしにやってみよう

```
raster=# SELECT AddRasterConstraints('dem10', 'rast');
NOTICE:  Adding SRID constraint
NOTICE:  Adding scale-X constraint
NOTICE:  Adding scale-Y constraint
NOTICE:  Adding blocksize-X constraint
NOTICE:  Adding blocksize-Y constraint
NOTICE:  Adding alignment constraint
NOTICE:  Adding number of bands constraint
NOTICE:  Adding pixel type constraint
NOTICE:  Adding nodata value constraint
NOTICE:  Adding out-of-database constraint
NOTICE:  Adding maximum extent constraint
 addrasterconstraints
----------------------
 t
(1 行)
```

```
raster=# SELECT AddRasterConstraints('dem10'::NAME, 'rast'::NAME, 'regular_blocking');
NOTICE:  Adding coverage tile constraint required for regular blocking
NOTICE:  Adding spatially unique constraint required for regular blocking
 addrasterconstraints
----------------------
 t
(1 行)
```

```
raster=# SELECT * FROM raster_columns;
 r_table_catalog | r_table_schema | r_table_name | r_raster_column | srid |   scale_x    |    scale_y    | blocksize_x | blocksize_y | same_alignment | regular_blocking | num_bands | pixel_types | nodata_values | out_db |                                                                                           extent                                                                                           | spatial_index
-----------------+----------------+--------------+-----------------+------+--------------+---------------+-------------+-------------+----------------+------------------+-----------+-------------+---------------+--------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------
 raster          | public         | dem          | rast            |    0 |              |               |             |             | f              | f                |           |             |               |        |                                                                                                                                                                                            | f
 raster          | public         | dem10        | rast            |    0 | 0.0001111111 | -0.0001111111 |        1125 |         750 | t              | t                |         1 | {32BF}      | {-9999}       | {f}    | 010300000001000000050000000000000000A860406AF3A9AAAA2A41400000000000A8604000000000004041400000000000B0604000000000004041400000000000B060406AF3A9AAAA2A41400000000000A860406AF3A9AAAA2A4140 | t
(2 行)
```

``gdalinfo``を実行して警告が消えたか確認しましょう。

```
% gdalinfo "PG:dbname=raster table=dem10 mode=2 user=postgres password=****"
Driver: PostGISRaster/PostGIS Raster driver
Files: none associated
Size is 2250, 1500
Origin = (133.250000000000000,34.500000000000000)
Pixel Size = (0.000111111100000,-0.000111111100000)
Corner Coordinates:
Upper Left  ( 133.2500000,  34.5000000)
Lower Left  ( 133.2500000,  34.3333333)
Upper Right ( 133.5000000,  34.5000000)
Lower Right ( 133.5000000,  34.3333333)
Center      ( 133.3750000,  34.4166667)
Band 1 Block=2048x1500 Type=Float32, ColorInterp=Gray
  NoData Value=-9999
```

警告が消えました。``gdalinfo``は4個のラスタデータから直接得たデータでなく、``raster_columns``からのデータを使うようにしたことを示しています。

# 制約を外す

## ファイルが追加できない

```
% psql -d raster -f FG-GML-5133-41-dem10b-20161001.sql
BEGIN
psql:FG-GML-5133-41-dem10b-20161001.sql:4: NOTICE:  Raster is not in the coverage
psql:FG-GML-5133-41-dem10b-20161001.sql:4: ERROR:  リレーション"dem10"の新しい行は検査制約"enforce_coverage_tile_rast"に違反しています
DETAIL:  失敗した行は(5, 0100000100322E915A8A201D3FE7FB795E8A201DBF0000000000A46040960C56..., FG-GML-5133-41-dem10b-20161001.xml)を含みます
CONTEXT:  dem10のCOPY、行 1: "0100000100322E915A8A201D3FE7FB795E8A201DBF0000000000A46040960C56555535414000000000000000000000000000..."
ROLLBACK
```

ご覧の通り、ここでは ``enforce_coverage_tile_rast``に違反したため、追加できませんでした。仮にこれをクリアできても、``extent``等、他の制約に違反するだけです。

そこで、いったん制約を削除してしまいましょう。制約を削除するには``DropRasterConstraints``を使います。引数は同じです。

```
SELECT DropRasterConstraints(<テーブル名>,<カラム名>);
SELECT DropRasterConstraints('<テーブル名>'::name,'<カラム名>'::name, 'regular_blocking');
```

一度削除します。

```
raster=# SELECT DropRasterConstraints('dem10', 'rast');
NOTICE:  Dropping SRID constraint
NOTICE:  Dropping scale-X constraint
NOTICE:  Dropping scale-Y constraint
NOTICE:  Dropping blocksize-X constraint
NOTICE:  Dropping blocksize-Y constraint
NOTICE:  Dropping alignment constraint
NOTICE:  The constraint "enforce_regular_blocking_rast" does not exist.  Skipping
NOTICE:  Dropping coverage tile constraint required for regular blocking
NOTICE:  Dropping spatially unique constraint required for regular blocking
NOTICE:  Dropping number of bands constraint
NOTICE:  Dropping pixel type constraint
NOTICE:  Dropping nodata value constraint
NOTICE:  Dropping out-of-database constraint
NOTICE:  Dropping maximum extent constraint
 droprasterconstraints
-----------------------
 t
(1 行)
```

このメッセージを見ると``regular_blocking``も削除されている旨表示されていて、``DropRasterConstraints('<テーブル名>'::name,'<カラム名>'::name, 'regular_blocking')``は使わなくても構わなさそうです。``raster_columns``を見てみるのが一番です。


```
raster=# select * from raster_columns;
 r_table_catalog | r_table_schema | r_table_name | r_raster_column | srid | scale_x | scale_y | blocksize_x | blocksize_y | same_alignment | regular_blocking | num_bands | pixel_types | nodata_values | out_db | extent | spatial_index
-----------------+----------------+--------------+-----------------+------+---------+---------+-------------+-------------+----------------+------------------+-----------+-------------+---------------+--------+--------+---------------
 raster          | public         | dem          | rast            |    0 |         |         |             |             | f              | f                |           |             |               |        |        | f
 raster          | public         | dem10        | rast            |    0 |         |         |             |             | f              | f                |           |             |               |        |        | t
(2 行)
```

すべて削除されているのが確認できます。

```
% psql -U postgres -d raster -f FG-GML-5133-41-dem10b-20161001.sql
BEGIN
COPY 1
COMMIT
```

何の問題もなくラスタを追加できました。

最後に、``AddRasterConstraints``で制約を追加します。

```
SELECT AddRasterConstraints('dem10', 'rast');
SELECT AddRasterConstraints('dem10'::NAME, 'rast'::NAME, 'regular_blocking');
```

``raster_columns``を見ます。

```
 r_table_catalog | r_table_schema | r_table_name | r_raster_column | srid |   scale_x    |    scale_y    | blocksize_x | blocksize_y | same_alignment | regular_blocking | num_bands | pixel_types | nodata_values | out_db |                                                                                           extent                                                                                           | spatial_index
-----------------+----------------+--------------+-----------------+------+--------------+---------------+-------------+-------------+----------------+------------------+-----------+-------------+---------------+--------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------
 raster          | public         | dem          | rast            |    0 |              |               |             |             | f              | f                |           |             |               |        |                                                                                                                                                                                            | f
 raster          | public         | dem10        | rast            |    0 | 0.0001111111 | -0.0001111111 |        1125 |         750 | t              | t                |         1 | {32BF}      | {-9999}       | {f}    | 010300000001000000050000000000000000A460406AF3A9AAAA2A41400000000000A4604000000000004041400000000000B0604000000000004041400000000000B060406AF3A9AAAA2A41400000000000A460406AF3A9AAAA2A4140 | t
```

問題ないようです。

# おわりに

ここでは、PostGISラスタに追加することができる制約と、制約を追加する方法と、制約を削除する方法とを紹介しました。また、制約の追加、削除両方について、一括で追加削除を行う版と個別に行う版とが用意されていることも紹介しました。追加では``regular_blocking``制約は対象外となり、別途制約を追加する必要がありますが、削除では一括削除の対象となっています。

制約はメタデータとしても使用するので、できるだけ制約を追加するべきです。ただし、ラスタを追加する際に、一時的に制約を外すことが必要となることがあります。私は、カラム数（ラスタデータの個数）が多くなると、制約追加の所要時間が意外とかかるので、全てのラスタを格納してから最後に制約を追加するのがいいかな、と考えています。

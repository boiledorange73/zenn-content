---
title: "PostGISラスタに加えることができる制約"
---
# はじめに

ここでは、ラスタ型に対する制約について説明します。

## 制約

データベースでは、カラムごとに型が規定され、型に反するデータは格納できませんが、より格納できる条件を厳しくしたいことが多くあり、その際には、制約を追加します。制約を追加すると、その制約に合ったデータだけが格納されていることが保証され、制約に反するデータが存在しない前提でロジックを書くことができるようになります。

PostGISラスタは、かなりフリーダムです。仕様で書いたとおり、たとえば、各カラムが隙間無くしきつめるタイルのひとつをなすことが保証されるような仕様ではありませんし、そもそもラスタごとに別個のSRIDが設定できます。フリーダムさを抑えるために、ラスタ型に対する制約を追加することができます。

特に、1テーブル1ラスタの場合には、制約がないと困ります。行ごとにSRIDやセルサイズが異なる場合には、じゃあ結局そのテーブル=ラスタのSRIDは何なのか、また、セルサイズはどの行のセルサイズを取ればいいのか、といったことが分からなくなります。制約が無い場合には、GDALは先頭10行から推定するようです。

極めつけはextent制約(全行のラスタが既定された地理的範囲内に存在する)で、extent制約があればそのテーブル=ラスタの地理的範囲をextent制約から推定できますが、extent制約がなければ、全行のラスタの地理的範囲を調べて集計します。数行程度ならすぐに計算できますが、10万行を超えるような状況(5mメッシュがそんなかんじ)になってくると、地理的範囲が必要なときに毎回計算するのはかなり面倒そうです。

# 制約を見る

``raster_columns``ビューでラスタについている制約を確認することができます。

```psql
db=# SELECT * FROM raster_columns WHERE r_table_name='dem';
 r_table_catalog | r_table_schema | r_table_name | r_raster_column | srid | scale_x | scale_y | blocksize_x | blocksize_y | same_alignment | regular_blocking | num_bands | pixel_types | nodata_values | out_db | extent 
-----------------+----------------+--------------+-----------------+------+---------+---------+-------------+-------------+----------------+------------------+-----------+-------------+---------------+--------+--------
 db              | public         | dem          | rast            |    0 |         |         |             |             | f              | f                |           |             |               |        | 
(1 行)

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

```psql
SELECT AddRasterConstraints(<テーブル名>,<カラム名>);
```

既にデータが入っている場合には、そのデータが制約に合うかどうかをチェックし、全行について制約に合う場合に限って制約を追加します。つまり、全ての制約が追加できるとは限りません。

特定の制約を追加する場合には、次のようにします。

```psql
SELECT AddRasterConstraints('<テーブル名>'::name,'<カラム名>'::name, '<制約名1>', '<制約名2>', ...);
```

第1引数、第2引数で``::name``をつけていることに注意して下さい。これらを文字列型にすると、全引数が文字列型となり、文字列型はたいていのスカラ値にキャスト可能なので、既にあるバリエーションと衝突してしまうためです。

## regular_blockingは個別に追加

制約名を文字列で指定するバリエーションを使うと、regular_blocking制約を追加できます。

```psql
SELECT AddRasterConstraints('<テーブル名>'::name,'<カラム名>'::name, 'regular_blocking');
```

# 基盤地図情報DEMがsame_alignment制約に反する問題

[PostGISラスタにデータを格納する](https://boiledorange73.qrunch.io/entries/eH7PXYsJjPmSd9Zq)で、``AddRasterConstraints``を実行したら、same_alignmentで失敗したことを、次のようなメッセージを付けて示しました。

```psql
NOTICE:  Adding alignment constraint
CONTEXT:  PL/pgSQL function addrasterconstraints(name,name,name,boolean,boolean,boolean,boolean,boolean,boolean,boolean,boolean,boolean,boolean,boolean,boolean) line 53 at RETURN
NOTICE:  The rasters (pixel corner coordinates) are not aligned
CONTEXT:  SQL statement "ALTER TABLE public.dem5 ADD CONSTRAINT enforce_same_alignment_rast CHECK (st_samealignment(rast, '01000000000D11915A8A200D3F0D11915A8A200DBF0000000000AC60401011111111414140000000000000000000000000000000000412000001000100'::raster))"
PL/pgSQL function _add_raster_constraint(name,text) line 4 at EXECUTE statement
PL/pgSQL function _add_raster_constraint_alignment(name,name,name) line 29 at RETURN
PL/pgSQL function addrasterconstraints(name,name,name,text[]) line 82 at assignment
PL/pgSQL function addrasterconstraints(name,name,name,boolean,boolean,boolean,boolean,boolean,boolean,boolean,boolean,boolean,boolean,boolean,boolean) line 53 at RETURN
NOTICE:  Unable to add constraint: enforce_same_alignment_rast
```

基盤地図情報DEMは、ピクセルサイズは0.4秒(10mメッシュ)または0.2秒(5mメッシュ)、ファイルごとのピクセル数は全て同じで作成されています。となると、少なくともsame_alignment制約に反することはないはずです。

## セルサイズが違う

FG-GML-5133-63-dem10b-20090201.xml をのぞいてみると、地理的範囲は次のようになっています。

```xml
<gml:lowerCorner>34.5 133.375</gml:lowerCorner>
<gml:upperCorner>34.583333333 133.5 </gml:upperCorner>
```

また、ピクセル数も書いています。

```xml
<gml:low>0 0</gml:low>
<gml:high>1124 749</gml:high>
```

横1125ピクセル、縦750ピクセルです。これはどのファイルも同じです。

北隣の FG-GML-5133-73-dem10b-20090201.xml の地理的範囲は、次のようになっています。

```xml
<gml:lowerCorner>34.583333333 133.375</gml:lowerCorner>
<gml:upperCorner>34.666666667 133.5 </gml:upperCorner>
```

緯度だけに注目してみます。Googleで計算すると同じになってしまったので、とりあえずCで書いてみました。

```c
#include <stdio.h>

int main(int ac, char *av[]) {
  printf("%.14f\n", (34.583333333 - 34.5)/750.0 );
  printf("%.14f\n", (34.666666667 - 34.583333333)/750.0 );
  return 0;
}
```

実行結果は次のようになりました。

```csh
% cc a.c
% ./a.out 
0.00011111111067
0.00011111111200
```

Y方向のセルサイズの値が違っています。これだと scale_y制約に違反することになります。

## 制約違反は若干目をつぶってくれる

本当なら scale_y制約に違反することになるのですが、違反していないように扱われています。
これは、2値の差の絶対値がfloat.hにあるDBL_EPSILON以内なら等価である、として、若干大目に見ているためです。

しかし、same_alignmentは、行ごとの地理的範囲で見るので、縦なら``セルサイズ*750ピクセル``で同じグリッド線上にあるかどうかを見るので、セルサイズの差より750倍に感度が上がることになります。

このため、scale_x, scale_y制約は満たしていると判断され、same_alignment制約は違反していると判断されたようです。

## スナップさせればOK…?

とりあえず、セルサイズと左上隅位置を、何らかの値でスナップさせればOKなかんじです。

10mメッシュの場合は、1ピクセルが0.4秒なので、0.4/3600=1/9000度でスナップさせればOKです。5mメッシュの場合は0.2秒なので、1/18000度でスナップさせます。

それでも、regular_blocking制約を満たしません。

## 若干膨張させる必要がある

raster_columnsのextentを見ようとするとEWKBHEXが見えると思います。これは、``ST_Union(ST_Envelope(rast))``の結果を投入しています。extent制約は、このEWKBHEXが示すポリゴンに、ラスタが含まれなければならない、という制約です。

スナップさせると、extent制約のためのポリゴンを生成するのに多大な時間を要することがあります。

集計関数としての``ST_Union``は、PostGIS 1.5あたりからは、タプルから出た順番のままで結合を試みることをせず、近隣のジオメトリ同士を結合しようとするようにしています。

仮にタプルから出た順番のままで結合を試みようとすると、とんでもなく時間がかかります。たとえば、本州の都府県ポリゴンを結合しようとして、最悪のパターンとして、はじめに青森県、ついで山口県が出現すると、これらを「結合」した結果、結局結合できずにマルチポリゴンが生成されます。このマルチポリゴンと岩手県とは結合できますが、青森ポリゴンと山口ポリゴンとの結合を試みます。結果として青森ポリゴンと結合できますが、青森・岩手ポリゴンと山口ポリゴンからなるマルチポリゴンが生成され、常に余計な計算をすることになります。
仮に青森に近い順に結合を試みると、まず青森・岩手のシングルポリゴンが生成され、続いて、青森・岩手・秋田のシングルポリゴンが生成され、順次、シングルポリゴンとシングルポリゴンの結合を行い、山口ポリゴンは、最後に山口以外が結合された本州ポリゴンと結合する際に登場することになります。これにより、計算量が大幅に削減されます。

話をもどして、extent制約をかける際ですが、ラスタが敷き詰められたタイルのように見えても、隙間があると、近隣から結合を試みても結合でないので、近隣から結合して計算量を削減することができなくなります。

この隙間は、数ミリ程度でも何でも、存在すると、結合できません。

タイルラスタ間に隙間ができるのは、タイルの右下隅の位置を、左上隅位置+セルサイズ*セル数 の計算で出すためです。丸められて、右下隅が思ったより左上にずれます。これは仕様上、どうしようもないものですので、若干セルを膨らませなければならなくなります。

# おわりに

ここでは、PostGISラスタに加えることができる制約について説明しました。

また、same_alignment制約はセルサイズや左上隅位置をスナップさせることで解決できるものの、regular_blocking制約は満たなかった、と述べました。

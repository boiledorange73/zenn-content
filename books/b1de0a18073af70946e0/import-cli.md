---
title: "シェープファイルのデータをインポートしてみよう コマンドライン編"
---
# はじめに

シェープファイルは、GIS大手(ArcGIS等を開発)であるesri社が策定した地理空間データのフォーマットのひとつです。ESRI社製品以外でも、ほとんどのGISでインポートできるので、データ交換用に非常によく使われています。

PostGISもシェープファイルのインポートとエクスポートを行うためのツールが同梱されています。インポートについては、コマンドラインのシェープファイルからSQLに変換するshp2pgsqlと、GUIインポータが同梱されています。

ここでは、shp2pgsqlを使ってシェープファイルのデータをPostGISにインポートする方法をご紹介します。

# シェープファイルについて

シェープファイルはジオメトリデータのみを納める、バイナリのファイル形式で、シェープファイルのPolygonはOGC Simple Feature Accessで言うところのMultiPolygonに相当します。OGC SFAで言うところのPolygonはシェープファイルにはありません。同じくOGC SFAで言うところのMultiLinestringはありますがLinestringはありません。

シェープファイルは拡張子を``.shp``とするのが通常です。``.shp``ファイルは``.dbf``ファイルと``.shx``ファイルの三つセットで利用します。グループの判別はプリフィックスを見ます。

# とりあえずデータを持ってくる

実験用データとして、国土数値情報 http://nlftp.mlit.go.jp/ksj/index.html から「行政区域」をもらってきましょう。以下の例では広島県のみを引っ張ってきています。ZIPでもらえます。これをunzipすると、xmlファイル（JPGIS 1.2形式)のほかに、shp, dbf, shx 等のファイルがあります。

# shp2pgsql

``shp2pgsql``というものがPostGISに添付されてきますので、これを使います。これは、シェープファイルをPostGIS上で実行可能なSQLに変換するものです。あくまで変換であって、実際にインポートするには、``psql``フロントエンドを使う必要があります。

``shp2pgsql``は、``(PostGIS)/bin/shp2pgsql`` にあります。
スタックビルダから入れた場合には、``(PostgreSQL)\(バージョン)\bin\shp2pgsql.exe`` にあります。

# 1ファイルだけのインポート

## SQLに変換する

```csh
% shp2pgsql -W cp932 -D -I -s 4612 N03-14_34_140401.shp t1 > N03-14_34_140401.sql
```

* ``-W cp932``で、ソースの文字コードをcp932と指定
* ``-D``で、ダンプ形式にする(指定しないとINSERT形式)
* ``-I``で、空間インデックスを貼るコマンドを追加する
* ``-s 4612``で、空間参照系を4612 (JGD2000地理座標系)と指定。
* シェープファイルをずらっとならべる
* コマンド引数の最後でテーブル名を与える
* 標準出力に吐き出されるのでリダイレクト

文字コードをshift_jisでなくcp932にしているのは、かつてShift JISの拡張部分にひっかかったことがあるためです。文字コード変換にはiconvが使われていて、iconvの場合は、shift_jisと指定された場合には、純粋な SHIFT JIS コードセットのみを対象とするので、拡張部分を含むcp932を使用する場合には、cp932を指定します。

```sql
SET CLIENT_ENCODING TO UTF8;
SET STANDARD_CONFORMING_STRINGS TO ON;
BEGIN;
CREATE TABLE "t1" (gid serial,
"n03_001" varchar(10),
"n03_002" varchar(20),
"n03_003" varchar(20),
"n03_004" varchar(20),
"n03_007" varchar(5));
ALTER TABLE "t1" ADD PRIMARY KEY (gid);
SELECT AddGeometryColumn('','t1','geom','4612','MULTIPOLYGON',2);
COPY "t1" ("n03_001","n03_002","n03_003","n03_004","n03_007",geom) FROM stdin;
広島県  \N      \N      庄原市  34210   01060000200C1A00000100000001030000000100...
...
\.
CREATE INDEX ON "t1" USING GIST ("geom");
COMMIT;
```

順を追ってみていきましょう。

まず、``CREATE TABLE``でジオメトリカラム以外のカラムを持つテーブルを作成し、続けて``AddGeometryColumn``でジオメトリカラムを追加します。

以上の2文は、次に示す、型修飾を使ってテーブルを作成するのと同じです。

```sql
CREATE TABLE "t1" (
  "gid" SERIAL PRIMARY KEY,
  "n03_001" varchar(10),
  "n03_002" varchar(20),
  "n03_003" varchar(20),
  "n03_004" varchar(20),
  "n03_007" varchar(5),
  "geom" GEOMETRY(MULTOPOLYGON, 4612)
);
```

テーブル作成完了後、``COPY``コマンドで、"\\."のみになっている行までタブ区切りと認識してテーブルに入れ込んでいきます。5カラム目はEWKBHEXと呼んでいますが、EWKB形式を引数にHEXを実行した結果の文字列です。EWKBについてはここでは説明しません。

最後に```CREATE INDEX```で空間インデックスを貼っています。

## インポート

次のコマンドでOKです。

```csh
% psql -d db -f N03-14_34_140401.sql
```

## Windowsでインポートする場合の注意

Windowsのコマンドプロンプトや、それを元にしている OSGeo4W Shell などでは、CP932をデフォルトの文字コードと認識しているため、UTF8のファイルを読ませようとすると、非ASCIIがあると、エラーが発生します。

回避するには、環境変数``PGCLIENTENCODING``を設定します。

```batch
> set PGCLIENTENCODING=UTF8
> psql -d db -f N03-14_34_140401.sql
```

# テーブル作成とデータコピーを分離する

同じスキーマだけれどもシェープファイルが複数に分かれている場合があります。ここで使わせてもらっている国土数値情報（行政区域）は、都道府県別に分かれています。

シェープファイルとSQLファイルとを一対一で管理したい場合に、ファイルごとに上記のコマンドを実行した場合、ファイルごとに``CREATE TABLE``を実行しようとするため、複数のSQLファイルを実行しようとすると、失敗します。

テーブル作成用のファイルをひとつ作り、シェープファイルと対応するSQLファイルはデータコピーだけを行うようにすれば、失敗しません。

## テーブル作成用ファイルの作成

``-p``オプションを付けて、テーブル作成のみのSQLを吐き出させます。

```csh
% shp2pgsql -p -W cp932 -D -I -s 4612 N03-14_34_140401.shp t2 > tables.sql
```

``-D``でダンプ形式にするよう指示しているのですが、データを出力しないので、意味がありません。害もありません。

```sql
SET CLIENT_ENCODING TO UTF8;
SET STANDARD_CONFORMING_STRINGS TO ON;
BEGIN;
CREATE TABLE "t2" (gid serial,
"n03_001" varchar(10),
"n03_002" varchar(20),
"n03_003" varchar(20),
"n03_004" varchar(20),
"n03_007" varchar(5));
ALTER TABLE "t2" ADD PRIMARY KEY (gid);
SELECT AddGeometryColumn('','t2','geom','4612','MULTIPOLYGON',2);
CREATE INDEX ON "t2" USING GIST ("geom");
COMMIT;
```

## データコピー用ファイルの作成

``-a``オプションを付けて、ダンプ形式のデータコピーを行うSQLを吐き出させます。ファイルごとにこれを繰り返します。

```csh
% shp2pgsql -a -W cp932 -D -s 4612 N03-14_34_140401.shp t2 > N03-14_34_140401.sql
```

``-I``オプションは、ここでは消さなければなりません。ファイルの末尾に``CREATE INDEX``が追加され、tables.sqlで既に貼られているのに、二重で同名のインデックスが貼られようとするので、エラーとなります。

出力されるファイルは次のようになります。

```sql
SET CLIENT_ENCODING TO UTF8;
SET STANDARD_CONFORMING_STRINGS TO ON;
BEGIN;
COPY "t2" ("n03_001","n03_002","n03_003","n03_004","n03_007",geom) FROM stdin;
広島県  \N      \N      庄原市  34210   01060000200C1A00000100000001030000000100...
...
\.
COMMIT;
```

インポートは、tables.sqlを最初に実行させ、その後はデータコピー用ファイルを順次実行させます。

```csh
% psql -d db -f tables.sql
SET
SET
BEGIN
psql:tables.sql:9: NOTICE:  CREATE TABLE will create implicit sequence "t2_gid_seq" for serial column "t2.gid"
CREATE TABLE
psql:tables.sql:10: NOTICE:  ALTER TABLE / ADD PRIMARY KEY will create implicit index "t2_pkey" for table "t2"
ALTER TABLE
                 addgeometrycolumn                  
----------------------------------------------------
 public.t2.geom SRID:4612 TYPE:MULTIPOLYGON DIMS:2 
(1 行)

CREATE INDEX
COMMIT
% psql -d db -f N03-14_34_140401.sql 
SET
SET
BEGIN
COMMIT
% 
```

# 余談: シェープファイルについてもう少し

上で、プリフィックスが同じでサフィックスがそれぞれ``.shp``、``.dbf``、``.shx``となる3ファイルをセットで利用する旨書きました。それぞれもう少し詳しく見てみましょう。

``.shp``ファイルは、ジオメトリデータのみ保存します。ジオメトリごとにポイント数が異なるので可変長になります（POINTタイプは除く）。

``.dbf``は、属性データ（ジオメトリに対して与えられる値の集合）を保存するDBase IV形式のファイルです。こちらは固定長です。

``.shx``は、``.shp``の物理レコード番号からレコードの先頭を指すオフセットを得るためのテーブルです。``.dbf``は(ヘッダ長)+(物理レコード番号)*(レコード長)でレコードの先頭を指すオフセットを得られるので不要ですね。

``.shp``レコードと``.dbf``レコードは１対１の関係を持ちます。この関係は「物理的」で、ファイル内のレコードの順序が鍵となります。「論理的」、たとえば両方のレコードにユニークな整数を振って、その数を鍵として関係の有無を判定する、というような方法はとっていません。

当然ですが、``.shp``、``.dbf``、``.shx``の一貫性は全く保証されません。ソフトウェア側で十分に管理する必要があります。

``.shp``と``.dbf``は統合できそうに思いますが、他のソフトウェアでも読める``.dbf``の利点が活かせなくなるので、場合によりけりでしょう。``.shp``と``.shx``との統合に至っては、ファイル構造が一気に複雑になります。

あと、``.shx``を無くすという選択肢は無いことも無いですが、データ交換用でなく、アプリケーションでそのまま利用することを考えたら、全ての情報をオンメモリに納められるわけではなく、ファイルから逐次読取りが必要な場合もあるでしょうから、``.shx``は必要でしょう。

では、そろそろ21世紀に帰りましょう。

前世紀の遺物ではあるのですが、地理空間データの交換でよく使われているのは事実で、好むと好まざるとに関わらず、この事実は受け入れなければなりません。

シェープファイルの仕様についてもっと知りたい方は、esriジャパン公式サイト内 http://www.esrij.com/getting-started/learn-more/shapefile/ から、シェープファイルの技術資料が公開されていますので、そちらをご覧下さい。かつて「ITな人」が「ジオの世界」に足を踏み入れた方で、この資料見ながらシェープファイルの読み込みルーチンを書いたという方はけっこういらっしゃったのではないかと思います。

# おわりに

国土数値情報（行政区域）のシェープファイルからSQLを生成するコマンドを動かし、PostGISデータベースにインポートしてみました。

EWKBという言葉が出てきました。EWKTと何か関連があると思ったあなたは鋭い。EWKTとマップ可能なバイナリ表現です。詳細は別の機会に。

冒頭、PostGISにはshp2pgsqlとともにGUIインポータが同梱されると述べました。GUIインポータの使い方は、別のページで紹介したいと思います。


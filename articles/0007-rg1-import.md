---
title: "逆ジオコーディングのためのデータ整備 #1 インポートと正規化編"
emoji: "😀"
type: "tech"
topics: [PostGIS]
published: false
---
# はじめに

「逆ジオコーディング」とは、位置を引数に取り、その位置が属する地名を計算することをさします。地名から位置を判定する「ジオコーディング」の「逆」です。

[国土数値情報](http://nlftp.mlit.go.jp/ksj/)を使った逆ジオコーディング機能を作ってみましょう。

ただし、国土数値情報（行政区域）だけを使うと市区町村名までしか分からないので、街区符号まで整備されている[位置参照情報](http://nlftp.mlit.go.jp/isj/)を視野に入れておきます。位置参照情報は平面直角座標系の座標値が入っているので、

# だいたいのスキーマ構成

* 都道府県マスタを g_pref として持つ。都道府県コードと都道府県名からなる。
* 市区町村マスタを g_mncpl として持ち、g_prefへ都道府県コードで参照する。
* ポリゴンは g_jpr に持ち、g_jprからg_mncplへ市区町村コードで参照する。

# とりあえずデータをデータベースに格納する

## 国土数値情報（行政区域）全国データ

ZIPファイルの中には、次のものがあります。

* KS-META-N03-14_140401.xml - 地理情報クリアリングハウスのためのメタデータ
* N03-14_140401.dbf - 属性ファイル
* N03-14_140401.prj - 投影法ファイル
* N03-14_140401.sbn - 空間インデックスファイル
* N03-14_140401.sbx - 空間インデックスファイル
* N03-14_140401.shp - 地物ファイル
* N03-14_140401.shx - インデックスファイル
* N03-14_140401.xml - JPGIS形式のデータファイル

このうち次のファイルを使います。

* N03-14_140401.dbf - 属性ファイル
* N03-14_140401.shp - 地物ファイル
* N03-14_140401.shx - インデックスファイル

## インポート

shp2pgsqlを使用してSQLファイルを作り、dbに格納します。

[シェープファイルのデータをインポートしてみよう コマンドライン編]
(https://zenn.dev/boiledorange73/books/b1de0a18073af70946e0/viewer/3)を参考にして下さい。

```csh
% shp2pgsql -s 4612 -D -i -I -W cp932 N03-14_140401.shp w_mncplraw > w_mncplraw.sql
Shapefile type: Polygon
Postgis type: MULTIPOLYGON[2]
% psql db -f w_mncplraw.sql
SET
SET
BEGIN
psql:w_mncplraw.sql:9: NOTICE:  CREATE TABLE will create implicit sequence "w_mncplraw_gid_seq" for serial column "w_mncplraw.gid"
CREATE TABLE
psql:w_mncplraw.sql:10: NOTICE:  ALTER TABLE / ADD PRIMARY KEY will create implicit index "w_mncplraw_pkey" for table "w_mncplraw"
ALTER TABLE
                     addgeometrycolumn                      
------------------------------------------------------------
 public.w_mncplraw.geom SRID:4612 TYPE:MULTIPOLYGON DIMS:2 
(1 行)

CREATE INDEX
COMMIT
%
```

## 都道府県コード、市区町村コード

各都道府県、市区町村には、総務省（自治省系）告示による「公共団体コード」が設定されています。

都道府県コードは2桁で、一意になっています。

市区町村コードは5桁の数で、上2桁は都道府県を示し、下3桁は各都道府県内で一意になっています。下3桁の部分については、もう少し法則があるのですが、ここでは割愛します。

# 都道府県テーブルを作る

都道府県テーブルを作ります。

プリフィックスを"g_"としているのは、個人的なルールで、ジオメトリを持つテーブルを示すこととしているためです。

また、c_geomは、最後に各都道府県の重心を入れるためです。今回の目的とは直接関係ないですが、都道府県名だけを重ねて表示するのに便利だからです。

```psql
db=# CREATE TABLE g_pref (
  gid SERIAL PRIMARY KEY,
  pcode INT UNIQUE,
  pname TEXT,
  c_geom GEOMETRY(POINT, 4612)
);
NOTICE:  CREATE TABLE will create implicit sequence "g_pref_gid_seq" for serial column "g_pref.gid"
NOTICE:  CREATE TABLE / PRIMARY KEY will create implicit index "g_pref_pkey" for table "g_pref"
NOTICE:  CREATE TABLE / UNIQUE will create implicit index "g_pref_pcode_key" for table "g_pref"
CREATE TABLE
db=# CREATE INDEX ix_g_pref_c_geom ON g_pref USING GiST(c_geom);
CREATE INDEX
```

各フィールドの意味は、次の通りです。

* pcode - 都道府県コード、市区町村コード(n03_007)の上2桁。
* pname - 都道府県名 (n03_001)
* c_geom - 都道府県の重心(centroid)を示すポイントを入れる予定ですが今回はNULLのままにします。

```psql
db=# INSERT INTO g_pref (pcode, pname)
  SELECT DISTINCT (n03_007::INT)/1000, n03_001
  FROM w_mncplraw
  WHERE NOT n03_007 IS NULL
  ORDER BY (n03_007::INT)/1000;
INSERT 0 47
```

## 市区町村コードが振られていないポリゴン

``INSERT``で``WHERE NOT n03_007 IS NULL``節を入れているのは、市区町村コードが振られていないポリゴンがあるためです。

```psql
db=# SELECT DISTINCT n03_001, n03_002, n03_003, n03_004 FROM w_mncplraw WHERE n03_007 IS NULL;
 n03_001  | n03_002 | n03_003 |  n03_004
----------+---------+---------+-----------
 千葉県   |         |         | 所属未定地
 東京都   |         |         | 所属未定地
 愛知県   |         |         | 所属未定地
 鹿児島県 |         |         | 所属未定地
 福岡県   |         |         | 所属未定地
 沖縄県   |         |         | 所属未定地
(6 行)
```

ここでは、都道府県コードと都道府県名を抽出するのが目的ですので、この問題は無視します。

# 市区町村テーブルを作る

市区町村テーブルを作ります。

```psql
db=# CREATE TABLE g_mncpl (
  gid SERIAL PRIMARY KEY,
  mcode INT UNIQUE,
  pcode INT REFERENCES g_pref(pcode),
  bname TEXT,
  gname TEXT,
  mname TEXT,
  c_geom GEOMETRY(POINT, 4612)
);
NOTICE:  CREATE TABLE will create implicit sequence "g_mncpl_gid_seq" for serial column "g_mncpl.gid"
NOTICE:  CREATE TABLE / PRIMARY KEY will create implicit index "g_mncpl_pkey" for table "g_mncpl"
NOTICE:  CREATE TABLE / UNIQUE will create implicit index "g_mncpl_mcode_key" for table "g_mncpl"
CREATE TABLE
db=# CREATE INDEX ix_g_mncpl_c_geom ON g_mncpl USING GiST (c_geom);
CREATE INDEX
```

各フィールドの意味は、次の通りです。

* mcode - 市区町村コード
* pcode - 都道府県コード、g_prefを参照。
* bname - n03_002 北海道の振興局名
* gname - n03_003 町・村にあっては郡名、指定都市にあっては市名
* mname - n03_004 指定都市にあっては区名、それ以外（市町村および特別区）は市区町村名
* c_geom - 市区町村の重心(centroid)を示すポイントを入れる予定ですが今回はNULLのままにします。

## 所属未定地のための仮コードを付与する

こういうのは後で気づくものですが、前に持ってきています。

都道府県テーブル作成時に、市区町村コードが振られていないポリゴンがあり、これらは「所属未定地」とされています。ただし、都道府県名は存在します。

逆ジオコーディングで所属未定地に該当する場合に、どうやっても市区町村名は返せませんが、せめて都道府県名までは返したいところ。そのたに、仮の市区町村コードを作成しておきます。

```psql
db=# INSERT INTO g_mncpl(mcode, pcode, mname)
  SELECT pcode*1000, pcode, '所属未定地' FROM g_pref
  ORDER BY pcode;
INSERT 0 47
```

## 市区町村コードの重複の確認と訂正

```psql
db=# INSERT INTO g_mncpl (mcode, pcode, bname, gname, mname)
  SELECT DISTINCT n03_007::INT, n03_007::INT/1000, n03_002, n03_003, n03_004 FROM w_mncplraw
  ORDER BY n03_007::INT;
ERROR:  duplicate key value violates unique constraint "g_mncpl_mcode_key"
DETAIL:  Key (mcode)=(24543) already exists.
```

重複があったため、エラーが出ました。

ひとつの市区町村コード n03_007 に対して市区町村名を形成する n03_002, n03_003, n03_004 の組が複数あるようです。

次のクエリで重複を見てみましょう。

```psql
db=# SELECT DISTINCT Q2.n03_007, w_mncplraw.n03_002, w_mncplraw.n03_003, w_mncplraw.n03_004 FROM (
  SELECT n03_007, COUNT(*) AS cnt FROM (
    SELECT DISTINCT n03_007, n03_002, n03_003, n03_004 FROM w_mncplraw
  ) AS Q1
  GROUP BY n03_007
) AS Q2
LEFT JOIN w_mncplraw ON Q2.n03_007=w_mncplraw.n03_007
WHERE cnt > 1;
 n03_007 | n03_002 | n03_003  |   n03_004
---------+---------+----------+--------------
 24543   |         | 北牟婁郡 | 紀北町
 24543   |         | 北牟婁郡 | 紀北町入会地
(2 行)
```

「紀北町入会地」にあたるポリゴンは離島です。以前はどうも所属未定地だったようです。
ただ、合併で解消されていますので、24543は北牟婁郡紀北町とします。

```psql
db=# UPDATE w_mncplraw SET n03_004='紀北町' WHERE n03_007='24543';
UPDATE 526
```

確認してみましょう。

```psql
db=# SELECT DISTINCT Q2.n03_007, w_mncplraw.n03_002, w_mncplraw.n03_003, w_mncplraw.n03_004 FROM (
  SELECT n03_007, COUNT(*) AS cnt FROM (
    SELECT DISTINCT n03_007, n03_002, n03_003, n03_004 FROM w_mncplraw
  ) AS Q1
  GROUP BY n03_007
) AS Q2
LEFT JOIN w_mncplraw ON Q2.n03_007=w_mncplraw.n03_007
WHERE cnt > 1;
 n03_007 | n03_002 | n03_003 | n03_004
---------+---------+---------+---------
(0 行)
```

これで重複の問題がなくなったので、市区町村テーブルにデータを挿入します。

```psql
db=# INSERT INTO g_mncpl (mcode, pcode, bname, gname, mname)
  SELECT DISTINCT n03_007::INT, n03_007::INT/1000, n03_002, n03_003, n03_004 FROM w_mncplraw
  ORDER BY n03_007::INT;
INSERT 0 1901
```

# ポリゴンテーブルを作る

```psql
db=# CREATE TABLE g_jpr (
  gid SERIAL PRIMARY KEY,
  mcode INT REFERENCES g_mncpl (mcode),
  syscode INT,
  geom GEOMETRY(POLYGON, 4612)
);
NOTICE:  CREATE TABLE will create implicit sequence "g_jpr_gid_seq" for serial column "g_jpr.gid"
NOTICE:  CREATE TABLE / PRIMARY KEY will create implicit index "g_jpr_pkey" for table "g_jpr"
CREATE TABLE

db=# CREATE INDEX ix_g_jpr_geom ON g_jpr USING GiST (geom);
CREATE INDEX
```

mcodeはg_mncplを参照します。syscodeには平面直角座標系の系番号を入れる予定です。現時点ではNULLにしておきます。

g_jprにポリゴンを挿入します。
w_mncplrawはマルチポリゴンなのでポリゴンにバラして挿入します。
また、所属未定地(w_mncplraw.mcodeがNULL)に対して仮コード(県コード*1000)を割り当てます。

```psql
db=# INSERT INTO g_jpr (mcode, geom)
  SELECT
    CASE 
      WHEN NOT n03_007 IS NULL THEN n03_007::INT
      ELSE (
        SELECT pcode*1000 FROM g_pref WHERE w_mncplraw.n03_001 = g_pref.pname LIMIT 1
      )
    END AS mcode,
    (ST_Dump(geom)).geom
  FROM w_mncplraw ORDER BY n03_007::INT;
INSERT 0 73274
```

# 逆ジオコーディングのSQLを投げる

東経133.36度、北緯34.48度で示される位置が属する市区町村を検索してみましょう。

```psql
db=# SELECT mcode FROM g_jpr WHERE ST_Within('SRID=4612;POINT(133.36 34.48)'::GEOMETRY, geom);
 mcode 
-------
 34207
(1 行)
```

市区町村コードが34207である市区町村に属していることが分かります。

ついでに、市区町村名をあわせて取得してみましょう。

```psql
db=# SELECT g_jpr.mcode, bname, gname, mname FROM g_jpr
  INNER JOIN g_mncpl ON g_jpr.mcode=g_mncpl.mcode
  WHERE ST_Within('SRID=4612;POINT(133.36 34.48)'::GEOMETRY, g_jpr.geom);
 mcode | bname | gname | mname  
-------+-------+-------+--------
 34207 |       |       | 福山市
(1 行)
```

福山市であることが分かります。さらに県名も取ってみましょう。

```psql
db=# SELECT g_jpr.mcode, pname, bname, gname, mname FROM g_jpr
  INNER JOIN g_mncpl ON g_jpr.mcode=g_mncpl.mcode
  INNER JOIN g_pref ON g_mncpl.pcode=g_pref.pcode
  WHERE ST_Within('SRID=4612;POINT(133.36 34.48)'::GEOMETRY, g_jpr.geom);
 mcode | pname  | bname | gname | mname  
-------+--------+-------+-------+--------
 34207 | 広島県 |       |       | 福山市
(1 行)
```

# おわりに

国土数値情報（行政区域）をインポートして、都道府県テーブル、市区町村テーブル、ポリゴンテーブルに分け、RDBっぽいかんじのことができました。また、逆ジオコーディングのクエリも試してみました。

RDBMSに慣れている方から見ると、WHERE節で幾何関数が使われている以外はあまり気にならないと思います。

次回は、平面直角座標系の情報をg_jprに入れてみます。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

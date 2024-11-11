---
title: "福山駅から半径1km圏内の人口を5次メッシュ (250mメッシュ)から計算する"
---
# はじめに

人口データなどで、地域メッシュコードを用いたGISデータとして提供されている場合があります。

このデータで必要な範囲の人口総数を求める場合、メッシュごとの人口を足し算していくだけで済めばいいのですが、それですまない場合があります。

たとえば円形の範囲内の人口を積算しようとすると、円がメッシュを2分割してしまい、円の外側は積算から外したいところです。円ならまだましでして、小学校区や字レベルで集計したい場合には、もっと状況が悪くなります。

円でメッシュを分割する場合には、そのメッシュの人口も、面積で按分した人口に減らします。

ここでは面積按分を、特定の円(福山駅から半径1km以内)を使って実際にやってみます。

# 人口データのダウンロードとインポート

## eStatから人口メッシュデータを取得

https://www.e-stat.go.jp/gis/statmap-search?page=1&type=1&toukeiCode=00200521

あたりから「5次メッシュ (250m)」をエクスパンドして、「人口及び世帯 (JGD2011)」を選択しますが、その前に「定義書」を落としておきましょう。

定義書の URL は
<https://www.e-stat.go.jp/gis/statmap-search/data?datatype=1&statsId=T001142&downloadType=1>
です。

それから「人口及び世帯 (JGD2011)」をクリックして、M5132 (広島市を含む瀬戸内海沿岸), M5133 (福山市を含む瀬戸内海沿岸) をダウンロードします。

下のURLは、``M5132``と``M5133``が出てるページです。
<https://www.e-stat.go.jp/gis/statmap-search?page=3&type=1&toukeiCode=00200521&toukeiYear=2020&aggregateUnit=Q&serveyId=Q002005112020&statsId=T001142&datum=2011>
(変更されている場合がありますのでご了承ください)

## テーブルの作成

CSVには見出しがついている可能性があるので、とりあえずダウンロードしたものを見てみましょう。

2行目に解説っぽいのがあります。

```
,,,,　人口（総数）,　人口（総数）　男,　人口（総数）　女,　０～１４歳人口　総数,　０～１４歳人口　男,　０～１４歳人口　女,　１５歳以上人口　総数,　１５歳以上人口　男,　１５歳以上人口　女,　１５～６４歳人口　総数,　１５～６４歳人口　男,　１５～６４歳人口　女,　１８歳以上人口　総数,　１８歳以上人口　男,　１８歳以上人口　女,　２０歳以上人口　総数,　２０歳以上人口　男,　２０歳以上人口　女,　６５歳以上人口　総数,　６５歳以上人口　男,　６５歳以上人口　女,　７５歳以上人口　総数,　７５歳以上人口　男,　７５歳以上人口　女,　８５歳以上人口　総数,　８５歳以上人口　男,　８５歳以上人口　女,　９５歳以上人口　総数,　９５歳以上人口　男,　９５歳以上人口　女,　外国人人口　総数,　外国人人口　男,　外国人人口　女,　世帯総数,　一般世帯数,　１人世帯数　一般世帯数,　２人世帯数　一般世帯数,　３人世帯数　一般世帯数,　４人世帯数　一般世帯数,　５人世帯数　一般世帯数,　６人世帯数　一般世帯数,　７人以上世帯数　一般世帯数,　親族のみの世帯数　一般世帯数,　核家族世帯数　一般世帯数,　核家族以外の世帯数　一般世帯数,　６歳未満世帯員のいる世帯数　一般世帯数,　６５歳以上世帯員のいる世帯数　一般世帯数,　世帯主の年齢が２０～２９歳の１人世帯数　一般世帯数,　高齢単身世帯数　一般世帯数,　高齢夫婦世帯数　一般世帯数
```

うーん、なんじゃこれは。とりあえず仕様通りのカラム名でテーブルを作ります。

```
CREATE TABLE ppl2020raw (
  KEY_CODE TEXT,
  HTKSYORI TEXT,
  HTKSAKI TEXT,
  GASSAN TEXT,
  T001142001 TEXT,
  T001142002 TEXT,
  T001142003 TEXT,
  T001142004 TEXT,
  T001142005 TEXT,
  T001142006 TEXT,
  T001142007 TEXT,
  T001142008 TEXT,
  T001142009 TEXT,
  T001142010 TEXT,
  T001142011 TEXT,
  T001142012 TEXT,
  T001142013 TEXT,
  T001142014 TEXT,
  T001142015 TEXT,
  T001142016 TEXT,
  T001142017 TEXT,
  T001142018 TEXT,
  T001142019 TEXT,
  T001142020 TEXT,
  T001142021 TEXT,
  T001142022 TEXT,
  T001142023 TEXT,
  T001142024 TEXT,
  T001142025 TEXT,
  T001142026 TEXT,
  T001142027 TEXT,
  T001142028 TEXT,
  T001142029 TEXT,
  T001142030 TEXT,
  T001142031 TEXT,
  T001142032 TEXT,
  T001142033 TEXT,
  T001142034 TEXT,
  T001142035 TEXT,
  T001142036 TEXT,
  T001142037 TEXT,
  T001142038 TEXT,
  T001142039 TEXT,
  T001142040 TEXT,
  T001142041 TEXT,
  T001142042 TEXT,
  T001142043 TEXT,
  T001142044 TEXT,
  T001142045 TEXT,
  T001142046 TEXT,
  T001142047 TEXT,
  T001142048 TEXT,
  T001142049 TEXT,
  T001142050 TEXT
);
CREATE INDEX ON ppl2020raw (KEY_CODE);
```

うーん、なんじゃこれは…。

## UNIQUE制約、NOT NULL制約ともにない

なんじゃこりゃと思いますよね。``KEY_CODE``に主キーを設定していませんし、``T``で始まる名前を持つカラムは人口データのはずなのですが、``TEXT``型にしています。

これは、CSVファイルの1行目が見出しで、2行目が説明となっていて、``KEY_CODE``にあたるカラムが空になっているため、``NULL``と認定されて、NOT NULL 制約がかかりませんし、複数ファイルをインポートしようとすると、全てのファイルの2行目の``KEY_CODE``が``NULL``になるため、UNIQUE 制約がかかりません。じゃあインポート時に先頭から2行を読まないようにすればいいのですが、``COPY``は、1行までしか飛ばせません。

それと、人口データで欠測値は ``*`` とされているので、``INT``型にはできません。

## インポートする

インポートは、次のいずれかです。

```
\COPY <tablename> FROM '<full path>' DELIMITER ',' CSV HEADER;
```

```
COPY <tablename> FROM '<full path>' DELIMITER ',' CSV HEADER;
```

``\`` が付いているのが、``psql``クライアントで実行するもので、クライアントプロセスが読める位置にファイルが置かれている必要があります。

``\`` が付いてない方は、サーバプロセスが実行するもので、サーバプロセスが読める位置にファイルが置かれている必要があります。

それぞれの例を挙げます。

```
\COPY ppl2020raw FROM 'C:/Users/foobar/tblT001142Q5132.txt' DELIMITER ',' CSV HEADER;
```

```
COPY ppl2020raw FROM 'C:/temp/tblT001142Q5133.txt' DELIMITER ',' CSV HEADER;
```

# メッシュ地物データの生成

https://github.com/boiledorange73/pg_jpgrid にある PostGIS関数を使います。jpgridcode2box.sql を、``db``上で実行すると、関数が入ります。

# テーブルの生成

``ppl2020raw`` からテーブルを生成します。

  * T001142001 だけを使いたくて、他のカラムは不要
  * CSV の欠測値として "*" 文字が入っている可能性があって、INTへのキャストでエラーが出る可能性がある (空白にして欲しいところ) ので避ける
  * ``KEY_CODE`` が ``NULL`` になっている場合がある、というか1ファイル1タプル``KEY_CODE``が``NULL``になるので避ける
  * ``KEY_CODE`` (1/4メッシュのコード) はあるが地物データがないので ``jpgridcode2polygon()`` (上述) を使う

テーブルを作って、インデックスを貼ります。

```
CREATE TABLE ppl2020 (
  key_code TEXT PRIMARY KEY,
  population INT,
  geom GEOMETRY(POLYGON, 4326)
);
CREATE INDEX ON ppl2020 USING GiST(geom);
```

``INSERT``でタプルを追加します。

```
INSERT INTO ppl2020(key_code, population, geom)
SELECT key_code, T001142001::INT, jpgridcode2polygon(key_code)
FROM ppl2020raw
WHERE key_code IS NOT NULL AND (T001142001 ~ E'\\s*^[0-9]+\\s*$');
```

これで多分OKです。

![人口分布の図](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/apportioned/01-population.png)

# 福山駅から半径1km以内の人口を計算する

## 福山駅から半径1kmの円を生成する

円の生成には [ST_Buffer](https://postgis.net/docs/ja/ST_Buffer.html) を使います。

引数にポイントを与えると、疑似円を生成してくれます。

ここでジオグラフィを指定していますが、こうすることで、PostGIS は、メートル単位の円を生成してくれます。[PostGIS入門](../../caea8d4c77dbba2e23a0) / [ジオグラフィを引数に取ることができる関数](../../caea8d4c77dbba2e23a0/viewer/geogfns) を参照してみて下さい。

```
db=# SELECT ST_Buffer('SRID=4326;POINT(133.36169 34.489)'::GEOGRAPHY, 1000)::GEOMETRY;
0103000020E61000000100000021000000F4CF9B26ECAB6040FD4FF3519C3E4140EFA391B7EAAB604072F5EE99623E4140600945E9E5AB6040A077FBEA2A3E41405BB809EBDDAB6040006BDD68F73D414071E88A0BD3AB6040645A3F0ECA3D4140B014C4B5C5AB6040E7E93B99A43D414031F2E36CB6AB6040EB6C3C7A883D4140512642C7A5AB6040B3F4D3C5763D41408247996894AB60409777212A703D4140BCCBBDFB82AB6040080B21E8743D4140E0780F2C72AB6040B2982DD1843D4140BD88E59E62AB60407D6ECB489F3D41405FE435ED54AB6040B816AA4AC33D41403F9FB59D49AB6040809EA274EF3D41405D47AC1F41AB604076625014223E4140A9DCACC63BAB6040441FBF37593E4140408F5EC739AB60406FB48AC0923E4140AC1075353BAB60403DEDB478CC3E414067B2EC0240AB60401FAD6428043F4140A628920048AB60407AB6B8AB373F41403B0CD1DF52AB6040314DD707653F41406C9DB73560AB6040CF426B7E8A3F41401A52127F6FAB60403B4ACE9EA63F4140DDDB762580AB604072313754B83F414073FE0C8591AB6040F5F75FF0BE3F4140D124DCF2A2AB604041863A32BA3F414084705FC3B3AB60408AB67148AA3F41406C3B1F51C3AB6040FFE39DCF8F3F41401ECC0E03D1AB6040ABB53ECC6B3F4140CD536F52DCAB6040BFA8B5A03F3F41408019FECFE4AB60405D5AA4FF0C3F414052C63A28EAAB6040912E35DBD53E4140F4CF9B26ECAB6040FD4FF3519C3E4140
```

見てみましょう。

![円の図](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/apportioned/02-circle.png)

## ビューを作成する

```
CREATE OR REPLACE VIEW v_ppl2020_1 AS
SELECT
  key_code,
  population,
  ppl2020.geom,
  ST_Intersection(buff.geom, ppl2020.geom)::GEOMETRY(POLYGON, 4326) AS intersection,
  ST_Area(ST_Intersection(buff.geom, ppl2020.geom)::GEOGRAPHY) AS area,
  population * ST_Area(ST_Intersection(buff.geom, ppl2020.geom)::GEOGRAPHY) / ST_Area(ppl2020.geom::GEOGRAPHY) AS apportioned
FROM ppl2020,
  (SELECT ST_Buffer('SRID=4326;POINT(133.36169 34.489)'::GEOGRAPHY, 1000)::GEOMETRY AS geom) AS buff
WHERE ST_Intersects(buff.geom, ppl2020.geom);
```

* ``FROM`` 対象は、``ppl2020`` と ``buff`` (福山駅を中心とした半径1kmの円) です。
* ``WHERE`` で ``ppl2020.geom`` と ``buff.geom`` とがインタセクトしている場合のみを抽出しています。
* ``ppl2020.geom``カラムは、グリッドを保存しています。
* ``population``カラムは、グリッドの人口値を保存しています。
* ``intersection``カラムは、``ppl2020.geom`` と ``buff.geom`` とのインタセクトしている部分の地物を計算しています。
* ``area``カラムは、インタセクトしている部分の面積を計算しています。インタセクトしている部分 (というかこのビュー全体に言えます) の座標系は地理座標系ですので、ジオグラフィ型にキャストして ``ST_Area()`` に渡しています ([ジオグラフィ型というものがある](../../caea8d4c77dbba2e23a0/viewer/geog) 参照)。
  * ``apportioned`` カラムで、按分された人口を計算しています。``メッシュ人口*インタセクト部分面積/メッシュ面積`` です。面積はジオグラフィ型にキャストして``ST_Area()``に渡しています。

``geom`` カラムは、円とインタセクトしたグリッドを四角形のまま保存しています。これを図示すると次の通りです。

![円とインタセクトしているグリッドの図](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/apportioned/03-intersected-grids.png)

``geom``に対応する人口は``population``ですが、``population``で積算すると、円の外側の人口も積算することになります。

``intersection``カラムは、円とグリッドのインタセクトした部分だけを保存しています。これを図示すると次の通りです。

![円とのインタセクトした部分の図](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-cookbook/apportioned/04-intersection.png)

``intersection``に対応する人口は``apportioned``カラムにあります。

## 人口の集計をしてみよう

```
db=# SELECT key_code, population, apportioned FROM v_ppl2020_1 ORDER BY key_code;
  key_code  | population |     apportioned
------------+------------+---------------------
 5133527832 |        441 | 0.04144751849257032
 5133527834 |        472 |   256.4159738505175
 5133527841 |        537 |  155.04904367729569
 5133527842 |        596 |  339.65023978474903
 5133527843 |        453 |                 453
 5133527844 |        487 |                 487
 5133527931 |        529 |  249.42016033353352
 5133527932 |        710 |   57.05315031088865
 5133527933 |        625 |                 625
 5133527934 |        290 |   259.8901918137173
 5133527943 |        440 |   56.78443964696688
 5133528811 |        408 |    92.5806078407965
 5133528812 |        551 |   549.9266775311804
 5133528813 |         66 |  38.446804032363325
 5133528814 |        294 |                 294
 5133528821 |        726 |                 726
 5133528822 |        456 |                 456
 5133528823 |        393 |                 393
 5133528824 |        512 |                 512
 5133528831 |        419 |   301.4390777079539
 5133528832 |        310 |                 310
 5133528833 |        507 |  337.63755436793036
 5133528834 |        522 |                 522
 5133528841 |        422 |                 422
 5133528842 |        424 |                 424
 5133528843 |        267 |                 267
 5133528844 |         19 |                  19
 5133528911 |        490 |                 490
 5133528912 |        354 |                 354
 5133528913 |        234 |                 234
 5133528914 |        385 |                 385
 5133528921 |        296 |  208.66994368305174
 5133528923 |        256 |  254.29133777254478
 5133528924 |        348 |   24.80067304515938
 5133528931 |        141 |                 141
 5133528932 |        784 |                 784
 5133528933 |        579 |                 579
 5133528934 |        593 |                 593
 5133528941 |        477 |                 477
 5133528942 |        369 |   74.51520606697927
 5133528943 |        323 |                 323
 5133528944 |        464 |   68.58743620686126
 5133529811 |        620 |  255.72308916289518
 5133529812 |        440 |                 440
 5133529813 |        578 |   23.48853989873539
 5133529814 |        644 |   552.0770594834347
 5133529821 |        377 |                 377
 5133529822 |        264 |                 264
 5133529823 |        275 |                 275
 5133529824 |         97 |                  97
 5133529832 |        576 |    71.6758658794673
 5133529841 |        560 |  408.82828724815045
 5133529842 |        438 |   434.0541094351968
 5133529844 |        669 |  11.684243449259029
 5133529911 |        640 |                 640
 5133529912 |        743 |                 743
 5133529913 |        457 |                 457
 5133529914 |        477 |   476.9817164207836
 5133529921 |        575 |   510.5078785828336
 5133529922 |        191 |  0.9921363226071495
 5133529923 |        451 |  171.61471967305152
 5133529931 |        278 |  253.45527266492627
 5133529932 |        392 |  162.83787260136924
 5133529933 |        574 |  0.6117738398178745
(64 rows)
```

人口の合計は次の通りです。

```
db=# SELECT SUM(apportioned) FROM v_ppl2020_1;
        sum
--------------------
 20222.732529853507
```

# おわりに

ここでは、eStat人口の地域メッシュコードの入ったデータを落としてきて、地物を生成する関数を使って GISデータを作り、かつ面積按分した、半径1km圏内人口を積算してみました。

集計の際には、ジオグラフィ型を複数回使いました。軽い処理の場合には、平気でこういうことをします。やはり楽ですね。

ここでは、円の中心位置と半径は固定ですが、手法で位置や半径を変えて、任意の範囲での人口集計が可能になります。応用すると、駅ごとの近隣人口を集計して見たり、いろいろなことに使えそうですね (てきとう)。

# 参照

* [PostGIS入門](../../caea8d4c77dbba2e23a0) / [ジオグラフィを引数に取ることができる関数](../../caea8d4c77dbba2e23a0/viewer/geogfns)

# 出典

本章の作成にあたり、次のデータを使用しました。

* eStat 人口及び世帯 （JGD2011） <https://www.e-stat.go.jp/gis/statmap-search?page=1&type=1&toukeiCode=00200521> (2024年6月29日取得)

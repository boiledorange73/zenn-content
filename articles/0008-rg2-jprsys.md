---
title: "逆ジオコーディングのためのデータ整備 #2 平面直角座標系の割り当て編"
emoji: "😀"
type: "tech"
topics: [PostGIS]
published: true
---
# はじめに

今回は、ポリゴンごとに平面直角座標系の系番号を入れていきます。

# syscodeを入れていく

## I系

I系は、長崎県と鹿児島県西部離島とが対象です。

長崎県については、pcode=42で片がつきます。

```
UPDATE g_jpr SET syscode=1
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND g_mncpl.pcode=42;


```

「鹿児島県のうち、北方北緯32度南方北緯27度西方東経128度18分東方東経130度を境界線とする区域内（奄美群島は東経130度13分までを含む。)にあるすべての島、小島、環礁及び岩礁」となっています。

「北方北緯32度南方北緯27度西方東経128度18分東方東経130度を境界線とする区域」は``POLYGON((128.3 27, 130 27, 130 32, 128.3 32, 128.3 27))``となります。これをQGISに描画すると次のようになります。

![I系領域の一部](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/1.png)

鹿児島県喜界町の島がオーバラップしてしまっています。これが「（奄美群島は東経130度13分までを含む。)」をつけている理由のようです。

``POLYGON((128.3 27, 130.22 27, 130.22 29, 128.3 29, 128.3 27))``を表示してみます。

![奄美群島の領域](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/2.png)

喜界町の島がすっぽり入りました。

これをひとつのポリゴンで表してみましょう。

```
SELECT ST_AsText(
  ST_Union(
    'POLYGON((128.3 27, 130 27, 130 32, 128.3 32, 128.3 27))'::GEOMETRY,
    'POLYGON((128.3 27, 130.22 27, 130.22 29, 128.3 29, 128.3 27))'::GEOMETRY
  )
);

                                       st_astext                                       
---------------------------------------------------------------------------------------
 POLYGON((130 27,128.3 27,128.3 29,128.3 32,130 32,130 29,130.22 29,130.22 27,130 27))
(1 行)
```

QGISで表示すると次のようになります。

![I系の領域](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/3.png)

これをI系領域とします。

行政区域ポリゴンのうち、鹿児島県であって、I系領域の内側にあるものをI系とする``UPDATE``を実行します。

```
UPDATE g_jpr SET syscode=1
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND g_mncpl.pcode=46
    AND ST_Covers('SRID=4612;POLYGON((130 27,128.3 27,128.3 29,128.3 32,130 32,130 29,130.22 29,130.22 27,130 27))'::GEOMETRY,geom);
```

## II系
II系は、福岡、佐賀、熊本、大分、宮崎と鹿児島県のうちI系に含まれるもの以外、です。

まずは、鹿児島以外をアップデートします。

```
UPDATE g_jpr SET syscode=2
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND (
    g_mncpl.pcode=40
    OR g_mncpl.pcode=41
    OR g_mncpl.pcode=43
    OR g_mncpl.pcode=44
    OR g_mncpl.pcode=45
  );
```

鹿児島県についてはI系と指定されたポリゴンを除いたものとします。これの判定のために``syscode IS NULL``をWHERE節に入れます。

```
UPDATE g_jpr SET syscode=2
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode
    AND g_mncpl.pcode=46
    AND g_jpr.syscode IS NULL;
```

## III系からX系 (IX系東京を除く)

このへんは、府県内で系をまたぐことがないので、非常に楽に進められます。

```
-- 3 山口 島根 広島
UPDATE g_jpr SET syscode=3
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND (
    g_mncpl.pcode=32
    OR g_mncpl.pcode=34
    OR g_mncpl.pcode=35
  );
-- 4 香川 愛媛 徳島 高知
UPDATE g_jpr SET syscode=4
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND (
    g_mncpl.pcode=36
    OR g_mncpl.pcode=37
    OR g_mncpl.pcode=38
    OR g_mncpl.pcode=39
  );
-- 5 兵庫県 鳥取県 岡山県
UPDATE g_jpr SET syscode=5
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND (
    g_mncpl.pcode=28
    OR g_mncpl.pcode=31
    OR g_mncpl.pcode=33
  );
-- 6 京都府 大阪府 福井県 滋賀県 三重県 奈良県 和歌山県
UPDATE g_jpr SET syscode=6
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND (
    g_mncpl.pcode=18
    OR g_mncpl.pcode=24
    OR g_mncpl.pcode=25
    OR g_mncpl.pcode=26
    OR g_mncpl.pcode=27
    OR g_mncpl.pcode=29
    OR g_mncpl.pcode=30
  );
-- 7 石川県 富山県 岐阜県 愛知県
UPDATE g_jpr SET syscode=7
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND (
    g_mncpl.pcode=16
    OR g_mncpl.pcode=17
    OR g_mncpl.pcode=21
    OR g_mncpl.pcode=23
  );
-- 8 新潟県 長野県 山梨県 静岡県
UPDATE g_jpr SET syscode=8
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND (
    g_mncpl.pcode=15
    OR g_mncpl.pcode=20
    OR g_mncpl.pcode=19
    OR g_mncpl.pcode=22
  );
-- 9 福島県 栃木県 茨城県 埼玉県 千葉県 群馬県 神奈川県 (東京都を除く)
UPDATE g_jpr SET syscode=9
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND (
    g_mncpl.pcode=7
    OR g_mncpl.pcode=8
    OR g_mncpl.pcode=9
    OR g_mncpl.pcode=10
    OR g_mncpl.pcode=11
    OR g_mncpl.pcode=12
    OR g_mncpl.pcode=14
  );
-- 10 青森県 秋田県 山形県 岩手県 宮城県
UPDATE g_jpr SET syscode=10
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND (
    g_mncpl.pcode=2
    OR g_mncpl.pcode=3
    OR g_mncpl.pcode=4
    OR g_mncpl.pcode=5
    OR g_mncpl.pcode=6
  );
```

## XI系

北海道は、みっつの系に分かれます。

まず、東部を実行します。

```
-- 11 小樽市　函館市　伊達市　北斗市　豊浦町　壮瞥町　洞爺湖町
-- 　北海道後志総合振興局の所管区域
-- 　北海道渡島総合振興局の所管地域
-- 　北海道檜山振興局の所管区域
UPDATE g_jpr SET syscode=11
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND (
    g_mncpl.mcode=1203
    OR g_mncpl.mcode=1202
    OR g_mncpl.mcode=1233
    OR g_mncpl.mcode=1236
    OR g_mncpl.mcode=1571
    OR g_mncpl.mcode=1575
    OR g_mncpl.mcode=1584
    OR g_mncpl.bname='後志総合振興局'
    OR g_mncpl.bname='渡島総合振興局'
    OR g_mncpl.bname='檜山振興局'
  );
```

## XIII系

続いて、XII系をとばして、西部のXIII系に行きます。

```
-- 13 北見市　帯広市　釧路市　網走市　根室市　美幌町　津別町　斜里町　清里町
-- 　小清水町　訓子府町　置戸町　佐呂間町　大空町
-- 　北海道十勝総合振興局の所管区域　北海道釧路総合振興局の所管区域　北海道根室振興局の所管区域
UPDATE g_jpr SET syscode=13
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND (
    g_mncpl.mcode=1208
    OR g_mncpl.mcode=1207
    OR g_mncpl.mcode=1206
    OR g_mncpl.mcode=1211
    OR g_mncpl.mcode=1223
    OR g_mncpl.mcode=1543
    OR g_mncpl.mcode=1544
    OR g_mncpl.mcode=1545
    OR g_mncpl.mcode=1546
    OR g_mncpl.mcode=1547
    OR g_mncpl.mcode=1549
    OR g_mncpl.mcode=1550
    OR g_mncpl.mcode=1552
    OR g_mncpl.mcode=1564
    OR g_mncpl.bname='十勝総合振興局'
    OR g_mncpl.bname='釧路総合振興局'
    OR g_mncpl.bname='根室振興局'
  );
```

## XII系の前に

XII系は、北海道のうちXI系でもXIII系でもない地域が対象です。

XII系を指定する際には、後述しますが、都道府県コードが1で、かつ、syscodeがNULLであるレコードのsyscodeを12にするようにします。

XI系またはXIII系を指定する際の``WHERE``節に間違いがあって、本来はXI系またはXIII系に含まれるべきなのにsyscodeがNULLのままであるのが残っている場合に、XII系を指定するクエリを実行すると、本来は11または13が入るべきところに12が入ってしまうかも知れません。

ここで必ず、QGISなどで確認して下さい。

### QGISでsyscodeごとに色分けして見てみよう

[QGISでPostGISのデータを見てみよう](https://zenn.dev/boiledorange73/books/b1de0a18073af70946e0/viewer/qgis)や[WMSで地域を絞り込んだうえでPostGISレイヤを表示する](https://zenn.dev/boiledorange73/books/b1de0a18073af70946e0/viewer/qgis-wms)を参考に、北海道を表示し、g_jprレイヤを作成します。

ここで、レイヤペインのg_jprレイヤを右クリックでおさえてコンテキストメニューを出し、「プロパティ」を選択します。

![g_jprのコンテキストメニューを出しているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/4.png)

「レイヤプロパティ」ダイアログが表示されるので、左側のタブを「スタイル」にして、左上隅の「共通シンボル」となっているところを「分類された」に変更します。

![「分類された」を選択しようとしているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/5.png)

新たに現れたドロップボックスで色分け表示対象とするカラムを指定します。今回は``syscode``を選択します。

![syscodeを選択しようとしているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/6.png)

何も考えずに画面中央から左下寄りにある「分類」ボタンをクリックします。

![「分類」ボタンを示した画像](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/7.png)

画面中央に``syscode``の値と色との組み合わせが表示されます。

![値と色の組み合わせ一覧を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/8.png)

「OK」をクリックすると、色分けされた地図になります。

![色分けされた地図を表示しているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/9.png)

この例では、syscodeを全部指定した後のものですので、1から17に分類されますが、現時点では1から11と13しか値がなく、12個の組み合わせしか存在しません。

新たに指定するたびに、このタブを開き「全削除」「分類」を順にクリックすると、新たなsyscodeも含む色一覧が作成されます。

![「全削除」と「分類」ボタンを示した画像](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/10.png)

## XII系

XI系とXIII系の確認が終わったら、北海道の最後となる、XII系を指定します。

```
UPDATE g_jpr SET syscode=12
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode
    AND g_mncpl.pcode=1
    AND syscode IS NULL;
```

## XIV系

XIV系は東京都のうち「北緯28度から南であり、かつ東経140度30分から東であり東経143度から西である区域」です。
南端は書いていません。計算して南端を出してもいいといえばいいのですが、わが国の領土は明らかに北半球にしかないので、赤道上まで南端を伸ばしてしまった方が、考える必要が無いので楽です。

領域ポリゴンは``POLYGON((140.5 0, 143 0, 143 28, 140.5 28, 140.5 0))``となります。

![XIV系](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/11.png)

```
-- 14 東京都 ＊北緯28度から南であり、かつ東経140度30分から東であり東経143度から西である区域
UPDATE g_jpr SET syscode=14
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND g_mncpl.pcode=13
    AND ST_Within(g_jpr.geom, 'SRID=4612;POLYGON((140.5 0, 143 0, 143 28, 140.5 28, 140.5 0))'::GEOMETRY);
```

## XV系

XV系は、沖縄県のうち「東経126度から東であり、かつ東経130度から西である区域」です。

今度は北端もありません。北端は90度にして、``POLYGON((126 0, 130 0, 130 90, 126 90, 126 0))''とします。

![XV系](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/12.png)

鹿児島県はこの領域にオーバラップしますが、``WHERE``節で沖縄県に限定しているので問題ありません。

```
-- 15 沖縄県 東経126度から東であり、かつ東経130度から西である区域
UPDATE g_jpr SET syscode=15
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND g_mncpl.pcode=47
    AND ST_Within(g_jpr.geom, 'SRID=4612;POLYGON((126 0, 130 0, 130 90, 126 90, 126 0))'::GEOMETRY);
```

## XVI系およびXVII系

XVI系は、沖縄県で「東経126度から西である地域」すなわち``POLYGON((0 0, 126 0, 126 90, 0 90, 0 0))``の領域にあるものです。

XVII系は、沖縄県で「東経130度から東である区域」すなわち``POLYGON((130 0, 180 0, 180 90, 130 90, 130 0))``の領域にあるものです。

![XVI系とXVII系](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/13.png)

やはり、他の都道府県ともインタセクトしていますが、沖縄県のみと考え、無視します。

```
-- 16 沖縄県 東経126度から西である区域
UPDATE g_jpr SET syscode=16 WHERE pcode=47 AND ST_Covers('SRID=4612;POLYGON((0 0, 126 0, 126 90, 0 90, 0 0))'::GEOMETRY,the_geom);
-- 17 沖縄県 東経130度から東である区域
UPDATE g_jpr SET syscode=17 WHERE pcode=47 AND ST_Covers('SRID=4612;POLYGON((130 0, 180 0, 180 90, 130 90, 130 0))'::GEOMETRY, the_geom);
```

## XVIII系とXIX系

XVIII系は、東京都のうち「北緯28度から南であり、かつ東経140度30分から西である区域」となっている地域で、XIX系は、東京都のうち「北緯28度から南であり、かつ東経143度から東である区域」です。

領域ポリゴンは、XVIII系については``POLYGON((0 0, 140.5 0, 140.5 28, 0 28, 0 0))``、XIX系については、``POLYGON((143 0, 170 0, 170 28, 143 28, 143 0))''となります。

```
-- 18 東京都 北緯28度から南であり、かつ東経140度30分から西である区域
UPDATE g_jpr SET syscode=18
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND g_mncpl.pcode=13
    AND ST_Within(g_jpr.geom, 'SRID=4612;POLYGON((0 0, 140.5 0, 140.5 28, 0 28, 0 0))'::GEOMETRY);
-- 19 東京都 北緯28度から南であり、かつ東経143度から東である区域
UPDATE g_jpr SET syscode=19
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode AND g_mncpl.pcode=13
    AND ST_Within(g_jpr.geom, 'SRID=4612;POLYGON((143 0, 170 0, 170 28, 143 28, 143 0))'::GEOMETRY);
```

## IX系

東京都のうち、XIV系、XVIII系またはXIX系のいずれにも属さない地域です。

```
UPDATE g_jpr SET syscode=9
  FROM g_mncpl
  WHERE g_jpr.mcode=g_mncpl.mcode
    AND g_mncpl.pcode=13
    AND syscode IS NULL;
```

# だいたい見る

まずは、syscodeがNULLになっているレコードがないか確認します。

```
db=# SELECT mcode, syscode, geom FROM g_jpr WHERE syscode IS NULL;
 mcode | syscode | geom 
-------+---------+------
(0 行)
```

あと、おかしいsyscodeが付いていないかをみますが、「ざっくり」でいきましょう。

```
CREATE TABLE w_syscode (
  gid SERIAL PRIMARY KEY,
  syscode INT UNIQUE,
  geom GEOMETRY(POLYGON,4612)
);
CREATE INDEX ix_w_syscode_geom ON w_syscode USING GiST (geom);

INSERT INTO w_syscode(syscode, geom)
SELECT
  syscode,
  ST_ConvexHull(ST_Collect(geom))
FROM g_jpr GROUP BY syscode;
```

``ST_ConvxHull``は、凸包を生成する関数です。凸包は、引数で与えたジオメトリを包含する、最小の、凸のポリゴンです。上の例だと、syscodeごとにまとめられたマルチポリゴンの凸包となります。

生成にはあまり時間がかからないですし、ノード数が大幅に減るので表示させる際に時間が大幅短縮されます。

上の例をQGISで表示すると、次のようになりました。

![syscodeごとのポリゴンをまとめて凸包にしたもの](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/14.png)

時間がかかりますが、平面直角座標系の系番号ごとのポリゴンを``ST_Union``で作ってみたら、次のようになりました。

![平面直角座標系地図](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0008/15.png)

# おわりに

ポリゴンごとに平面直角座標系の系番号を入れていきました。

その際、テーブルのジオメトリとWKTで生成したジオメトリとを''ST_Within''で空間関係をテストしたり、QGISで色分け表示をして確認しながら進めたり、PostGISと直接関係ない操作をしたりと、あれこれ使いながら進めました。

このデータに位置参照情報を連結させるのは、今回よりも面倒ですし、PostgreSQL/PostGISから外れていきます。私の場合はPerlをよく使います。気が向いたら書いてみます。

# 出典
この記事では、国土数値情報（行政区域）を使用しました。
また、平面直角座標系の適用範囲を示す画像の作成に際しては、基盤地図情報を使用しました。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

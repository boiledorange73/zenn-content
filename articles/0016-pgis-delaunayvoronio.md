---
title: "ドロネー図からボロノイ図を作る"
emoji: "😀"
type: "tech"
topics: [PostGIS, OSM]
published: true
---
# ご注意

実装する必要があるのは、PostGIS 2.1以上、PostGIS 2.3未満です。対応バージョンがかなりピンポイントなので、ほとんど意味が無いと思います。
この記事は、どちらかと言うと、ドロネー図からボロノイ図を作成する方法を見て頂くためのものです。

# はじめに

PostGIS 2.1以上では、ドロネー図を生成する関数が用意されています。そこからボロノイ図を作る話です。

ボロノイ図とドロネー図とは関係が深いので、ドロネー図からボロノイ図を作れるのではないかと考える人は多数いらっしゃったと思います。私も「できるんじゃないか」とは思っていましたが、できないでいました。

そして、http://gis.stackexchange.com/questions/114764/how-to-use-st-delaunaytriangles-to-construct-a-voronoi-diagram で、とうとうドロネー図からボロノイ図を作る方法を提示された方が出現しました。さらには解説付きです。

ここでは、それをもうちょっと小さく分割してご紹介したいと思います。

# 概要

## ボロノイ図

![ボロノイ図](https://storage.googleapis.com/zenn-user-upload/2xl1z9pub6f6m5wpwvk6aa9usngc)

複数のポイント（複数の母点）から、どのポイントに最も近いかで領域を分けた図をボロノイ図といいます。

ある位置から最寄のポイントを検索するために、普通なら、全てのポイントについて、そのポイントまでの距離を計算して、距離でソートして最も距離値の小さいものを選択する、という方針を取ります。この際、計算対象ポイントの絞込みで検討が必要となります。範囲を広く取ると冗長な計算を行わざるを得なくなります（二乗のオーダーと見てよい）し、極端に狭く取ると、計算対象ポイントが空になる可能性もあります。

ボロノイ図を使うと、指定位置に最も近いポイントを検索することは、指定位置を含むポリゴンを検索することに置き換えることができます。

母点を小学校や消防署等としてボロノイ図を生成すると、小学校や消防署の所掌範囲の設定に援用することができます。ただし、道路網などの移動時間に関する情報や社会的側面は一切考慮されないので、ボロノイ図だけでは使えないとは思います。

## ドロネー図

![ドロネー図](https://storage.googleapis.com/zenn-user-upload/cgu1ecjm6yzk63kdh8vior1o3gmr)

ドロネー図は、母点を頂点とした三角形で分割したもので、三角形の最小内角が最大になるようにしたものです。

極端に小さな内角を持つ三角形が現れにくく、扱いやすいものですので、標高点の集合からTINを生成したり、有限要素法の領域分割などに使われます。

三角形の外接円の内側に他の点が入ることはない(三角形の外接円の境界上に他の点が1点入る場合はある)という性質があります。

## ドロネー図とボロノイ図を重ねる

ドロネー図はボロノイ図から生成する方法があり、関係が深いものです。

![ドロネー図とボロノイ図を重ねた様子](https://storage.googleapis.com/zenn-user-upload/b78lvjwd636quzgkye10addkxg3d)

ドロネー図の辺はボロノイ図の隣接関係を示すグラフです。上の図でいうと、ボロノイ辺（青線）を共有する領域の母点間にドロネー辺（赤線）が引かれています。

## ドロネー三角形の外接円の中心点を結ぶとボロノイ図になる

もう少し踏み込みます。ボロノイ図の性質から、母点から等距離にある点の集合がボロノイ図の辺を形成するので、ドロネー辺の垂直二等分線（の一部分）は、ボロノイ辺です。注意深く上の図を見ると、赤線の垂直二等分線が青線になっているのが見えます（一部に赤線の垂直二等分線と赤線が交わらない場合があります）。

さらに言うと、ドロネー三角形の外接円の中心点は、三点の等距離上にあり、ボロノイ頂点となります。また、辺（弦）を共有する二つの外接円の中心を結ぶ線は、必ずその辺の垂直二等分線となります（これはドロネー三角形特有の性質ではありません）。

これらより、辺を共有する外接円の中心点を結んでいくことで、線分のボロノイ辺ができあがります。半直線についてはこれではできませんが、少なくとも、外接円の中心点が半直線の端点となります。

# ST_DelaunayTriangles

PostGIS 2.1.0以上では、``ST_DelaunayTriangles``でドロネー図を作成できます。入力はマルチポイントで、出力はポリゴンです。集計関数ではないので、単一ポイントの集合からドロネー図を作りたい場合は、``ST_Collect``でマルチポイントにする必要があります。出力はマルチポリゴンですので、``ST_Dump``でバラすことになると思います。

# 下準備

ポイントデータを用意します。
たとえば次のようにします。ただし、COPYステートメントとデータとは分けてコピーペーストしないとうまくいきません。

```
CREATE TABLE points (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(POINT,0)
);

-- COPYステートメントと中身を一緒にコピペすると失敗するかも
-- 知れないので、分けてコピペしたほうがいいです。
COPY points (geom) FROM stdin;

POINT(12 5)
POINT(5 7)
POINT(2 5)
POINT(19 6)
POINT(19 13)
POINT(15 18)
POINT(10 20)
POINT(4 18)
POINT(0 13)
POINT(0 6)
POINT(4 1)
POINT(10 0)
POINT(15 1)
POINT(19 6)
\.
```

表示させると、次のようになります。

![ポイント](https://storage.googleapis.com/zenn-user-upload/78vtq5pef1ob3klbmb2lfh9j9buj)

# ドロネー図を作り外接円中心点を追加する

まずテーブルを作ります。

```
CREATE TABLE triangles (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(LINESTRING,0),
  p1 GEOMETRY(POINT, 0),
  p2 GEOMETRY(POINT, 0),
  p3 GEOMETRY(POINT, 0),
  circular1 GEOMETRY,
  circular2 GEOMETRY,
  centroid GEOMETRY(POINT, 0)
);
```

## ドロネー図を作る

ドロネー三角形を挿入します。入力はマルチポイントでなければならないので、``ST_Collect``でまとめます。

```
INSERT INTO triangles (geom)
SELECT ST_ExteriorRing((dumped).geom) AS geom
FROM (
  SELECT ST_Dump(ST_DelaunayTriangles(ST_Collect(geom))) AS dumped
  FROM points
) AS Q;
```

結果、次のようになります。

![ドロネー図](https://storage.googleapis.com/zenn-user-upload/43zz44tvdp97rfzx2bfo4qijwtnl)

## 三角形の三辺を格納する

次に、三角形の三辺をp1, p2, p3にそれぞれ格納します。

```
UPDATE triangles
SET
  p1 = ST_PointN(geom,1),
  p2 = ST_PointN(geom,2),
  p3 = ST_PointN(geom,3);
```

## 三角形の外接円中心点を得る

外接円中心点を得るためには外接円が必要です。

まず、三角形の頂点p1,p2,p3を通る円弧(circular1)と三角形の頂点p2,p3,p1を通る円弧(circular2)を生成します。

ここで円弧のコンストラクタ関数が無いので、トリッキーなやり方をしています。いったんWKTにすると、``LINESTRING(x1 y1, x2 y2, x3 y3)``の文字列が得られます。これのうち``LINE``を``CURCULAR``に置き換え、``CIRCULARSTRING(x1 y1, x2 y2, x3 y3)``にしてしまっています。これはやられた。

```
UPDATE triangles
SET
  circular1 = ST_CurveToLine(REPLACE(ST_AsText(ST_LineMerge(ST_Union(ST_MakeLine(p1,p2),ST_MakeLine(p2,p3)))),'LINE','CIRCULAR'),15),
  circular2 = ST_CurveToLine(REPLACE(ST_AsText(ST_LineMerge(ST_Union(ST_MakeLine(p2,p3),ST_MakeLine(p3,p1)))),'LINE','CIRCULAR'),15);
```

母点と、circular1(赤線)、circular2(青線)とを表示すると、次のようになります。このうち、circular1とcircular2とは一部が重なっています。

![二種類の円弧の図](https://storage.googleapis.com/zenn-user-upload/u345bk5jk5ll8a4u2gk8acc1n16m)

これにドロネー三角形をあわせると、次のようになります。

![二種類の円弧の図とドロネー図](https://storage.googleapis.com/zenn-user-upload/5d3qsatm3tupxp5vespm7pemv7qu)

circular1, circular2から外接円を生成し、外接円中心点を格納します。``ST_Union(circular1, circular2)``でマルチラインストリングの近似外接円を生成し、``ST_ConvexHull``でポリゴン化し、``ST_Centroid``で中心点を得ます。``ST_ConvexHull``では思ったものと全く違う図形が生成されることがあると思われるかも知れませんが、円は凸ですので問題にはなりません。

```
UPDATE triangles SET centroid = ST_Centroid(ST_ConvexHull(ST_Union(circular1, circular2)));
```
外接円と中心点とを描いた図を次に示します。

![外接円と中心点の図](https://storage.googleapis.com/zenn-user-upload/4bze2v7hecp8flgi5tsgqjshj6ij)

# 三辺と外接円中心点を別テーブルに抜き出す

''triangles''で作ったジオメトリのうち、三辺と外接円中心点を抜き出します。
ただし、辺ごとに行を変えるようにします。

```
CREATE TABLE edges (
  gid SERIAL PRIMARY KEY,
  edge GEOMETRY(LINESTRING, 0),
  centroid GEOMETRY(POINT, 0)
);

INSERT INTO edges (edge, centroid)
SELECT
  UNNEST(ARRAY[ST_MakeLine(p1,p2),ST_MakeLine(p2,p3),ST_MakeLine(p3,p1)]) AS edge,
  centroid
FROM triangles;
```

# ボロノイ分割の線分を作る

ボロノイ分割の線分を作ります。

```
CREATE TABLE result_lines (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(LINESTRING, 0),
  infline INT
);

INSERT INTO result_lines (geom, infline)
SELECT
  ST_MakeLine(
    Q1.centroid,
    (
      CASE WHEN Q2.gid IS NULL THEN
        CASE WHEN
          ST_WithIn(
            Q1.centroid,
            (SELECT ST_ConvexHull(ST_Collect(geom)) FROM points)
          )
        THEN
          ST_MakePoint(
            ST_X(Q1.centroid) + ((ST_X(ST_Centroid(Q1.edge)) - ST_X(Q1.centroid)) * 2),
            ST_Y(Q1.centroid) + ((ST_Y(ST_Centroid(Q1.edge)) - ST_Y(Q1.centroid)) * 2)
          )
        END
      ELSE
        Q2.centroid
      END
    )
  ),
  (CASE WHEN Q2.gid IS NULL THEN 1 ELSE 0 END) AS infline
FROM edges AS Q1 LEFT JOIN edges AS Q2 ON Q1.gid <> Q2.gid AND ST_Equals(Q1.edge, Q2.edge);
```

``FROM``以降を見てみましょう。テーブルは、``Q1``, ``Q2``とも、同じ``edges``テーブルを指していて、これを``LEFT JOIN``でつなげています。

``ON Q1.gid <> Q2.gid AND ST_Equals(Q1.edge, Q2.edge)``によって、``Q1.edge``と``Q2.edge``が同じで、かつ``Q1``と``Q2``が同じ行を指していない、つまり``Q1``と``Q2``とで辺を共有していることを条件にテーブルを結合します。``LEFT JOIN``としていることで、``Q2``と共有する辺が無い場合には、Q2は全て``NULL``が入ることになります。

``Q2.gid``が``NULL``でない、すなわち、``Q1``と``Q2``とが辺を共有している場合には、``Q1.centroid``から``Q2.centroid``への線を引きます。

``Q2.gid``が``NULL``である場合は、``Q1``と``Q2``とが辺を共有していない(Q1の辺がドロネー三角形の集合のうち最も外側にある辺である)場合には、``Q1.centroid``から``Q1.edge``の中点を通る線分を引きます。

次の図は、赤線が``Q1``と``Q2``とが辺を共有している場合の線、青線が``Q1``と``Q2``とが辺を共有していない場合の線を示しています。

![共有辺と非共有辺](https://storage.googleapis.com/zenn-user-upload/gt8jbxt6k1eqiqxbf66t3buw8tcf)

この線がボロノイ辺となっています。

# 分割線をポリゴンにする

ボロノイ図はポリゴンにしないと使いづらいので、ここまでで作ったボロノイ辺を使って、ポリゴンにします。

## ボロノイ辺線をマージする

```
CREATE TABLE result_multiline (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(MULTILINESTRING, 0)
);
INSERT INTO result_multiline (geom)
SELECT ST_LineMerge(ST_Union(geom)) FROM result_lines;
```

## 外周線を作ってボロノイ辺線にマージする

青線で示したボロノイ辺を持つ領域は、閉じたラインストリングになっていないので、そのままではポリゴン化できません。

そこで、ボロノイ辺の凸包を作り、境界線をラインストリングとして抜き出し、ボロノイ辺にマージします。
さらに、``ST_Node``で

```
CREATE TABLE result_closed (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(MULTILINESTRING, 0)
);

INSERT INTO result_closed (geom)
SELECT
  ST_Node(
    ST_LineMerge(
      ST_Union(
        geom,
        ST_ExteriorRing(ST_ConvexHull(geom))
      )
    )
  )
FROM result_multiline;
```

結果、次のようになり、ポリゴン生成が可能となります。

![ポリゴン化可能なラインストリング](https://storage.googleapis.com/zenn-user-upload/yvp19e8rpj7jpn99k0zxov21heci)

## ポリゴン化する

```
CREATE TABLE result (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(POLYGON, 0)
);

INSERT INTO result (geom)
SELECT geom FROM (
  SELECT (ST_Dump(ST_Polygonize(geom))).geom AS geom
  FROM result_closed
) AS Q
WHERE ST_GeometryType(geom) = 'ST_Polygon';
```
# 正方形の角にあたるマルチポイントだとヘンに見える

``MULTIPOINT(0 0, 10 0, 10 10, 0 10)``のマルチポイントと、それに対する本クエリの結果は、次のようになります。

![四角形の角にあたるマルチポイントに対するボロノイ図](https://storage.googleapis.com/zenn-user-upload/c8j016eyjybti24ok0z30tan41sm)

母点がボロノイ辺上にあり、一見すると計算に失敗したかのように思います。

しかし、ボロノイ辺（縦線と横線）と母点の凸包をとっているのでヘンに見えますが、間違いではありません。

# おわりに

ドロネー図からボロノイ図をPostGISで作る方法を、他人のフンドシで相撲を取るかたちで示しました。

今回は解説のために、冗長にテーブルを作っていきましたが、もっと減らすことができると思います。関数化もできると思いますのでトライしてみてもいいかも知れません。

個人的には、ドロネー図からボロノイ図を作る方法が分からず、ずっと気になっていたものでして、すっきりしました。また、もとのクエリを読んでいて「そうだったのか」「そうすれば良いのか」と驚かされ、なかなか面白いものでした。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

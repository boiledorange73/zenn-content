---
title: "ジオメトリの足し算、引き算をする"
---
# はじめに

データベースに格納されている数値に対して足し算や引き算等を駆使して所望のデータを導出します。たとえば、市区町村ごとの人口データがあって、都道府県単位の人口データが欲しい場合には、都道府県コードでグループに分けてSUMで集計します。

PostGISでは、数値でなくジオメトリに対して足し算や引き算を駆使することができます。上の例に沿うと、市区町村ごとのポリゴンがあって、都道府県単位のポリゴンが欲しい場合には、都道府県コードでグループに分けて``ST_Union``で集計することになります。

今回は、ジオメトリ演算のための関数を紹介します。

## ジオメトリとベン図

下で紹介するジオメトリ演算の関数について、どうもピンと来ない方は、論理演算とベン図を思い出していただければよいかと思います。

# 関数たち

## ST_Union

![ST_Unionの結果](https://storage.googleapis.com/zenn-user-upload/2606sovwnmim11nf4qdp7nrl40ad)

```psql
db=# SELECT ST_AsText(
  ST_Union(
    'POLYGON((0 0, 2 0, 2 2, 0 2, 0 0))'::GEOMETRY,
    'POLYGON((1 1, 3 1, 3 3, 1 3, 1 1))'::GEOMETRY
  )
);
                   st_astext                    
------------------------------------------------
 POLYGON((2 1,2 0,0 0,0 2,1 2,1 3,3 3,3 1,2 1))
(1 行)
```

結合した結果を返します。論理演算とベン図のORに相当します。

## ST_Intersection

![ST_Intersectionの結果](!https://storage.googleapis.com/zenn-user-upload/xyln3ubv74n94cu6sg3n37t3lxxl)

```psql
db=# SELECT ST_AsText(
  ST_Intersection(
    'POLYGON((0 0, 2 0, 2 2, 0 2, 0 0))'::GEOMETRY,
    'POLYGON((1 1, 3 1, 3 3, 1 3, 1 1))'::GEOMETRY
  )
);
           st_astext            
--------------------------------
 POLYGON((1 2,2 2,2 1,1 1,1 2))
(1 行)
```

共有部分を抜き出します。論理演算とベン図のANDに相当します。

## ST_SymDifference

![ST_SymDifferenceの結果](https://storage.googleapis.com/zenn-user-upload/qvr8pgima9foslpz0bkyqodc8kyq)

```psql
db=# SELECT ST_AsText(
  ST_SymDifference(
    'POLYGON((0 0, 2 0, 2 2, 0 2, 0 0))'::GEOMETRY,
    'POLYGON((1 1, 3 1, 3 3, 1 3, 1 1))'::GEOMETRY
  )
);
                                   st_astext                                   
-------------------------------------------------------------------------------
 MULTIPOLYGON(((2 1,2 0,0 0,0 2,1 2,1 1,2 1)),((2 1,2 2,1 2,1 3,3 3,3 1,2 1)))
(1 行)
```

両者の共有していない部分だけを抜き出します。論理演算とベン図のXORに相当します。

## ST_Difference

![ST_Differenceの結果](https://storage.googleapis.com/zenn-user-upload/eibc147ay1bxyz56smkrin490ua2)

```psql
db=# SELECT ST_AsText(
  ST_Difference(
    'POLYGON((0 0, 2 0, 2 2, 0 2, 0 0))'::GEOMETRY,
    'POLYGON((1 1, 3 1, 3 3, 1 3, 1 1))'::GEOMETRY
  )
);
               st_astext                
----------------------------------------
 POLYGON((2 1,2 0,0 0,0 2,1 2,1 1,2 1))
(1 行)
```

「引き算」を行います。論理演算とベン図に相当するものはありません。

交換則は成り立ちません。``ST_Difference``は``ST_Intersection(A, ST_SymDifference(A,B))
``と同じです。

## 集計関数

上記で紹介した関数の全てが、ふたつのジオメトリを引数にとる形式を持ちます。集計関数の形式を持つのは、``ST_Union``のみです（ふたつのジオメトリを取る形式と、集計関数の形式と、配列を引数にとる形式のみっつの形式を持ちます）。

# マルチジオメトリと単一ジオメトリの混在を避ける

これらの関数で生成されるジオメトリは、結果ジオメトリが単一ジオメトリになるかマルチジオメトリになるかは、結果ジオメトリで決定され、引数が単一ジオメトリであってもマルチジオメトリであっても考慮されません。このため、簡単にマルチジオメトリと単一ジオメトリが混合します。

マルチジオメトリと単一ジオメトリとは異なるジオメトリタイプとされるので、マルチジオメトリと単一ジオメトリが混在していると、結構面倒なことになります。

そこで、マルチジオメトリか単一ジオメトリかに揃える必要があります。

どちらに揃えるかは場合によって異なるとは思いますが、とりあえずは「大は小を兼ねる」で、単一ジオメトリもマルチジオメトリにしてしまい、その後に``ST_Dump``で単一ジオメトリに変える方が確認しやすいです。

## ST_Multi

マルチジオメトリを強制するには、``ST_Multi``を使います。この関数の返り値は、引数がマルチジオメトリの場合はそのままの複製を、引数が単一ジオメトリの場合は強制的にマルチジオメトリにしたジオメトリを、それぞれ生成して返します。

# 広島県内の市区町村ポリゴンから広島県ポリゴンを作成する

[シェープファイルのデータをインポートしてみよう コマンドライン編](https://zenn.dev/boiledorange73/books/b1de0a18073af70946e0/viewer/3)で、広島県内の市区町村ポリゴンをインポートしました。

これをST_Unionを使って、広島県ポリゴンを作ってみましょう。

```psql
db=# CREATE TABLE tp (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(MULTIPOLYGON, 4612)
);

db=# INSERT INTO tp (geom) SELECT ST_Multi(ST_Union(geom)) FROM t1;
```

![広島県ポリゴン](https://storage.googleapis.com/zenn-user-upload/txrbl9y2aav2t772zfwtyo1mu47r)

南西地域に不思議な線が浮き出ていますが、現時点では無視して下さい。

全国の市区町村ポリゴンから都道府県ポリゴンを作成する場合には、上記の集計クエリに``GROUP BY``節を追加します。このあたりは、数値の集計と同じようなものです。

# おわりに

今回は、ふたつのジオメトリの結合(``ST_Union``)、共有部分(``ST_Intersection``)、対象差(``ST_SymDifference``)および差(``ST_Difference``)を計算する関数を紹介しました。このうち、``ST_Difference``は交換則が成り立ちません。また、``ST_Union``だけ集計関数の形式を持ちます。

また、ジオメトリ演算の関数を使うとマルチジオメトリと単一ジオメトリが混在することがあり、ST_Multiで単一ポリゴンをマルチポリゴンに変換して処理した方がやりやすいことも述べました。

``ST_Union``は非常によく使うので覚えておくべきです。それ以外の関数も、ちょっとした加工に使ったりするので、全体的に覚えておいたほうが良いでしょう。

# 出典

広島県ポリゴンの作成にあたっては、国土数値情報（行政区域）を使用しています。

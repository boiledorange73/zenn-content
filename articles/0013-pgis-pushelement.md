---
title: "マルチ系ジオメトリに単一ジオメトリを必ず末尾に追加する"
emoji: "😀"
type: "tech"
topics: [PostGIS]
published: false
---
# はじめに

マルチ系ジオメトリに単一ジオメトリを必ず末尾に追加する必要があるとします。

たとえば、マルチポイントにポイントを追加するけれども、マルチポイントのポイント配列の末尾にしたい、という場合です。

まず、マルチ系ジオメトリを、並び順を示す数字を付けたうえで単一ジオメトリにバラして、その結果に新たな単一ジオメトリを追加して、``ST_Collect``でまとめなおす、という方針でやってみます。

# まずはST_Dumpを確認

``ST_Dump``は、マルチ系ジオメトリを単一ジオメトリの集合にバラしてくれるものです。その際、pathとgeomからなるユーザ定義データ型を使いますが、pathは順序の番号が入る配列です。

```psql
db=# WITH pts AS (SELECT 'MULTIPOINT(1 1, 2 2, 3 3, 4 4)'::GEOMETRY AS geom)
  SELECT ST_Dump(geom) FROM pts;
                     st_dump                      
--------------------------------------------------
 ({1},0101000000000000000000F03F000000000000F03F)
 ({2},010100000000000000000000400000000000000040)
 ({3},010100000000000000000008400000000000000840)
 ({4},010100000000000000000010400000000000001040)
(4 行)
```

なんで配列やねんと思うのは正しいと思います。ただ、次のクエリだと配列にしなければならないように思います。

```psql
db=# SELECT ST_DumpPoints('POLYGON((0 0, 10 0, 10 10, 0 10, 0 0),(1 1, 9 1, 9 9, 1 9, 1 1))'::GEOMETRY);
                    st_dumppoints                     
------------------------------------------------------
 ("{1,1}",010100000000000000000000000000000000000000)
 ("{1,2}",010100000000000000000024400000000000000000)
 ("{1,3}",010100000000000000000024400000000000002440)
 ("{1,4}",010100000000000000000000000000000000002440)
 ("{1,5}",010100000000000000000000000000000000000000)
 ("{2,1}",0101000000000000000000F03F000000000000F03F)
 ("{2,2}",01010000000000000000002240000000000000F03F)
 ("{2,3}",010100000000000000000022400000000000002240)
 ("{2,4}",0101000000000000000000F03F0000000000002240)
 ("{2,5}",0101000000000000000000F03F000000000000F03F)
(10 行)
```

``ST_Dump``の場合は、ジオメトリの配列（でできあがっているマルチ系ジオメトリ）をジオメトリにばらすわけですので、pathの要素数は必ず1です。

ですので、気にせず``path[1]``を使います（PostgreSQLの配列の添字は1始まり）。

```psql
db=# WITH pts AS (
    SELECT 'MULTIPOINT(1 1, 2 2, 3 3, 4 4)'::GEOMETRY AS geom
  ),
  dumped AS (
    SELECT (dumped).path[1] AS path, (dumped).geom AS geom FROM (
      SELECT ST_Dump(geom) AS dumped FROM pts
    ) AS Q
  )
  SELECT * FROM dumped;
 path |                    geom                    
------+--------------------------------------------
    1 | 0101000000000000000000F03F000000000000F03F
    2 | 010100000000000000000000400000000000000040
    3 | 010100000000000000000008400000000000000840
    4 | 010100000000000000000010400000000000001040
(4 行)
```

# ここに無理やり単一ジオメトリを追加してみましょう

上の結果に1行分、UNIONで無理やり追加してみましょう。

追加する行の``path``には、``ST_Dump``で出てくる``path``の最大値よりも大きい値にしてやれば、``ORDER BY``で使えるでしょう。

あえてORDER BYを使わないとどうなるでしょうか。

```psql
db=# WITH pts AS (
    SELECT 'MULTIPOINT(1 1, 2 2, 3 3, 4 4)'::GEOMETRY AS geom
  ),
  dumped AS (
    SELECT (dumped).path[1] AS path, (dumped).geom AS geom FROM (
      SELECT ST_Dump(geom) AS dumped FROM pts
    ) AS Q
    UNION SELECT ST_NumGeometries(geom)+1 AS path, 'POINT(100 100)'::GEOMETRY FROM pts
  )
  SELECT * FROM dumped path;

 path |                    geom                    
------+--------------------------------------------
    5 | 010100000000000000000059400000000000005940
    1 | 0101000000000000000000F03F000000000000F03F
    4 | 010100000000000000000010400000000000001040
    3 | 010100000000000000000008400000000000000840
    2 | 010100000000000000000000400000000000000040
(5 行)
```

はい、失敗しました。
なお、このまま``ST_Collect``にかけるとどうなるでしょうか。

```psql
db=# WITH pts AS (
    SELECT 'MULTIPOINT(1 1, 2 2, 3 3, 4 4)'::GEOMETRY AS geom
  ),
  dumped AS (
    SELECT (dumped).path[1] AS path, (dumped).geom AS geom FROM (
      SELECT ST_Dump(geom) AS dumped FROM pts
    ) AS Q
    UNION SELECT ST_NumGeometries(geom)+1 AS path, 'POINT(100 100)'::GEOMETRY FROM pts
  )
  SELECT ST_AsText(ST_Collect(geom)) FROM dumped path;
              st_astext              
-------------------------------------
 MULTIPOINT(100 100,1 1,4 4,3 3,2 2)
(1 行)
```

先頭に付いてしまいました。

では、ORDER BYを付けてみましょう。

```psql
db=# WITH pts AS (
    SELECT 'MULTIPOINT(1 1, 2 2, 3 3, 4 4)'::GEOMETRY AS geom
  ),
  dumped AS (
    SELECT (dumped).path[1] AS path, (dumped).geom AS geom FROM (
      SELECT ST_Dump(geom) AS dumped FROM pts
    ) AS Q
    UNION SELECT ST_NumGeometries(geom)+1 AS path, 'POINT(100 100)'::GEOMETRY FROM pts
  )
  SELECT * FROM dumped ORDER BY path;

 path |                    geom                    
------+--------------------------------------------
    1 | 0101000000000000000000F03F000000000000F03F
    2 | 010100000000000000000000400000000000000040
    3 | 010100000000000000000008400000000000000840
    4 | 010100000000000000000010400000000000001040
    5 | 010100000000000000000059400000000000005940
(5 行)
```

こんなかんじならOKですね。

この順序を保持したまま``ST_Collect``に渡ればOKです。

# それっぽくソートできたけど本当はダメ

ORDER BYを付けたサブクエリを突っ込んでみます。

```psql
WITH pts AS (
    SELECT 'MULTIPOINT(1 1, 2 2, 3 3, 4 4)'::GEOMETRY AS geom
  ),
  dumped AS (
    SELECT (dumped).path[1] AS path, (dumped).geom AS geom FROM (
      SELECT ST_Dump(geom) AS dumped FROM pts
    ) AS Q
    UNION SELECT ST_NumGeometries(geom)+1 AS path, 'POINT(100 100)'::GEOMETRY FROM pts
  )
  SELECT ST_AsText(ST_Collect(geom)) FROM (
    SELECT geom FROM dumped ORDER BY path
  ) AS Q1;

              st_astext              
-------------------------------------
 MULTIPOINT(1 1,2 2,3 3,4 4,100 100)
(1 行)
```

そうそう、こんなかんじ。これならOKな結果ですね、たまたまですが。

これ、実は順序は保障されていないんだそうです。

https://aginfo.cgk.affrc.go.jp/docs/pgisman/2.4.0/ST_MakeLine.html あたりに書いてたりします。

# 集約関数の中にORDER BYを書くといいらしい (9.0以降)

集約関数の実引数の中に``ORDER BY``を書いておくと、この並び順が保障されたものが集約関数に渡ります。これなら確実です。

https://www.sraoss.co.jp/technology/postgresql/9.0/ 等を参照して下さい。

9.0以降という条件が付いていますが、きょうび8.x以下を使う機会もそうとう少ないので、まあ実質的にバージョン制限はかかってないと見ても許されるかなと思います。

```psql
db=# WITH pts AS (
    SELECT 'MULTIPOINT(1 1, 2 2, 3 3, 4 4)'::GEOMETRY AS geom
  ),
  dumped AS (
    SELECT (dumped).path[1] AS path, (dumped).geom AS geom FROM (
      SELECT ST_Dump(geom) AS dumped FROM pts
    ) AS Q
    UNION SELECT ST_NumGeometries(geom)+1 AS path, 'POINT(100 100)'::GEOMETRY FROM pts
  )
  SELECT ST_AsText(ST_Collect(geom ORDER BY path)) FROM dumped;
              st_astext              
-------------------------------------
 MULTIPOINT(1 1,2 2,3 3,4 4,100 100)
(1 行)
```

# おわりに

マルチ系ジオメトリに対して新しいジオメトリを必ず末尾に追加する方法として、一度バラして、ジオメトリを追加し、求める通りの順序でまとめる方法をご紹介しました。

この際に、``ST_Dump``の``path``の使い方と、集約関数の実引数に``ORDER BY``を書き込んで並び順を崩さずに渡すことができることをご紹介しました。

並び順が気になる場合には、こういった若干ややこしいこともする必要があります。もちろん、並び順が気にならないなら、もっと簡単にできます（``ORDER BY``を入れる必要がなくなるだけでなく、``path``を引く必要もなくなりますから）。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

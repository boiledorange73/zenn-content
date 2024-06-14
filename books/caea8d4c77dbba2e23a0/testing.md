---
title: "PostGISの空間関係をテストする関数たち"
---
# はじめに

PostGISでは「図形」を扱うことになります。
WHERE条件の式で、スカラ値を扱う場合でしたら大小関係とか等価とかの演算子ですみますが、図形では全く違う道具を使う必要があります。

そして困ったことに、その道具の種類が多く、ある程度PostGISとかを扱っていた人にとっても整理しきれていないと思われます（ソースは自分）。

そこで今回は、WHERE条件で使う、空間関係をテストする関数のざっとご紹介します。

# 概要

## 関数名規則

OGC SQL/MM関数は、先頭に"ST_"が付きます。後半は、原則として、関数名が 動詞+三人称のs からなり、「（第一引数）が（第二引数）を（動詞）する」という意味を持ちます。

たとえば、ST_Contains(A,B) は、「AがBを含む」という意味になります。

be動詞+過去分詞（受動態）等の場合には、be動詞は省略されます。
ST_CoverdBy(A, B) は "A is coverd by B" つまり「AはBに覆われている」という意味になります。

## 点、面、線と次元

ここで出てくる「次元」は座標軸の次元ではありません。

ジオメトリを「点」「線」「面」に分別するためのものであって、点は０次元、線は１次元、面は２次元、となります。

## 内部、境界、外部

ジオメトリの関係関数を理解するためには、ジオメトリの内部、境界、外部、のみっつを押さえておく必要があります。

外部は、ジオメトリの外側の領域を指します。これはごく自然に受け入れられると思います。

内部は、ジオメトリのうち、境界を除いた部分です。これも、境界ということばが出てくるので、境界を除いた部分に限定するのは受け入れられると思います。

若干困るのが境界です。
ジオメトリが面の場合には、ジオメトリの境界線が「境界」にあたります。これはまあ問題ないと思います。
ジオメトリが線の場合には、へんに思うかも知れませんが、端点が境界です。ラインストリングの節点は境界ではありません。
ジオメトリが点の場合には、境界はありません。

面、線の境界（赤色）と内部（青色）を以下に示します。

![面の境界](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/testing/1.png)

![線の境界](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/testing/2.png)

# 「含む」系関数

## ST_ContainsとST_Covers

![ST_ContainsとST_Coversのテスト結果](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/testing/3.png)

ST_ContainsとST_Coversは、いずれも、AがBを含む、というかんじの意味で、概ね同じ真偽値を返します。

が、若干異なることがあります。

ST_Containsは、Aの外部にBがなく、Aの内部にBの内部がある場合にTRUEを返します。
ST_Coversは、Aの外部にBがない場合にTRUEを返します。

ST_Containsの二つ目の条件がクセモノです。
Aがポリゴンで、BがAの境界と同じラインストリングとします。Bは、Aの境界と共通部分を持ちますが、Aの内部とは共通部分を持ちません。よってST_Contains(A,B)は偽で、ST_Covers(A,B)は真となります。

```
db=# SELECT ST_Contains('POLYGON((0 0, 1 0, 1 1, 0 0))'::GEOMETRY,'LINESTRING(0 0, 1 0, 1 1, 0 0)'::GEOMETRY);
 st_contains 
-------------
 f
(1 行)

db=# SELECT ST_Covers('POLYGON((0 0, 1 0, 1 1, 0 0))'::GEOMETRY,'LINESTRING(0 0, 1 0, 1 1, 0 0)'::GEOMETRY);
 st_covers 
-----------
 t
(1 行)
```

## ST_WithInとST_CoveredBy

![ST_WithInとST_CoveredByのテスト結果](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/testing/4.png)

ST_WithInはST_Containsと引数を入れ替えたものです。ST_CoveredByもST_Coversと引数を入れ替えたものです。
それぞれ、単に引数の順序を変えただけです。

## ST_ContainsProperly

ST_Containsをみたし、かつBがAの境界と共通部分を持たない場合にTRUEを返します。
ST_Containsとほぼ同じですが、ふたつの引数が同じジオメトリである場合には、違いが出ます。
ST_Contains(A,A)はTRUEを返しますが、ST_ContainsProperly(A,A)はFALSEを返します。

# 「交差」系関数

AとBに共通部分があり、かつ一方がもう一方に包含されない関係をテストする関数には、
ST_Intersects, ST_Crosses, ST_Overlaps, ST_Touches があります。あと、NOT ST_Intersects を意味するST_Disjointもあります。

このあたりは結構ややこしいです。特に、intersectとcrossは、いずれも「交差する」と訳されますので、クセモノ感が倍増します。

## ST_Intersects

![ST_Intersectsのテスト結果](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/testing/5.png)

ST_Intersectsは、AとBに共通部分が存在する場合にTRUEを返します。それ以上の条件はありません。

## ST_Crosses
ST_Crossesは次の全てを満たす場合にTRUEを返します。

1. AとBに共通部分がある
1. 共通部分の次元がAの次元とBの次元の最大値より小さい
1. 共通部分がAと一致しない
1. 共通部分がBと一致しない

ST_Intersectよりも条件が厳しくなっています。

+ Aが複数の点でなっている場合には、Bが線または面で、Aの要素にBの内部にあるポイントと、Bの外部にあるポイントの両方が存在すればクロスしていることになります。
+ Aが線で、Bが線の場合には、共通要素が点であるなら、クロスしていることになります。
+ Aが線で、Bが面の場合には、Aの一部がBの外にあるなら、クロスしていることになります。線が面を突き刺している状態です。

上記の例を次に示します。青い丸が複数あるのは「複数の点」でひとつのジオメトリを形成していることに注意して下さい。

![ポリゴンとマルチポイントのクロス](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/testing/6.png)

![ラインストリングとラインストリングのクロス](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/testing/7.png)

![ポリゴンとラインストリングのクロス](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/testing/8.png)

また、次の点にも注意が必要です。

+ AとBがともに点である場合には、共通部分があっても必ず点であり、第２条件から、クロスすることはありません。
+ Aが単一の点である場合には、第３条件または第４条件から、クロスすることがありません。
+ AとBがともに面である場合には必ずクロスしません。本来は同一面上にジオメトリがあるという条件下でのみ成り立ちますが。

## ST_Overlaps

![ST_Overlapsのテスト結果](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/testing/9.png)

AとBに共通部分が存在し、AとBと共通部分は同じ次元で、かつAまたはBが、もう一方に包含さていない場合にTRUEを返します。

## ST_Disjoint

共通部分が無い場合にTRUEを返します。

![ST_Disjointのテスト結果](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/testing/10.png)

# 「接触」系関数

## ST_Touches

![ST_Touchesのテスト結果](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/testing/11.png)

![ST_Touchesのテスト結果 その2](https://github.com/boiledorange73/zenn-content/raw/main/books-images/caea8d4c77dbba2e23a0/testing/12.png)

AとBに共通部分があり、共通部分はAの境界上とBの境界上にしかない場合にTRUEを返します。

# 「同じ」系関数

## ST_Equals

ST_Within(A,B) AND ST_Within(B,A) である場合にTRUEを返します。構成するノードの順序、同一線上のノードの多寡は考慮しません。

## ST_OrderingEquals

ST_Equals(A,B)かつノードの順序が同じである場合にTRUEを返します。

## ST_EqualsとST_OrderingEqualsの異同

同じ位置に同じ形状であるAとBについて、ST_Equals()とST_OrderingEquals()との異同を見ていきます。

同じノード数、ノード順である場合は、どちらもTRUEを返します。

```
db=# SELECT ST_Equals(A,B), ST_OrderingEquals(A,B) FROM
(
  SELECT
    'POLYGON((0 0, 1 0, 1 1, 0 1, 0 0))'::GEOMETRY AS A,
    'POLYGON((0 0, 1 0, 1 1, 0 1, 0 0))'::GEOMETRY AS B
) AS Q;
 st_equals | st_orderingequals 
-----------+-------------------
 t         | t
(1 行)
```

ノード順が右回りと左回りとで異なる場合は、ST_Equals()はTRUEを返し、ST_OrderingEquals()はFALSEを返します。

```
db=# SELECT ST_Equals(A,B), ST_OrderingEquals(A,B) FROM
(
  SELECT
    'POLYGON((0 0, 1 0, 1 1, 0 1, 0 0))'::GEOMETRY AS A,
    'POLYGON((1 0, 1 1, 0 1, 0 0, 1 0))'::GEOMETRY AS B
) AS Q;
 st_equals | st_orderingequals 
-----------+-------------------
 t         | f
(1 行)
```

どちらも位置、形状は同じなうえ、どちらも左回りだけれども、Bに余分なノードが挿入されている場合は、ST_Equals()はTRUEを返し、ST_OrderingEquals()はFALSEを返します。

```
db=# SELECT ST_Equals(A,B), ST_OrderingEquals(A,B) FROM
(
  SELECT
    'POLYGON((0 0, 1 0, 1 1, 0 1, 0 0))'::GEOMETRY AS A,
    'POLYGON((0 0, 1 0, 1 1, 0 1, 0 0.1, 0 0))'::GEOMETRY AS B
) AS Q;
 st_equals | st_orderingequals 
-----------+-------------------
 t         | f
(1 行)
```

# おわりに

ジオメトリの位置的な関係をテストする関数を紹介しました。また、「内部」「境界」「外部」の概念について紹介しました。

ここの関数以外のテストは、基本的にはありませんので、しっかり覚えておいた方がいいと思います。

また、「境界」が少し思っていたものと違うかもしれませんが、こういうものとお考え下さい。しばらくすると、この方がしっくりくるようになります。

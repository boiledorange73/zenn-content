---
title: "コンマ区切り文字列を分割してみる"
emoji: "😀"
type: "tech"
topics: [PostgreSQL]
published: false
---
# はじめに

[空港近くの水平表面をつくる](0031-pgis-airportcircle)で、空港近くに設定される制限表面のうち水平表面をざっくり作りました。

続いて進入平面を作ってみようと思ったのですが、たいへんなことが判明。滑走路長と滑走路幅はありますが、滑走路の形状データが無いという。なので、自分で作ってみようと思います。

その前段階として、滑走路テーブルを作り、国土数値情報（空港）をそのまま叩き込んだ ``airport`` テーブルとリレーションをはります。

なお、空港に1本以上の滑走路が存在します。複数は十分にありえます。なので、仮に滑走路をrunwayとしたら、airportとrunwayは1対多の関係になります。

## ご注意

今回はPostGIS度がゼロです。

# 前段階

## 滑走路データの確認

とりあえず滑走路長と滑走路幅を見てみましょう。

```psql
db=# SELECT c28_005, c28_012, c28_013 FROM airport WHERE c28_005='東京国際空港';
   c28_005    |       c28_012       |   c28_013   
--------------+---------------------+-------------
 東京国際空港 | 3000,2500,3360,2500 | 60,60,60,60
(1 行)
```

各滑走路をコンマ区切りにした文字列で収められています。確かに羽田空港は滑走路が4本ありますね。

目で確認はできたけれども、このcsvを分離してやらないと、1対多のrunwayテーブルは作れません。splitみたいなものを使って行に分離しないといけないですね。

なお、この問題は国土数値情報がおかしいことをしているわけではなく、シェープファイルというデータ書式の限界で、仕方なくこうしているようです。実際、同じzipに納められているGMLファイルでは、滑走路が複数ある場合にはRunway要素が複数現れるようになっていて、このようなコンマ区切りではありません。

## 紋別空港のチェック

前回も出てきましたが、``airport``テーブルでは、紋別空港は2行に分かれています。滑走路関連で困ることはないか確認します。

```psql
db=# SELECT c28_005, c28_012, c28_013 FROM airport WHERE c28_005='紋別空港';
 c28_005  | c28_012 | c28_013 
----------+---------+---------
 紋別空港 | 2000    | 45
 紋別空港 | 2000    | 45
(2 行)

db=# =# SELECT DISTINCT c28_005, c28_012, c28_013 FROM airport WHERE c28_005='紋別空港';
 c28_005  | c28_012 | c28_013 
----------+---------+---------
 紋別空港 | 2000    | 45
(1 行)
```

DISTINCTにすると1行になったので、問題なさげです。

# splite的な関数

regexp_split_to_array, regexp_split_to_table があてはまります。

どちらも指定された正規表現をデリミタとして繋がっている文字列を分割します。結果にはデリミタはくっつきません。

regexp_split_to_arrayは、分割した結果を文字列配列で返し、regexp_split_to_tableは、文字列集合(setof text)で返します。

```psql
db=# SELECT regexp_split_to_table('11,12,13,14', ',');
 regexp_split_to_table 
-----------------------
 11
 12
 13
 14
(4 行)

db=# SELECT regexp_split_to_array('11,12,13,14', ',');
 regexp_split_to_array 
-----------------------
 {11,12,13,14}
(1 行)
```

## regexp_split_to_tableを2カラムで使うのをためらう

分割自体は問題なさそうですが、今回は、滑走路長と滑走路幅の2カラムで同時に分割しないといけません。

こんなかんじです。

```psql
db=# SELECT
  regexp_split_to_table('11,12,13,14',','),
  regexp_split_to_table('21,22,23,24',',');
 regexp_split_to_table | regexp_split_to_table 
-----------------------+-----------------------
 11                    | 21
 12                    | 22
 13                    | 23
 14                    | 24
(4 行)
```

これだと問題ありません。ところが、次の場合はどうでしょう。

```psql
db=# SELECT
  regexp_split_to_table('11,12,13,14',','),
  regexp_split_to_table('21,22,23',',');
 regexp_split_to_table | regexp_split_to_table 
-----------------------+-----------------------
 11                    | 21
 12                    | 22
 13                    | 23
 14                    | 21
 11                    | 22
 12                    | 23
 13                    | 21
 14                    | 22
 11                    | 23
 12                    | 21
 13                    | 22
 14                    | 23
(12 行)
```

デカルト積になってしまいます。


## 配列要素番号を得る

じゃあ、2カラムで配列を生成して、一方の添え字のシリーズを抽出してループを回して、もう一方の配列は添え字を使う、というのがいいかな、と考えました。

配列の添え字のシリーズって出るの？と思ったらありました。

```psql
db=# SELECT generate_subscripts(ARRAY[20,19,21,23,17], 1);
 generate_subscripts 
---------------------
                   1
                   2
                   3
                   4
                   5
(5 行)
```

``generate_subscripts``の第2引数は、次元です。今回は1次元配列を使っているので1しかありえません。

これで添え字が出てくるので、中身を取り出してみます。

```psql
db=# SELECT arr[ix] FROM (
  SELECT generate_subscripts(arr,1) AS ix, arr
  FROM (
    SELECT ARRAY[20,19,21,23,17] AS arr) AS Q1
  ) AS Q2;
 arr 
-----
  20
  19
  21
  23
  17
(5 行)
```

2カラムでやってみましょう。

```psql
db=# SELECT ary1[arg] AS e1, ary2[arg] AS e2 FROM (
  SELECT ary1, ary2, generate_subscripts(ary1, 1) AS arg FROM (
    SELECT ARRAY[20,19,21,23,17] AS ary1, ARRAY[17,23,21,19,20] AS ary2
  ) AS Q1
) AS Q2;
 e1 | e2 
----+----
 20 | 17
 19 | 23
 21 | 21
 23 | 19
 17 | 20
(5 行)
```

出ますね。

# クエリを段階的に作っていく

では滑走路テーブルに入れるタプルを作っていってみます。

まず、紋別空港の重複を除いて(DISTINCT)、コンマ区切り文字列を配列にします。
```
db=# SELECT
  c28_005,
  regexp_split_to_array(c28_012, ',') AS arr_length,
  regexp_split_to_array(c28_013, ',') AS arr_width
FROM (
  SELECT DISTINCT c28_005, c28_012, c28_013 FROM airport
) AS Q1;

    c28_005     |      arr_length       |   arr_width   
----------------+-----------------------+---------------
 奥尻空港       | {1500}                | {45}
...
 大阪国際空港   | {1828,3000}           | {45,60}
...
(96 行)
```

次に配列添字集合を出します。この時点ではarr_length, arr_width には何もしていません。

``` psql
db=# SELECT
  c28_005,
  generate_subscripts(arr_length, 1) AS number,
  arr_length,
  arr_width
FROM (
  SELECT
    c28_005,
    regexp_split_to_array(c28_012, ',') AS arr_length,
    regexp_split_to_array(c28_013, ',') AS arr_width
  FROM (
    SELECT DISTINCT c28_005, c28_012, c28_013 FROM airport
  ) AS Q1
) AS Q2;

    c28_005     | number |      arr_length       |   arr_width   
----------------+--------+-----------------------+---------------
 奥尻空港       |      1 | {1500}                | {45}
...
 大阪国際空港   |      1 | {1828,3000}           | {45,60}
 大阪国際空港   |      2 | {1828,3000}           | {45,60}
...
(107 行)
```

配列添字(number)を使って、配列の要素を抽出したらできあがり。

あと、regexp_split_to_arrayで生成されるのは文字列配列で、最終的には整数にしたいので、キャストしちゃいましょう。

```psql
db=# SELECT 
  c28_005,
  number,
  arr_length[number]::INT AS length,
  arr_width[number]::INT AS width
FROM (
  SELECT
    c28_005,
    generate_subscripts(arr_length, 1) AS number,
    arr_length,
    arr_width
  FROM (
    SELECT
      c28_005,
      regexp_split_to_array(c28_012, ',') AS arr_length,
      regexp_split_to_array(c28_013, ',') AS arr_width
    FROM (
      SELECT DISTINCT c28_005, c28_012, c28_013 FROM airport
    ) AS Q1
  ) AS Q2
) AS Q3;

    c28_005     | number | length | width 
----------------+--------+--------+-------
 奥尻空港       |      1 |   1500 |    45
...
 大阪国際空港   |      1 |   1828 |    45
 大阪国際空港   |      2 |   3000 |    60
...
(107 行)
```

# テーブルに入れる

まず、テーブルを作ります。
```psql
db=# CREATE TABLE t_runway (
  gid SERIAL PRIMARY KEY,
  name TEXT REFERENCES g_airport(name),
  number INT,
  length INT,
  width INT,
  UNIQUE (name, number)
);
```

挿入します。

```psql
db=# INSERT INTO t_runway (name, number, length, width)
SELECT 
  c28_005,
  number,
  arr_length[number]::INT AS length,
  arr_width[number]::INT AS width
FROM (
  SELECT
    c28_005,
    generate_subscripts(arr_length, 1) AS number,
    arr_length,
    arr_width
  FROM (
    SELECT
      c28_005,
      regexp_split_to_array(c28_012, ',') AS arr_length,
      regexp_split_to_array(c28_013, ',') AS arr_width
    FROM (
      SELECT DISTINCT c28_005, c28_012, c28_013 FROM airport
    ) AS Q1
  ) AS Q2
) AS Q3;

INSERT 0 107
```

最後にテーブルを確認します。

```psql
db=# SELECT * FROM t_runway;
 gid |      name      | number | length | width 
-----+----------------+--------+--------+-------
  16 | 奥尻空港       |      1 |   1500 |    45
  17 | 新千歳空港     |      1 |   3000 |    60
  18 | 新千歳空港     |      2 |   3000 |    60
  19 | 福井空港       |      1 |   1200 |    30
  20 | 下地島空港     |      1 |   3000 |    60
  21 | 広島空港       |      1 |   3000 |    60
  22 | 三沢飛行場     |      1 |   3050 |    45
...
(107 行)
```

# おわりに

``regexp_split_to_array``と``generate_subscripts``を使って、2カラムの文字列を分割してテーブルに納めました。

文字列分割はできれば避けたいところですが、インポート元の書式の制約から、分割せざるをえない場合もあります。そういった場合でもPostgreSQLは、そういったことにも対応してくれました。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

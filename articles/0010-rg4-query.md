---
title: "逆ジオコーディングのためのデータ整備 #4 クエリを投げる"
emoji: "😀"
type: "tech"
topics: [PostGIS]
published: true
---
# はじめに

「逆ジオコーディングのためのデータ整備」と称して、[国土数値情報のインポートと正規化](https://zenn.dev/boiledorange73/articles/d49d208c654c1b58b4d7)、
[市区町村ポリゴンごとの平面直角座標系の割り当て](https://zenn.dev/boiledorange73/articles/441a025db9914be8677e)、[位置参照情報のインポート](https://zenn.dev/boiledorange73/articles/ae59d0b37fbe55081756)を行ってきました。

ここでは、整備したデータに対して二つのクエリを投げてみます。「逆ジオコーディング」は難しそうに見えますが、クエリ自体は非常に簡単です。あとはガワを付けてやれば逆ジオコーディングサービスの完成です（そのガワが面倒臭いんですけどね）。

それだけでは面白くないので、もう少しクエリを投げてみましょう。

# 逆ジオコーディング

## 市区町村の判定

```psql
db=# SELECT mcode, syscode FROM g_jpr
WHERE ST_Contains(geom, 'SRID=4612;POINT(135 35)'::GEOMETRY);
 mcode | syscode 
-------+---------
 28213 |       5
(1 行)
```
POINTの数値を変えることができます。

## 街区の判定

若干ややこしくなります。

まず、``SRID=4612;POINT(135 35)``の位置は、市区町村コードが28213で、平面直角座標系5系にあたります。なので、``g_section``のうち、``mcode=28213 AND syscode=5``の行を抜き出せば良いことになります。

が、まずは対象となる行数を見てみましょう。

```psql
db=# SELECT count(*) FROM g_section WHERE mcode=28213 AND syscode=5;
 count 
-------
  7510
(1 行)
```

ちょっと多いですね。

そこで、``ST_Dwithin``を使い、一定の距離内にあるもののみを使用することにしましょう。

``ST_DWithin(GEOMETRY,GEOMETRY,FLOAT)``の距離計算は、単純にユークリッド距離を計算するだけですので、地理座標系では使えません。そのために、平面直角座標系によるポイントデータを追加しています(``g_section.jpr_geom``)。

``SRID=4612;POINT(135 35)``を平面直角座標系V系にあわせるには、``ST_Transform()``を使います。JGD2000 平面直角座標系V系のSRIDは``2442+5=2447``です。


```psql
db=# SELECT count(*) FROM g_section WHERE mcode=28213 AND syscode=5
    AND ST_DWithin(
      jpr_geom,
      ST_Transform('SRID=4612;POINT(135 35)'::GEOMETRY, 2447),
      1000
    );
ERROR:  Operation on two GEOMETRIES with different SRIDs
```

g_section.jpr_geomは、系が入り乱れることになるので、SRID=0にしていますが、ST_Transform()で生成されるポイントのSRIDは2447ですので、SRIDが異なり、エラーとなりました。

SRIDを強制的に0にしてみましょう。

```psql
db=# SELECT count(*) FROM g_section WHERE mcode=28213 AND syscode=5
    AND ST_DWithin(
      jpr_geom,
      ST_SetSRID(
        ST_Transform('SRID=4612;POINT(135 35)'::GEOMETRY, 2447),
        0
      ),
      1000
    );
 count 
-------
   550
(1 行)
```

``ST_DWithin()``を使わなかった場合には7510行だったので、7.3%程度にまで絞り込めました。

```psql
SELECT section, blockcode, ST_Distance(
      jpr_geom,
      ST_SetSRID(
        ST_Transform('SRID=4612;POINT(135 35)'::GEOMETRY, 2447),
        0
      )
    ) FROM g_section WHERE mcode=28213 AND syscode=5
    AND ST_DWithin(
      jpr_geom,
      ST_SetSRID(
        ST_Transform('SRID=4612;POINT(135 35)'::GEOMETRY, 2447),
        0
      ),
      1000
    )
  ORDER BY ST_Distance(
    jpr_geom,
    ST_SetSRID(
      ST_Transform('SRID=4612;POINT(135 35)'::GEOMETRY, 2447),
      0
    )
  )
  LIMIT 5
  ;
 section  | blockcode |   st_distance    
----------+-----------+------------------
 上比延町 | 330       |  136.72427501881
 上比延町 | 334       |  136.72427501881
 上比延町 | 314       |  136.72427501881
 比延町   | 543       | 210.404389922206
 比延町   | 550       | 217.461596355077
(5 行)
```

``SELECT``の``ST_Distance``と``ORDER BY``の``ST_Distance``とは同じで、冗長ですが、今回は無視して下さい。

``ORDER BY``で、距離についてソートするよう指示し、``LIMIT``で、上位何行までを返すかを指示しています。

# 系をまたぐ市町村

平面直角座標系は、[空間参照系の概要](https://zenn.dev/boiledorange73/books/caea8d4c77dbba2e23a0/viewer/srs)で示したとおり、わが国を19に分割しています。都道府県のうちの多くは、ひとつの府県で複数の系にまたがることはありませんが、北海道、東京都、鹿児島県、沖縄県は、ひとつの都道県で複数の系にまたがります。

では、このうち、ひとつの市町村で複数の系にまたがることがあるでしょうか。

逆ジオコーディングサービスでのデータ構造からは、同じ市町村でも系が異なれば別のエントリとして扱っているので、特に問題は生じませんが、どれぐらい系をまたいでいるかを見てみましょう。

## 国土数値情報（行政区域）

国土数値情報（行政区域）から作成したg_jprで確認してみましょう。

```psql
db=# SELECT Q2.mcode, bname, gname, mname, cnt FROM (
    SELECT mcode, count(*) AS cnt FROM (
      SELECT DISTINCT mcode, syscode FROM g_jpr
    ) AS Q1 GROUP BY mcode
  ) AS Q2
  LEFT JOIN g_mncpl ON Q2.mcode=g_mncpl.mcode
  WHERE cnt > 1 ORDER BY mcode;
 mcode | bname |  gname   |   mname    | cnt 
-------+-------+----------+------------+-----
 13421 |       |          | 小笠原村   |   4
 46215 |       |          | 薩摩川内市 |   2
 46220 |       |          | 南さつま市 |   2
 46303 |       | 鹿児島郡 | 三島村     |   2
 46304 |       | 鹿児島郡 | 十島村     |   2
(5 行)
```

小笠原村と鹿児島県の離島部で、系をまたいでいる市と村があるのが分かります。

## 街区レベル位置参照情報

街区レベル位置参照情報は、都市計画区域内に限定されています（住居表示の有無は関係ありません）。

市や村が系をまたいでいても、街区レベル位置参照情報のポイントデータが存在する区域だけに注目すると、ひとつの系に限定されているかも知れません。

```psql
db=# SELECT Q2.mcode, bname, gname, mname, cnt FROM (
    SELECT mcode, count(*) AS cnt FROM (
      SELECT DISTINCT mcode, syscode FROM g_section
    ) AS Q1 GROUP BY mcode
  ) AS Q2
  LEFT JOIN g_mncpl ON Q2.mcode=g_mncpl.mcode
  WHERE cnt > 1 ORDER BY mcode;
 mcode | bname | gname | mname | cnt 
-------+-------+-------+-------+-----
(0 行)
```

# おわりに

逆ジオコーディングサービスのためのデータ整備を行ったところに、クエリを投げて様子を見てみました。

これで、だいたい逆ジオコーディングサービスが作れるはずです。が、データ更新の方が面倒ですので、ご注意下さい。
# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

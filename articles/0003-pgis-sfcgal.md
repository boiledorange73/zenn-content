---
title: " PostGIS+SFCGALの話"
emoji: "😀"
type: "tech"
topics: [PostGIS]
published: false
---
# SFCGAL

CGAL (http://www.cgal.org/) は高性能な幾何演算ライブラリで、SFCGAL (http://oslandia.github.io/SFCGAL/) は CGAL を ISO 19107:2013 と OGC Simple Features Access 1.2 for 3D operations とに対応するためのラッパライブラリです。

PostGISは2.1以降でSFCGALに対応しています。

SFCGAL利用関数はいくつかありますが、はっきり言う、私もよく分かっていません。http://smathermather.wordpress.com/2013/12/04/postgis-with-sfcgal/ で興奮されている方がいらっしゃる、ということを確認していましたが、それ以上の情報は得られていませんでした。

とりあえず分かるところからメモっていきます。

## 幾何演算ならGEOSがあるじゃないかと思ったあなた

あなたはするどい。確かに、機能面で共通する部分があります。

PostGIS 2.1 から 2.5 までは、共通機能を使用する関数でSFCGALを使うかGEOSを使うかを切り替えられるようになっていました。http://aginfo.cgk.affrc.go.jp/docs/pgisman/2.2.0/postgis_backend.html あたりを参照して下さい。

PostGIS 3では、切り替えができなくなり、共通機能はGEOSを使い、SFCGAL独自のものだけSFCGALを使うようになっています。

# インストールなど

まずは、CGAL, SFCGALをインストールする必要があります。
FreeBSDのportsには既に入れてくれてあります。助かります、本当に助かります。

PostGISのコンフィギュア時に `--with-sfcgal=/.../sfcgal-config` を追加。sfcga-configへのパスは、Packagesから入れた場合には "/usr/local/bin/" でした。

インストールまで終わったら、今度はPostGISのデータベースへのインストールと、SFCGAL関数のインストールを行います。

PostGIS 2.2 では、次のステートメントを順次実行します。

```psql
% psql -d (データベース) -c "CREATE EXTENSION postgis"
% psql -d (データベース) -c "CREATE EXTENSION postgis_sfcgal"
```

2.1ではEXTENSIONではなく、別途インストール用スクリプトがあります。

```csh
% psql -d (データベース) -c "CREATE EXTENSION postgis"
% psql -d (データベース) -f (postgresql)/contrib/sfcgal.sql
```


# 関数たち
## postgis_sfcgal_version

SFCGALのバージョンを示す文字列を返します。

```
db=# SELECT postgis_sfcgal_version();
 postgis_sfcgal_version 
------------------------
 1.0.4
(1 行)
```

## ST_Extrude
3D CADでは「押し出し」とも言われる、おなじみの機能。

```
db=# SELECT ST_AsText(ST_Extrude('POLYGON((0 0, 10 0, 10 10, 0 0))'::GEOMETRY,0,0,1));
                                                                                                st_astext                                                                                                 
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 POLYHEDRALSURFACE Z (((0 0 0,10 10 0,10 0 0,0 0 0)),((0 0 1,10 0 1,10 10 1,0 0 1)),((0 0 0,0 0 1,10 10 1,10 10 0,0 0 0)),((10 10 0,10 10 1,10 0 1,10 0 0,10 10 0)),((10 0 0,10 0 1,0 0 1,0 0 0,10 0 0)))
(1 行)

```

注意深く追っていけば、三角柱であることが分かると思います。

なおここで、本当に注意深く追っていると、法線方向に見て時計回りになっていて(右ねじ)、逆に外から内を見て反時計回りになっているのが分かると思います。

三次元の面には表裏あって、どちらが立体の外側でどちらが内側かが分からないので、「右ねじ」で表現することになります。STL (Standard Triangulated Language)がまさにそれ。STLデータ生成等の経験があるなら、こうなってくれてるとありがたい、というのが分かっていただけるかと思います。


## ST_StraightSkeleton

コの字型を左右逆にしたポリゴンに適用してみます。

```
db=# SELECT ST_AsText(ST_StraightSkeleton('POLYGON((0 0, 50 0, 50 10, 10 10, 10 20, 50 20, 50 30, 0 30, 0 0))'::GEOMETRY));
                                                                        st_astext                                                                        
---------------------------------------------------------------------------------------------------------------------------------------------------------
 MULTILINESTRING((0 0,5 5),(50 0,45 5),(50 10,45 5),(10 10,5 5),(10 20,5 25),(50 20,45 25),(50 30,45 25),(0 30,5 25),(45 5,5 5),(45 25,5 25),(5 5,5 25))
(1 行)
```

図示すると、次のようなかんじです。

![ST_StraightSkeletonの結果](https://storage.googleapis.com/zenn-user-upload/l21diuwicu4zxischsz076sr2zzo)

ここで、藍色に塗りつぶされた面が元のポリゴン、オレンジの線がST_StraightSkeletonで出されたマルチラインストリングです。

しかし http://en.wikipedia.org/wiki/Straight_skeleton を見ると、ポリゴンになってるのが気になる。

## ST_Orientation
ポリゴンがどちら回りかを見ます。右回りなら正数、左回りなら負数のようです。

```
db=# SELECT ST_Orientation('POLYGON((0 0, 10 10, 10 0, 0 0))'::GEOMETRY);
 st_orientation 
----------------
              1
(1 行)

db=# SELECT ST_Orientation('POLYGON((0 0, 10 0, 10 10, 0 0))'::GEOMETRY);
 st_orientation 
----------------
             -1
(1 行)

```

## ST_ForceLHR
右回りのポリゴンを左回りに強制します。
次の実行例のうち、ひとつ目では左回りポリゴンを引数にし、ふたつ目では右回りポリゴンを引数にしています。

```
db=# SELECT ST_AsText(ST_ForceLHR('POLYGON((0 0, 10 0, 10 10, 0 0))'::GEOMETRY));
           st_astext           
-------------------------------
 POLYGON((0 0,10 0,10 10,0 0))
(1 行)

db=# SELECT ST_AsText(ST_ForceLHR('POLYGON((0 0, 10 10, 10 0, 0 0))'::GEOMETRY));
           st_astext           
-------------------------------
 POLYGON((0 0,10 0,10 10,0 0))
(1 行)
```

## ST_MinkowskiSum
ミンコフスキー和を求めます。ミンコフスキー和についてはググって下さい。

```
db=# SELECT ST_AsText(ST_MinkowskiSum(
  'POLYGON((0 0, 20 0, 0 20, 0 0))'::GEOMETRY,
  'POLYGON((30 30, 50 30, 50 50, 30 50, 30 30))'::GEOMETRY
));
                             st_astext                             
-------------------------------------------------------------------
 MULTIPOLYGON(((30 30,50 30,70 30,70 50,50 70,30 70,30 50,30 30)))
(1 行)
```

図で示してみます。第1引数が水色の三角形、第2引数が赤い四角形です。

![ミンコフスキー和の引数](https://storage.googleapis.com/zenn-user-upload/o50e7d9eampo1zkn4gdso27vuxer)

結果は次のようになります。

![ミンコフスキー和の結果](https://storage.googleapis.com/zenn-user-upload/ddvrq2ayxji4z0l8r0bv21takn13)

### ミンコフスキー和

http://slis.tsukuba.ac.jp/~fujisawa.makoto.fu/lecture/iml/text/3_collision.html にミンコフスキー和について書いてくれています。

ミンコフスキー和は、一方の図形を握って、もう一方の図形の内部をくまなく回した軌跡の集合、と言えるかなと思います。

一方を原点を中心に点対称の図形に変え（X,Yともに正負裏返す）てミンコフスキー和をとると、ミンコフスキー差が取れます。ミンコフスキー差の結果集合が原点を含むと、もとの画像は衝突しているのだそうです。

## ST_IsPlanar

平面(planar)であるか否かをテストします。

「平面」の条件は、全ての点が同一線上にあるようなことになっていない、と言えます。よって、妥当な(ST_IsValid()でTrueを返す)ポリゴンは必ず平面です。また、自己交差または自己接触するポリゴンは、妥当ではないですが平面です。

まず妥当なポリゴンの例です。

```
db=# SELECT ST_IsPlanar('POLYGON((0 0, 2 0, 2 2, 0 0))'::GEOMETRY);
 st_isplanar 
-------------
 t
(1 行)

db=# SELECT ST_IsValid('POLYGON((0 0, 2 0, 2 2, 0 0))'::GEOMETRY);
 st_isvalid 
------------
 t
(1 行)
```

妥当でないポリゴンの例を次に示します。

```
db=# SELECT ST_IsPlanar('POLYGON((0 0, 1 1, 2 2, 0 0))'::GEOMETRY);
 st_isplanar 
-------------
 f
(1 行)

db=# SELECT ST_IsValid('POLYGON((0 0, 1 1, 2 2, 0 0))'::GEOMETRY);
NOTICE:  Self-intersection at or near point 0 0
 st_isvalid 
------------
 f
(1 行)
```

最後に、平面だけれども妥当でない例を示します。

```
db=# SELECT ST_IsPlanar('POLYGON((0 0, 2 0, 2 2, 1 0, 0 0))'::GEOMETRY);
 st_isplanar 
-------------
 t
(1 行)

db=# SELECT ST_IsValid('POLYGON((0 0, 2 0, 2 2, 1 0, 0 0))'::GEOMETRY);
NOTICE:  Self-intersection at or near point 0 0
 st_isvalid 
------------
 f
(1 行)
```

## ST_Tesselate

ポリゴンや多面体サーフェス等を三角形に分割します。

```
db=# SELECT ST_AsText(ST_Tesselate(ST_Extrude('POLYGON((0 0, 10 0, 10 10, 0 0))'::GEOMETRY, 0,0,20)));
                                                                                                                                                  st_astext                                                                                                                                                  
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 GEOMETRYCOLLECTION Z (TIN Z (((0 0 0,10 10 0,10 0 0,0 0 0)),((0 0 20,10 0 20,10 10 20,0 0 20)),((10 10 0,0 0 20,10 10 20,10 10 0)),((10 10 0,0 0 0,0 0 20,10 10 0)),((10 10 0,10 10 20,10 0 20,10 10 0)),((10 0 0,10 10 0,10 0 20,10 0 0)),((10 0 0,10 0 20,0 0 20,10 0 0)),((0 0 0,10 0 0,0 0 20,0 0 0))))
(1 行)
```

## ST_3DIntersection

3次元のインタセクション(共通領域)を計算します。

返り値がTIN(三角形)の集合になっているところは、結構読みづらくなります。もっとも、三次元を扱うんだったらサーフェスをTINに分けてしまいますよね、と。

```
db=# SELECT ST_AsText(ST_3DIntersection(
  ST_Extrude('POLYGON((0 0, 2 0, 0 2, 0 0))'::GEOMETRY, 0, 0, 1),
  ST_Extrude('POLYGON((0 0, 2 2, 0 2, 0 0))'::GEOMETRY, 0, 0, 1)
));
                                                                                                                                                           st_astext                                                                                                                                                           
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 POLYHEDRALSURFACE Z (((1 1 0,0 0 0,0 2 0,1 1 0)),((0 2 0,1 1 0.5,1 1 0,0 2 0)),((0 0 0,0 2 1,0 2 0,0 0 0)),((1 1 0,1 1 0.5,0 0 0,1 1 0)),((0 2 0,0 2 1,1 1 0.5,0 2 0)),((0 0 0,0 0 1,0 2 1,0 0 0)),((1 1 0.5,0 0 1,0 0 0,1 1 0.5)),((0 2 1,1 1 1,1 1 0.5,0 2 1)),((0 0 1,1 1 1,0 2 1,0 0 1)),((1 1 0.5,1 1 1,0 0 1,1 1 0.5)))
(1 行)

```

## ST_3DArea

ポリゴンの面積を求めます。ST_Area()との違いを味わって下さい。

```
db=# SELECT ST_3DArea('POLYGONZ((0 0 0, 2 0 1, 2 2 0, 0 0 0))'::GEOMETRY);
    st_3darea     
------------------
 2.44948974278318
(1 行)

db=# SELECT ST_Area('POLYGONZ((0 0 0, 2 0 1, 2 2 0, 0 0 0))'::GEOMETRY);
 st_area 
---------
       2
(1 行)
```

## ST_MakeSolid

内部使用インスタンスに「立体」フラグを付けます。(https://postgis.net/docs/doxygen/2.2/da/d9e/postgis_2lwgeom__sfcgal_8c_a0741e0aa4cf04ff6b6671db1877cbe1f.html)

本当にフラグを付けるだけで、他には何もしません。
また、このフラグはWKB等には出てこないものです。

## ST_IsSolid

内部使用インスタンスに「立体」フラグが付いていかどうかを返します。

次は、1面だけの多面体サーフェス（立体とは到底言えない）についてST_IsSolidを実行しています。

```
db=# SELECT ST_IsSolid(
  'POLYHEDRALSURFACE Z (
  ((0 0 0,0 1 0,1 1 0,1 0 0,0 0 0))
  )'::geometry
);
 st_issolid 
------------
 f
(1 行)

db=# SELECT ST_IsSolid(
  ST_MakeSolid(
    'POLYHEDRALSURFACE Z (
    ((0 0 0,0 1 0,1 1 0,1 0 0,0 0 0))
    )'::geometry
  )
);
 st_issolid 
------------
 t
(1 行)
```

ここから見て分かる通り``ST_MakeSolid``でフラグが立っているかどうかだけで判定しています。

立体らしく正六面体をなす多面体サーフェスでも見てみましょう。

```
db=# SELECT ST_IsSolid(
  'POLYHEDRALSURFACE Z (
  ((0 0 0,0 1 0,1 1 0,1 0 0,0 0 0)),
  ((0 0 1,1 0 1,1 1 1,0 1 1,0 0 1)),
  ((0 0 0,0 0 1,0 1 1,0 1 0,0 0 0)),
  ((0 1 0,0 1 1,1 1 1,1 1 0,0 1 0)),
  ((1 1 0,1 1 1,1 0 1,1 0 0,1 1 0)),
  ((1 0 0,1 0 1,0 0 1,0 0 0,1 0 0)))'::geometry
);
 st_issolid 
------------
 f
(1 行)

db=# SELECT ST_IsSolid(
  ST_MakeSolid(
    'POLYHEDRALSURFACE Z (
    ((0 0 0,0 1 0,1 1 0,1 0 0,0 0 0)),
    ((0 0 1,1 0 1,1 1 1,0 1 1,0 0 1)),
    ((0 0 0,0 0 1,0 1 1,0 1 0,0 0 0)),
    ((0 1 0,0 1 1,1 1 1,1 1 0,0 1 0)),
    ((1 1 0,1 1 1,1 0 1,1 0 0,1 1 0)),
    ((1 0 0,1 0 1,0 0 1,0 0 0,1 0 0)))'::geometry
  )
);
 st_issolid 
------------
 t
(1 行)
```

正六面体であろうとも、ST_MakeSolidで立体フラグが立っているかどうかだけで判定されていることがお分かりいただけるかと思います。

# 今回のまとめ

PostGISからSFCGALを使用できるようになりました。そこで、インストール方法をほんの少しだけ示し、SFCGALを使用するPostGIS関数群について紹介しました。
PostGIS 2.2 では、これで全てです。はっきり言って少ないです。ただ、押し出しの際に右ねじにするよう気をつけてくれてたり、三次元のインタセクション等が実現でき、秘めているものは大きいと思います。CGALにはボロノイ図を作ったりする機能があるようですので、そういったところがPostGISから使えるようになってくると、末恐ろしいことになりそうです。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。
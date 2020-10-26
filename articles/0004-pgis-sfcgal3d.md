---
title: "PostGIS+SFCGALで3次元幾何演算を行う"
emoji: "😀"
type: "tech"
topics: [PostGIS]
published: false
---
# はじめに

PostGISにSFCGALを使用するエクステンションがありますが、PostGIS 2.2.0から、SFCGALを使って、3次元ジオメトリの結び(ST_3DUnion)、交わり(ST_3DIntersection)、差分(ST_3DDifference)が実装されています。

今回は、3次元の結び、交わり、差分を実際に実行してみましょう。

## 必要なもの
PostGIS 2.2以上で、コンフィギュレーション時にSFCGAL対応にしている必要があります。

## 下準備
PostGISエクステンションだけでなく、"postgis_sfcgal"エクステンションも入れる必要があります。

```psql
db=# CREATE EXTENSION postgis;
db=# CREATE EXTENSION postgis_sfcgal;
```

## HTMLとして出力します

PostGISは3次元ジオメトリをX3Dで出力する関数がありますが、ジオメトリに相当する要素しか出ません。本稿では、``ST_AsX3D()``の前後に文字列リテラルを付けてTML文書にしています。

# 二つのジオメトリ

ひとつは、最小値が(-1,-1,-1)で最大値が(1,1,1)となる、立方体です。1辺の長さは2で、重心は原点にあります。

```sql
ST_Extrude('POLYGON Z((-1.0 -1.0 -1.0, -1.0 1.0 -1.0, 1.0 1.0 -1.0, 1.0 -1.0 -1.0, -1.0 -1.0 -1.0))'::GEOMETRY, 0, 0, 2)
```

もうひとつは、円柱です。(0,0,-0.5)を中心にした、XY平面に平行な半径0.4の円(正確には近似円)を作り、上向きに2単位押し上げています。

```sql
ST_Extrude(ST_Buffer('POINT Z(0 0 -0.5)'::GEOMETRY, 0.4), 0, 0, 2)
```

円柱は立方体に突き刺さっていて、かつ一部が立方体の外に出ています。

# ST_3DUnion

まずは結びです。足し算みたなものです。

```psql
--
-- psql -t -A -d (dbname) -f (sqlfile) -o (outfile)
--
SELECT '<!DOCTYPE html>
<html>
<head>
<meta http-equiv="X-UA-Compatible" content="IE=edge"/> 
<script type="text/javascript" src="http://www.x3dom.org/download/x3dom.js"></script> 
<link rel="stylesheet" type="text/css" href="http://www.x3dom.org/download/x3dom.css">
</head>
<body>
<X3D width="600px" height="600px">
  <Scene>
    <Transform>
      <Shape>
        <Appearance>
          <Material emissiveColor=''0 0 1''/>   
        </Appearance> 
        ' || ST_AsX3D(
ST_3DUnion(
  ST_Extrude('POLYGON Z((-1.0 -1.0 -1.0, -1.0 1.0 -1.0, 1.0 1.0 -1.0, 1.0 -1.0 -1.0, -1.0 -1.0 -1.0))'::GEOMETRY, 0, 0, 2)
  ,
  ST_Extrude(ST_Buffer('POINT Z(0 0 -0.5)'::GEOMETRY, 0.4), 0, 0, 2)
)
      ) || '
      </Shape>
    </Transform>
  </Scene>
</X3D>
</body>
</html>
' As x3ddoc;
```

![ST_3DUnionの結果](https://storage.googleapis.com/zenn-user-upload/en011kad24l35dhe4cgb7seltj0y)

# 交わり

共通部分を抽出します。

```sql
--
-- psql -t -A -d (dbname) -f (sqlfile) -o (outfile)
--
SELECT '<!DOCTYPE html>
<html>
<head>
<meta http-equiv="X-UA-Compatible" content="IE=edge"/> 
<script type="text/javascript" src="http://www.x3dom.org/download/x3dom.js"></script> 
<link rel="stylesheet" type="text/css" href="http://www.x3dom.org/download/x3dom.css">
</head>
<body>
<X3D width="600px" height="600px">
  <Scene>
    <Transform>
      <Shape>
        <Appearance>
          <Material emissiveColor=''0 0 1''/>   
        </Appearance> 
        ' || ST_AsX3D(
ST_3DIntersection(
  ST_Extrude('POLYGON Z((-1.0 -1.0 -1.0, -1.0 1.0 -1.0, 1.0 1.0 -1.0, 1.0 -1.0 -1.0, -1.0 -1.0 -1.0))'::GEOMETRY, 0, 0, 2)
  ,
  ST_Extrude(ST_Buffer('POINT Z(0 0 -0.5)'::GEOMETRY, 0.4), 0, 0, 2)
)
      ) || '
      </Shape>
    </Transform>
  </Scene>
</X3D>
</body>
</html>
' As x3ddoc;
```

![ST_3DIntersectionの結果](https://storage.googleapis.com/zenn-user-upload/w4jrwle46r14ts763ohok7r66n2e)

これはちょっと見えにくいと思いますが、円柱のうち立方体内に潜っている部分です。

# 差分

引き算みたいなものです。

```sql
--
-- psql -t -A -d (dbname) -f (sqlfile) -o (outfile)
--
SELECT '<!DOCTYPE html>
<html>
<head>
<meta http-equiv="X-UA-Compatible" content="IE=edge"/> 
<script type="text/javascript" src="http://www.x3dom.org/download/x3dom.js"></script> 
<link rel="stylesheet" type="text/css" href="http://www.x3dom.org/download/x3dom.css">
</head>
<body>
<X3D width="600px" height="600px">
  <Scene>
    <Transform>
      <Shape>
        <Appearance>
          <Material emissiveColor=''0 0 1''/>   
        </Appearance> 
        ' || ST_AsX3D(
ST_3DDifference(
  ST_Extrude('POLYGON Z((-1.0 -1.0 -1.0, -1.0 1.0 -1.0, 1.0 1.0 -1.0, 1.0 -1.0 -1.0, -1.0 -1.0 -1.0))'::GEOMETRY, 0, 0, 2)
  ,
  ST_Extrude(ST_Buffer('POINT Z(0 0 -0.5)'::GEOMETRY, 0.4), 0, 0, 2)
)
      ) || '
      </Shape>
    </Transform>
  </Scene>
</X3D>
</body>
</html>
' As x3ddoc;
```

![ST_3DDifferenceの結果](https://storage.googleapis.com/zenn-user-upload/olw6nfozdvstomn2uj11dhamc7d5)

立方体に、円柱（の一部）で穴を開けたようになっているのが見えます。

# おわりに

PostGIS 2.2で、3次元ジオメトリの交わり、結び、差分ができることを、ビジュアル的に示しました。

基本的な幾何演算ができるようになり、本格的な3次元が始まるかも知れません。地形データからトンネルをズドーンと掘るなんていうのはお手の物です。3D CAD的な使い方が可能になったので、建物データの保存、修正や、3次元空間クエリを投げるのも可能ですね。

ただ、やってみるとわかると思いますが、ちょっと時間がかかります。特に円柱が重いです。実際には曲線は直線にしていて、円柱は多角形柱になっているためだろうと思います。
とはいえ、文句言うなら曲線を含むジオメトリを自前で処理してみろよと言われたら確実に逃げます。なのでありがたく使うべきで、速度や曲線・曲面対応はは今後のことかなと思います。

それと、簡易的なものでなく本格的なビューアと、GUIベースのエディタが必要になってくると思いますが、この問題からも私は逃亡したいと思います。

# 本記事のライセンス


![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

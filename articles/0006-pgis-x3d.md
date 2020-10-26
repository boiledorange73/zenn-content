---
title: "PostGISの3次元ジオメトリをブラウザで見てみよう"
emoji: "😀"
type: "tech"
topics: [PostGIS]
published: false
---
# はじめに

3次元ジオメトリは、結果をWKTで見ててもピンと来ません。やはり3次元ジオメトリは3次元で表示するべきです。

そこで、3次元ジオメトリをX3Dで出力したり、HTMLに出してブラウザで見たりする方法についてご紹介します。

# ST_AsX3D

これを、ST_AsX3Dによって、ジオメトリを引数に取り、X3Dの「断片」を得ることができます。

```psql
db=# SELECT ST_AsX3D(ST_Extrude('POLYGON Z((-1.0 -1.0 -1.0, -1.0 1.0 -1.0, 1.0 1.0 -1.0, 1.0 -1.0 -1.0, -1.0 -1.0 -1.0))'::GEOMETRY, 0, 0, 2));
                                                                                                                                                                st_asx3d                                                                                                                                                                
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 <IndexedFaceSet  coordIndex='0 1 2 3 -1 4 5 6 7 -1 8 9 10 11 -1 12 13 14 15 -1 16 17 18 19 -1 20 21 22 23'><Coordinate point='-1 -1 -1 -1 1 -1 1 1 -1 1 -1 -1 -1 -1 1 1 -1 1 1 1 1 -1 1 1 -1 -1 -1 -1 -1 1 -1 1 1 -1 1 -1 -1 1 -1 -1 1 1 1 1 1 1 1 -1 1 1 -1 1 1 1 1 -1 1 1 -1 -1 1 -1 -1 1 -1 1 -1 -1 1 -1 -1 -1' /></IndexedFaceSet>
(1 行)

```

これに定まった文字列を前後に追加することで、完全なX3D文書になります。

# 完全なX3D文書

次の立方体をX3D文書にしてみます。

```sql
POLYHEDRALSURFACE Z (
  ((-1 -1 -1, -1  1 -1,  1  1 -1,  1 -1 -1, -1 -1 -1)),
  ((-1 -1  1,  1 -1  1,  1  1  1, -1  1  1, -1 -1  1)),
  ((-1 -1 -1, -1 -1  1, -1  1  1, -1  1 -1, -1 -1 -1)),
  ((-1  1 -1, -1  1  1,  1  1  1,  1  1 -1, -1  1 -1)),
  (( 1  1 -1,  1  1  1,  1 -1  1,  1 -1 -1,  1  1 -1)),
  (( 1 -1 -1,  1 -1  1, -1 -1  1, -1 -1 -1,  1 -1 -1))
)
```

SQLは次のようになります。

```sql
--
-- 1.sql
-- psql -t -A -d (dbname) -f (sqlfile) -o (outfile)
--
SELECT '<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE X3D PUBLIC "ISO//Web3D//DTD X3D 3.0//EN" "http://www.web3d.org/specifications/x3d-3.0.dtd">
<X3D>
  <Scene>
    <Transform>
      <Shape>
        <Appearance>
          <Material emissiveColor=''0 0 1''/>   
        </Appearance> 
        ' || ST_AsX3D(
-- begin editable
'POLYHEDRALSURFACE Z (
  ((-1 -1 -1, -1  1 -1,  1  1 -1,  1 -1 -1, -1 -1 -1)),
  ((-1 -1  1,  1 -1  1,  1  1  1, -1  1  1, -1 -1  1)),
  ((-1 -1 -1, -1 -1  1, -1  1  1, -1  1 -1, -1 -1 -1)),
  ((-1  1 -1, -1  1  1,  1  1  1,  1  1 -1, -1  1 -1)),
  (( 1  1 -1,  1  1  1,  1 -1  1,  1 -1 -1,  1  1 -1)),
  (( 1 -1 -1,  1 -1  1, -1 -1  1, -1 -1 -1,  1 -1 -1))
)'::GEOMETRY
-- end editable
      ) || '
      </Shape>
    </Transform>
  </Scene>
</X3D>' As x3ddoc;
```

これをpsqlを使って実行して、ファイルを作成します。

次の例は、1.sqlに上記SQLを保存した場合です。

```csh
% psql -t -A -d db -f 1.sql -o 1.x3d
```

これにより、``1.x3d``というファイルが作成されます。中身は次のようになります。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE X3D PUBLIC "ISO//Web3D//DTD X3D 3.0//EN" "http://www.web3d.org/specifications/x3d-3.0.dtd">
<X3D>
  <Scene>
    <Transform>
      <Shape>
        <Appearance>
          <Material emissiveColor='0 0 1'/>   
        </Appearance> 
        <IndexedFaceSet  coordIndex='0 1 2 3 -1 4 5 6 7 -1 8 9 10 11 -1 12 13 14 15 -1 16 17 18 19 -1 20 21 22 23'><Coordinate point='-1 -1 -1 -1 1 -1 1 1 -1 1 -1 -1 -1 -1 1 1 -1 1 1 1 1 -1 1 1 -1 -1 -1 -1 -1 1 -1 1 1 -1 1 -1 -1 1 -1 -1 1 1 1 1 1 1 1 -1 1 1 -1 1 1 1 1 -1 1 1 -1 -1 1 -1 -1 1 -1 1 -1 -1 1 -1 -1 -1' /></IndexedFaceSet>
      </Shape>
    </Transform>
  </Scene>
</X3D>
```

もし、psql起動時にコマンドオプションが指定できない場合には、いちどpsqlを起動してから、その中で指定していきます。

* ``-t``は``\t on``に相当します
* ``-A``は``\a``に相当します (トグル型ですので不用意に実行すると元に戻るときがあります)
* ``-o (パス)``は``\o '(パス)'`に相当します。Windowsの場合はパスセパレータの'\'がエスケープされるので'\\'にするようにします。
* ``-f (パス)``は``\i '(パス)'``に相当します。

# X3DOMを使う

X3DOM ( http://www.x3dom.org/ )は、HTML文書にX3Dを埋め込むと、それを読み取って表示してくれるJavaScriptです。サンプルを見ると、jsファイルとcssファイルとを各1個ずつ取り込む必要があります。

X3D文書と全く違いますので、前後に全く違う文字列を追加する必要があります。次のようになります。

```sql
--
-- 2.sql
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
-- begin editable
'POLYHEDRALSURFACE Z (
  ((-1 -1 -1, -1  1 -1,  1  1 -1,  1 -1 -1, -1 -1 -1)),
  ((-1 -1  1,  1 -1  1,  1  1  1, -1  1  1, -1 -1  1)),
  ((-1 -1 -1, -1 -1  1, -1  1  1, -1  1 -1, -1 -1 -1)),
  ((-1  1 -1, -1  1  1,  1  1  1,  1  1 -1, -1  1 -1)),
  (( 1  1 -1,  1  1  1,  1 -1  1,  1 -1 -1,  1  1 -1)),
  (( 1 -1 -1,  1 -1  1, -1 -1  1, -1 -1 -1,  1 -1 -1))
)'::GEOMETRY
-- end editable
      ) || '
      </Shape>
    </Transform>
  </Scene>
</X3D>
</body>
</html>
' As x3ddoc;
```

```csh
% psql -tA -d db -f 2.sql -o 2.html
```

これで生成された2.htmlをブラウザで見てみます。

![真上から見た立方体](https://storage.googleapis.com/zenn-user-upload/a5d741idyyv5eyhfccdaee2rk9q0)

2次元の四角形に見えますが、ドラッグすると、立方体であることが分かります。

![斜めから見た立方体](https://storage.googleapis.com/zenn-user-upload/37r1kxed5n0vu79uby6avypdawnh)

# おわりに

今回は、X3D文書の出力とX3DOMを使ったHTML文書の出力についてご紹介しました。

WKTで見て丹念に追いかけていくと把握できることもありますが、立方体（6面）を超えてくると、ちょっと厳しくなると思います。

そういう場合は、やはりこちらの方が一目で分かることができるので、便利だと思います。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

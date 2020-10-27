---
title: "PostGISユーザがSQL Server 2017の空間機能を少し触ってみた"
emoji: "😀"
type: "tech"
topics: [GIS, SQLServer]
published: true
---
# はじめに

[RDBMS-GIS(MySQL,PostgreSQLなど) Advent Calendar 2018](https://qiita.com/advent-calendar/2018/rdbms_gis)は、RDBMSの地理空間機能ならOKということで、でもたぶんMySQLとかPostGISとかが例に挙がると思いますが、まさかのT-SQLです。

ちょっと触ってみたので、ここにメモを残します。行くべきか引き返すべきかは各自判断して下さい。

# 思ったことを実行していく

## ジオメトリの生成

ジオメトリの生成は``geometry::STGeomFromText()``でやるのだそうです。

```sql
SELECT geometry::STGeomFromText('POINT(135 35)', 4326);
go

--
0x

(1 行処理されました)
```

実行できたけど、ちゃんとできたかよく分からん。

## STAsTextの実行

``ST_AsText()``でWKTが出るはずなのでやってみるけどその前に``STAsText()``ですからね、注意して下さい。

```sql
SELECT STAsText(geometry::STGeomFromText('POINT(135 35)', 4326));
go
メッセージ 195、レベル 15、状態 10、サーバー ...\SQLEXPRESS、行 1
'STAsText' は 組み込み関数名 として認識されません。
1>
```

無いんだそうだ…いや、そんなわけない。

```sql
SELECT geometry::STGeomFromText('POINT(135 35)', 4326).STAsText();
go
                                                                                                                                                                                                                                                      
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
POINT (135 35)                                                                                                                                                                                                                                        

(1 行処理されました)
```

うーん、geometry型のメソッドという扱いですね。そういう設計なんですね（たじたじ）。

## 横着して全部小文字にしたら失敗した

前後するのですが、次のようなクエリを投げてエラーが返ってきました。

```sql
SELECT geometry::STGeomFromText('POINT(135 35)', 4326).stastext();
go
メッセージ 6506、レベル 16、状態 10、サーバー ...\SQLEXPRESS、行 1
型 'Microsoft.SqlServer.Types.SqlGeometry' のメソッド 'stastext' がアセンブリ 'Microsoft.SqlServer.Types' に見つかりません でした
```

問題は``stastext()``です。``STAsText()``としていないためのエラーです。大文字小文字の区別があるので気を付けてください。

## UNION (結合)をやってみる

``STUNion()``は、geometryのメソッドです。集約関数には``UnionAggregate()``があるようですが、試していません。


```sql
SELECT (
  geometry::STGeomFromText('POLYGON((1 1, 2 1, 2 2, 1 2, 1 1))',0)
).STUnion(
  geometry::STGeomFromText('POLYGON((1 0, 2 0, 2 1, 1 1, 1 0))',0)
).STAsText();
go
                                                                                                                                                                                                                                                
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
POLYGON ((1 0, 2 0, 2 1, 2 2, 1 2, 1 1, 1 0))                                                                                                                                                                                                   

(1 行処理されました)
```

後ろにメソッドを付けていくのが、PostGISユーザから見ると許しがたいけど、慣れるとjQueryとかやってるみたいで意外と気持ちいいです（個人の感想です）。

同じようなクエリをPostGISでも実行してみました。

```sql
db=# SELECT ST_AsText(
  ST_Union(
    'POLYGON((1 1, 2 1, 2 2, 1 2, 1 1))'::geometry,
    'POLYGON((1 0, 2 0, 2 1, 1 1, 1 0))'::geometry
  )
);

               st_astext                
----------------------------------------
 POLYGON((1 1,1 2,2 2,2 1,2 0,1 0,1 1))
(1 行)
```

うん、これは同じ結果になりましたね。

## ジオグラフィで距離を計測する。

まず、ジオグラフィはgeographyクラスであることに注意して下さい。そして``STGeomFromText()``で生成することには、さらなる注意が必要です。``ST_GeomFromText()``でない、と。

距離計測は、geographyのメソッドに``STDistance``があります。2回目なのでもう慣れましたね？

```sql
SELECT (
  geography::STGeomFromText('POINT(135 35)', 4326)
).STDistance(
  geography::STGeomFromText('POINT(136 36)', 4326)
);
go

------------------------
      143321.57798743498

(1 行処理されました)
```

実行できました。

PostGISでは次のようになります。

```sql
db=# SELECT ST_Distance(
  'SRID=4326;POINT(135 35)'::GEOGRAPHY,
  'SRID=4326;POINT(136 36)'::GEOGRAPHY
);
   st_distance   
-----------------
 143321.57818178
(1 行)
```

ま、大丈夫ですね。

あと、ここで、よいこのみんな、もとい、あやしいおともだちは、Axis Orderが気になりますよね？さっきのクエリでは経度緯度で通ってるけど、逆にしたらどうなるだろう、って。

```sql
SELECT (
  geography::STGeomFromText('POINT(35 135)', 4326)
).STDistance(
  geography::STGeomFromText('POINT(36 136)', 4326)
);
go

メッセージ 6522、レベル 16、状態 1、サーバー ...\SQLEXPRESS、行 1
ユーザー定義のルーチンまたは集計 "geography" を実行中に .NET Framework エラーが発生しました:
System.FormatException: 24201: 緯度の値は -90 ～ 90 度の範囲で指定する必要があります。
System.FormatException:
   場所 Microsoft.SqlServer.Types.GeographyValidator.ValidatePoint(Double x, Double y, Nullable`1 z, Nullable`1 m)
   場所 Microsoft.SqlServer.Types.Validator.BeginFigure(Double x, Double y, Nullable`1 z, Nullable`1 m)
   場所 Microsoft.SqlServer.Types.ForwardingGeoDataSink.BeginFigure(Double x, Double y, Nullable`1 z, Nullable`1 m)
   場所 Microsoft.SqlServer.Types.CoordinateReversingGeoDataSink.BeginFigure(Double x, Double y, Nullable`1 z, Nullable`1 m)
   場所 Microsoft.SqlServer.Types.WellKnownTextReader.ParsePointText(Boolean parseParentheses)
   場所 Microsoft.SqlServer.Types.WellKnownTextReader.ParseTaggedText(OpenGisType type)
   場所 Microsoft.SqlServer.Types.WellKnownTextReader.Read(OpenGisType type, Int32 srid)
   場所 Microsoft.SqlServer.Types.SqlGeography.ParseText(OpenGisType type, SqlChars taggedText, Int32 srid)
   場所 Microsoft.SqlServer.Types.SqlGeography.GeographyFromText(OpenGisType type, SqlChars taggedText, Int32 srid)
。
```

ということで、経度-緯度 (東西-南北)のオーダーです。

## Tokyoは？

```sql
SELECT (
  geography::STGeomFromText('POINT(135 35)', 4301)
).STDistance(
  geography::STGeomFromText('POINT(136 36)', 4301)
);
go

------------------------
      143305.61766686899

(1 行処理されました)
```

PostGISでは次のようになりました。

```sql
db=# SELECT ST_Distance('SRID=4301;POINT(135 35)'::GEOGRAPHY,
'SRID=4301;POINT(136 36)'::GEOGRAPHY);
   st_distance   
-----------------
 143305.61785684
(1 行)
```

うん、たぶん、回転楕円体がちゃんと変更されてるっぽですね。

## 空間参照系テーブルはどこ？

若干さがしましたが、なんとか発見。``sys.spatial_reference_systems``です。

```sql
SELECT * FROM sys.spatial_reference_systems;
```

これは、``sqlcmd``でなくManagement Studioで実行することをお勧めします。エントリが「それなりに」多いですから。

## これが空間参照系テーブルなのか？

見て頂くと分かると思いますが、4000番台だけです。

TSQLは``STTransform()``を持っていないので、使用局面は、``geography#STDistance()``での準拠回転楕円体面等の情報を取得するだけでないかと思われます。だとすると、4000番台はWGS84地理座標系は4326、JGD2000地理座標系は4612といったように、地理座標系が集まるところなので、必要ないといえば必要ないです。

いや、そういうわけにはいかない。

JGD2011は6667ですね。4000番台じゃないので、これはまずいのではないか？

ドキドキしながら、次のクエリを実行します。

```sql
SELECT (
  geography::STGeomFromText('POINT(135 35)', 6667)
).STDistance(
  geography::STGeomFromText('POINT(136 36)', 6667)
);
go

メッセージ 6522、レベル 16、状態 1、サーバー ...\SQLEXPRESS、行 1
ユーザー定義のルーチンまたは集計 "geography" を実行中に .NET Framework エラーが発生しました:
System.ArgumentException: 24204: SRID (spatial reference identifier) が無効です。指定する SRID は、sys.spatial_reference_systems カタログ ビューに表示されたサポートされている SRID の 1 つと一致する必要があります。
System.ArgumentException:
   場所 Microsoft.SqlServer.Types.SqlGeography.set_Srid(Int32 value)
   場所 Microsoft.SqlServer.Types.SqlGeography..ctor(GeoData g, Int32 srid)
   場所 Microsoft.SqlServer.Types.SqlGeography.GeographyFromText(OpenGisType type, SqlChars taggedText, Int32 srid)
。
```

だめでした。

# 気付いた「クセ」

ここまでのをPostGISユーザから見た差異を上げてみます。

* 関数名は大文字小文字の区別あり
* ``geometry``, ``geography``の静的メソッドでインスタンス生成
* 関数名等のプリフィックスは``ST`` (``ST_``でない)
* 第1引数に``geometry``, ``geography``を取る関数は、インスタンスのメソッドとして実行
* ``STTransform()``が無い等機能が非常に少ない
* 集約関数は静的メソッドとして別途用意 (PostGISの``ST_Union(geometry set)``はTSQLでは``UnionAggregate()``)
* Axis Orderは東西-南北の順で固定（MySQLとは違う）
* 空間参照系テーブルは``sys.spatial_reference_systems``
* SRIDは実質的にはgeographyのみ使用
* spatial_refernce_systemsに登録されているのはSRID 4000番台のみ
* JGD 2011地理座標系は非対応

# おわりに

まあそういうことです。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

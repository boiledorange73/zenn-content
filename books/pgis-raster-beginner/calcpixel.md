---
title: "ピクセル演算をやってみよう"
---
# はじめに

[指定位置の標高を得るクエリとインデックス](pickvalue)の前半では、ピクセル値の取得を行いました。今度は、ピクセル値を近隣ピクセルから独立して演算を行い、その結果から新しいラスタを生成します。

## 結構遅い点にご注意

今回の処理はSQL一文で済みますが、その割に処理時間がかかります。現時点では、まだちょっと実用的な速度に達していないと言えます。


# DEMテーブルがあるとします

ここでは、``dem``という名前のテーブルがあり、ラスタが一つ格納されている状態であると仮定しています。格納については、[PostGISラスタにGeoTIFFデータを格納する](raster2pgsql)を参照して下さい。

# 標高10m以上かそうでないかのラスタを作る

浸水域の計算は標高データを用います。実際にはシミュレーション計算を行うので、ここでご紹介する方法で浸水危険区域を決定することはありませんが、浸水危険区域を見積もり際に、指定した標高以上である土地の分布を調べることは重要です。せっかく標高 (DEM)データがあるので、これを使って演算をやってみましょう。今回の条件は標高10m以上かそうでないかを示すラスタを生成します。

## テーブルの作成

``z10``という名前のテーブルを作成します。ここでは、実行してはその都度消すことにしています。

```sql
CREATE TABLE z10 (
  rid SERIAL PRIMARY KEY,
  rast RASTER
);
CREATE INDEX ix_z10_rast ON z10 USING GiST (ST_ConvexHull(rast));
```

## まずはピクセルタイプは気にしないで演算

ピクセルタイプとは、ピクセルの数値型を指します。数値型には、1ビット符号なし整数、8ビット符号付き整数、32ビット浮動小数点数といったもので、ビット数および、整数（符号の有無を含む）ならびに浮動小数点数の別からなります。

ピクセルタイプはバンドごとに決まっていますが、ラスタ内でもバンドが異なるとピクセルタイプが違ってもかまいません。

詳細は[PostGISラスタの仕様](pgisraster)をご覧下さい。

``dem``テーブルのピクセルタイプは、32ビット浮動小数点数です。

ともかく、次のクエリを実行してみましょう。

```sql
DELETE FROM z10;

INSERT INTO z10 (rid, rast)
SELECT rid, ST_MapAlgebraExpr(
  rast,
  NULL,
  'CASE WHEN [rast]::NUMERIC >= 10 THEN 1 ELSE 0 END'
)
FROM dem;
```

``dem``のラスタから、第三引数で指定した数式にあった値に変換したラスタを生成しています。

ここで注意して頂きたいのは、新しいラスタを生成している点です。引数で与えられたラスタは変更されません。

## QGISで見てみる

QGISで様子を見てみましょう。次のようになります。

![0と1を白黒で表示した場合](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-raster-beginner/calcpixel/2-1.png)


ちょっと分かりにくいですね。

## QGISで色を変えてみる

色を変えてみましょう。対象レイヤを右クリックしてコンテキストメニューを出し、「レイヤのプロパティ」を選択します。

![「レイヤのプロパティ」を選択しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-raster-beginner/calcpixel/2-2.png)

「レイヤプロパティ」ダイアログで「シンボロジ」タブを選びます。次いで「グラデーション」で「単バンド疑似カラー」を選択します。

![「単バンド疑似カラー」を選択しているところ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-raster-beginner/calcpixel/2-3.png)

「分類」ボタンをクリックすると、値に基づく色の塗分け設定が作成されます。

![「分類」ボタンをクリックした後の状況](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-raster-beginner/calcpixel/2-4.png)

今回は、0と1しかないので2色が必要です。対してそのままだと5色に分かれますが、無視して下さい。

「OK」を押して閉じると、次の図のように、赤と青で塗分けられます。この図では、赤が0、青が1をそれぞれ示します。

![「分類」ボタンをクリックした後の状況](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-raster-beginner/calcpixel/2-5.png)

## 1ビットでいいんじゃね？

0と1しか値を取りえないなら、ピクセルタイプは1ビットでいいのではないか、と考えられないでしょうか。

ためしに、ピクセルタイプを1ビットにしてみます。

```sql
DELETE FROM z10;

INSERT INTO z10 (rid, rast)
SELECT
  rid,
  ST_MapAlgebraExpr(
    rast,
    '1BB',
    'CASE WHEN [rast]::NUMERIC >= 10 THEN 1 ELSE 0 END'
  )
FROM dem;
```

``ST_MapAlgebraExpr()``の第2引数が``NULL``から``'1BB'``に変わっています。これで、ピクセルタイプを指定しています。

この結果をQGISで見てみましょう。

![ピクセルタイプが1ビットの時の結果](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-raster-beginner/calcpixel/3-1.png)

黒と白しか出てきません。そして、0になるべき箇所も1になるべき箇所も、1になっているようです。

色の塗分け設定のためにダイアログを開いてみます。

![ピクセルタイプが1ビットの時の色の塗分け設定ダイアログ](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-raster-beginner/calcpixel/3-2.png)

「分類」ボタンがクリックできません。これは、分類できるだけの値の幅が無いためです。最大値が1かつ最小値が1になっていますし。


さらに、``gdalinfo``で、どういうデータになっているかを確認してみましょう。

```
Driver: PostGISRaster/PostGIS Raster driver
Files: none associated
Size is 2250, 1500
Origin = (133.250000000000000,34.500000000000000)
Pixel Size = (0.000111111111111,-0.000111111111333)
Corner Coordinates:
Upper Left  ( 133.2500000,  34.5000000)
Lower Left  ( 133.2500000,  34.3333333)
Upper Right ( 133.5000000,  34.5000000)
Lower Right ( 133.5000000,  34.3333333)
Center      ( 133.3750000,  34.4166667)
Band 1 Block=2048x1500 Type=Byte, ColorInterp=Gray
  NoData Value=0
  Image Structure Metadata:
    NBITS=1
```

NODATA値が0にされています。ここで、なぜNODATA値が0になったのか、この辺の処理は正直分かっていません。

それはともかく、1ビットの場合は、NODATA値に1値が割り当てられたため、残るは1値だけとなります。10m未満に1値、10m以上に1値をそれぞれ割り当てることができません。

言い換えると、10m以上と10m未満とを識別するたけでなく、NODATAであるかどうかを識別できる必要があるので、1ビットでは、値が0か1の2種類しか取りえないので、サポートできません。

## やはり2ビットにしよう

ということで2ビットにしてみました。NODATAを0、10m未満を1、10m以上を2にそれぞれ設定することとしましょう。

```sql
DELETE FROM z10;

INSERT INTO z10 (rid, rast)
SELECT
  rid,
  ST_MapAlgebraExpr(
    rast,
    '2BUI',
    'CASE WHEN [rast]::NUMERIC >= 10 THEN 2 ELSE 1 END'
  )
FROM dem;
```

``CASE``節では、上述していますが、10m未満を0でなく1とし、10m以上を1でなく2とした点に注意して下さい。

![ピクセルタイプが2ビットの時の結果](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-raster-beginner/calcpixel/4-1.png)

うまくいきそうです。色の塗分け設定を変更すると次のようになります。

![色の塗分け設定を変更した場合の2ビットの結果](https://github.com/boiledorange73/zenn-content/raw/main/books-images/pgis-raster-beginner/calcpixel/4-2.png)

うまくいきました。

さっと``gdalinfo``を見てみましょう。

```
Driver: PostGISRaster/PostGIS Raster driver
Files: none associated
Size is 2250, 1500
Origin = (133.250000000000000,34.500000000000000)
Pixel Size = (0.000111111111111,-0.000111111111333)
Corner Coordinates:
Upper Left  ( 133.2500000,  34.5000000)
Lower Left  ( 133.2500000,  34.3333333)
Upper Right ( 133.5000000,  34.5000000)
Lower Right ( 133.5000000,  34.3333333)
Center      ( 133.3750000,  34.4166667)
Band 1 Block=2048x1500 Type=Byte, ColorInterp=Gray
  NoData Value=0
  Image Structure Metadata:
    NBITS=2
```

2ビットで、NODATAは0、となっています。

# おわりに

ピクセル値を近隣ピクセルから独立して演算して、その結果から新しいラスタを生成しました。また、ピクセルタイプに関してもデータサイズを絞りすぎても良くないことをご紹介しました。

これにより、たとえばDEMデータを元に浸水域を算出するのに使えるかも知れません（今回の例は、そのままでは浸水域算出には使えません）。

ただ、使用した関数の実行速度は、ちょっと実用的な速度に達していないように感じられます。大量のラスタデータを処理する場合にはご注意ください。

---
title: "ワールドファイルのはなし"
---
# はじめに

PostGISラスタの仕様に触れる前に、ラスタってそもそも何なの？というのを若干触れたいと思います。特にワールドファイルについて説明したいと思います。

# ワールドファイル

ラスタデータは、主にビットマップ画像データをそのまま利用していると説明しました。

ビットマップ画像データは、左上を``(0,0)``とする左手系の座標系を取り、地理空間とは全く関係ありません。

ビットマップ画像データと地理空間とを結びつける方法の一つがワールドファイルです（他にはGeoTIFFのように直接埋め込む方法があります）。

ワールドファイルは、ラスタデータファイルと同じディレクトリに、異なる拡張子で置かれます。GISは、ラスタデータファイルを読み込む際に、対応するワールドファイルの有無を確認し、存在すればワールドファイルを読みます。

ラスタデータファイル拡張子に対応するワールドファイルの拡張子は ``.TIF``ファイルに対しては``.TGW``、``.PNG``に対しては``.PGW``となります。


## ワールドファイルには空間参照系情報もNoData値も無い

ワールドファイルは、アフィン変換のパラメータしか持っていません。

よって、空間参照系ID (SRID) に関する情報は持ちません。GISでファイルをロードする際に、空間参照系をユーザが手動で指定する必要があります。

また、NoData値も持っていません。

対して、GeoTIFFなど、通常のラスタデータは、空間参照系IDもNoData値も持つことができます。

## ワールドファイルの書式

ワールドファイルは、6行からなり、各行に数値スカラが入っています。これは実はアフィン変換の行列の要素を単純に書いているだけです。

先ほど提示した行列

```
| a11 a12 a13 |
| a21 a22 a23 |
|   0   0   1 |
```

は、ワールドファイルでは、次のようになります。

```
a11
a21
a12
a22
a13
a23
```

## a22は負数を取るのが普通

ピクセル座標系は左上を原点とし、Xは右方向に、Yは下方向に進む、左手系です。対して、たいていの空間参照系は、左下を原点とし、Yについては上方向に進み、右手系になります。

このため、a22 (Y方向のセルサイズ)は、たいてい負の数を取ります。

## ピクセル座標系の原点

ワールドファイルでは、ピクセル座標系の原点は、左上隅ピクセルの中心です。左上隅ピクセルの左上隅ではありません。

# ワールドファイルで遊んでみよう

まず「ラスタデータ」のもととなるビットマップファイルをダウンロードして下さい。

![0.png](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/worldfile/0.png?raw=true)

なんてことはない、20px * 20px のPNG画像です。

これを 0.png とします。

## a21, a12が0の場合

ワールドファイルがたとえば次のようになっているとしましょう。

1.pgw

```
1
0
0
-1
10
20
```

行列で表現すると次の通りとなります。

```
|  1  0 10 |
|  0 -1 20 |
```

ピクセル（左上隅原点）の座標値を``(px, py)``とすると、投影された座標値は次のようになります。

```
ux =  1 * px + 10
uy = -1 * py + 20
```

これは、左上隅ピクセルの中心が(10,20)で、セルサイズが横方向に1, 縦方向に1となることを示しています。

同じフォルダで、先ほどダウンロードした [0.png](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/worldfile/0.png?raw=true) を単純に複写して 1.png を作り、GISで読み込んでみてください。次のように表示されると思います。

![表示された0.png](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/worldfile/1.png?raw=true)

これに、(10,20)の位置に円を描いてみます。

![(10,20)に点を打った0.png](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/worldfile/2.png?raw=true)

左上隅のピクセルの中心あたりに円が描かれました。

次に、横長にしてみましょう。

2.pgw

```
2
0
0
-1
10
20
```

``(ux,uy)``と``(px,py)``の関係は次のようになります。

```
ux =  2 * px + 10
uy = -1 * py + 20
```

同じフォルダで、先ほどダウンロードした [0.png](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/worldfile/0.png?raw=true) を単純に複写して 2.png を作り、GISで読み込んでみてください。

![横長になった図](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/worldfile/3.png?raw=true)

たしかに横長になりました。

## 回転する例

1.png + 1.pgw の画像を、反時計回りに``cos(t) = 0.6``, ``sin(t) = 0.8``となる``t``(おおむね52度)で回転させてみましょう。

2*2の回転行列は、右手系（右下隅原点の座標系）だと、数学の教科書で出てくると思います。

```
| cos(t) -sin(t) |
| sin(t)  cos(t) |
```

右から順に、Yを逆転して左手系（左上隅座標系）から右手系にして、``t``だけ回転させ、(10, 20) ぶんシフトさせてみます。

```
| 1 0 10 |   | 0.6 -0.8 0 |   | 1  0 0 |   | 0.6  0.8 10 |
| 0 1 20 | * | 0.8  0.6 0 | * | 0 -1 0 | = | 0.8 -0.6 20 |
| 0 0  1 |   |   0    0 1 |   | 0  0 1 |   |   0    0  1 |
```

合成された行列の3行目を捨てると、次のようになります。

```
| 0.6  0.8 10 |
| 0.8 -0.6 20 |
```

これをワールドファイルにすると、次のようになります。

3.pgw

```
0.6
0.8
0.8
-0.6
10
20
```

``(ux,uy)``と``(px,py)``の関係は次のようになります。

```
ux = 0.6 * px + 0.8 * py + 10
uy = 0.8 * px - 0.6 * py + 20
```

これもやはり、同じフォルダで、先ほどダウンロードした [0.png](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/worldfile/0.png?raw=true) を単純に複写して 3.png を作り、GISで読み込んでみてください。

![回転した図](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/worldfile/4.png?raw=true)

回転しているのが見えます。

1.png と重ねると、次のようになります。

![回転前と回転後を重ねた図](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/worldfile/5.png?raw=true)

## スキューのある行列

Y方向へのスキューを行ってみましょう。

4.pgw

```
1
0.5
0
-1
10
20
```

繰り返しになりますが、``(ux,uy)``と``(px,py)``の関係は次のようになります。

```
ux = 1.0 * px + 10
uy = 0.5 * px - 1 * py + 20
```

これは、pxが1増えるたびに、uxは1.0, uyは0.5増えて、pxはux, uyの双方に影響を与えることになります。pyが1増えるたび、uyは1減るだけで、pyはuxのみに影響を与え、uxに影響は与えません。

同じフォルダで、先ほどダウンロードした [0.png](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/worldfile/0.png?raw=true) を単純に複写して 4.png を作り、GISで読み込んでみてください。

![スキューした図](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/worldfile/6.png?raw=true)

Yの上方向にスキューが行われました。

1.png と重ねてみましょう。

![スキュー前とスキュー後を重ねた図](https://github.com/boiledorange73/zenn-content/blob/main/books-images/pgis-raster-beginner/worldfile/7.png?raw=true)

ここでは示しませんが、2行目と3行目を入れ替えるとX方向へのスキューとなります。

## a21, a12 は 0 にする方が無難

ワールドファイルは読むけど、a21, a12 はまともに読まずに0にしてしまうGISもあるとか聞いてますので、使わなくていいなら使わない方が無難です。スキューを持つ画像については、リサンプリングしていいなら、リサンプリングを使ってスキューを持たない画像にした方がいいです。ただ、NoData値を含むラスタデータについては、リサンプリングが許されず、どうしてもスキューを持つラスタデータも必要になる可能性もあります。

# おわりに

ラスタデータというものがあり、ラスタデータは、ビットマップデータと地理参照情報、NoData値からなることを示しました。また、地理参照情報付与手法としてワールドファイルとよぱれるものがあり、これはアフィン変換の行列の要素値を記述していることを説明しました。

ラスタデータの基本は、どんな種類のデータでも、これと大差ありません。

ただし、GISで扱うラスタデータは一般的なデジカメ写真をはるかに超えるピクセル数を扱うことが結構ありますので、ステップアップするごとに苦しくなっていくと思います（利用者も計算機も）。

---
title: "MapServerのWCSサーバでタイムアウトまで待たされる場合の対応"
emoji: "😀"
type: "tech"
topics: [MapServer]
published: true
---
# はじめに

まずそういうおかしい人はいらっしゃらないと思いますが、[WCS](http://www.opengeospatial.org/standards/wcs)サービスで、全国網羅した10mメッシュのラスタを配信しようとしているとします。

[MapServer](http://www.mapserver.org/)はWCSサーバになれるので、これを使うとするのは、おかしな話ではありません。

しかしここで、ひっかかるところがあります。

# 前振り
## 具体的な症状

* QGISからWCSレイヤの追加をしようとすると、操作できない状態に陥る
* １分程度以上経った後、QGISで「ネットワークがタイムアウトしました.受信したデータは不完全かもしれません」というエラーが出て、操作不能から復帰する
* MapServerのERRORFILEを取っていると、この時点で``msDrawRasterLayerLow(<レイヤ名>): entering.``*だけ*が出る（``mapserv request processing time``等が出ない）

## 原因

HTTPのログを見ると QGISからGetCoverageリクエストが出ていること、それを受けてMapServerが動作しているのは確かなのですが、QGISからのリクエストが
``BBOX=122.875,20.41666666666419871,154.0000000000030127,45.58333333333330017&WIDTH=10&HEIGHT=10``(CRSはEPSG:4612)となっていました。

このリクエストは、日本全体(280124x226499ピクセル)のデータを舐めて、10x10ピクセルでリサンプリングした結果を返せ、と求めているものです。

HTTPサーバは、QGISの無茶振りなリクエストを受け付けて、正直にMapServerを起動させてしまい、無茶振りなのでMapServerはえんえんとデータ取得を続け、HTTPサーバが途中であきらめて 504 Gateway Timeout をQGISに返して、「タイムアウトしました」と表示された、というわけです。

## 対応方針

BBOXが広範囲になる無茶振りは毅然と断ることにします。

# MAXSCALEDENOMが効かない

``LAYER``に対して``MAXSCALEDENOM``を設定すると、指定した縮尺（の逆数）より小縮尺（広範囲）では表示しないようにできます。この指定はWMSでは動いたので、これでOKだろうと思っていましたが、効きませんでした。

# 対応策

## ソースを見る

なんでMAXSCALEDENOMが効かないのか、ソースを見て原因を探ってみましょう。

### maxscaledenomのチェックをしてくれない

GetCoverageリクエストの処理は、``mapwcs.c``の``msWCSGetCoverage()``で行っています。

この関数から、``mapraster.c``にある``msDrawRasterLayerLow()``を呼んで、カバレッジになるデータを取得しています。

この関数の中で、maxscaledenomが使われている箇所は次のようになっています。

```c
  if(map->scaledenom > 0) {
    if((layer->maxscaledenom > 0) && (map->scaledenom > layer->maxscaledenom)) {
```

なお、``map->scaledenom``はデフォルトでは``-1``になっています。そして、``mapwcs.c``では``map->scaledenom``は設定されていません。

ということで、maxscaledenomのチェックは行われません。

### GEOWIDTH で対応する

``mapraster.c``の上記の箇所の直後に次のような箇所があります。

```c
  if(layer->maxscaledenom <= 0 && layer->minscaledenom <= 0) {
    if((layer->maxgeowidth > 0) && ((map->extent.maxx - map->extent.minx) > layer->maxgeowidth)) {
```

これならextentは設定されているので使えます。

ということで、マップファイルに次のものを追加します。

```
  LAYER
    ...
    GEOWIDTH 1
  END
```

これで、（元データの空間参照系がEPSG:4612等の場合には）リサンプリング元のデータのサイズの最大値が緯度1度×経度1度(10mメッシュの場合12000×8000ピクセル)となります。

### MAXSCALEDENOMは入れちゃダメ

先ほど示した``mapraster.c``の一部をもういちど出します。

```c
  if(layer->maxscaledenom <= 0 && layer->minscaledenom <= 0) {
    if((layer->maxgeowidth > 0) && ((map->extent.maxx - map->extent.minx) > layer->maxgeowidth)) {
```

このコードから、MAXSCALEDENOMが設定されている場合にはMAXGEOWIDTHのチェックが行われないことが分かります。

GEOWIDTHとMAXSCALEDENOMは同居できないうえ、MAXSCALEDENOMが優先されることになります。そのうえWCSでは必ずMAXSCALEDENOMのチェックは行われません。

少なくとも現時点では、WCSではMAXSCALEDENOMを入れてはダメ、ということになります。

# おわりに

MapServerのWCSサーバを使用する際に、タイムアウトの原因となる、広範囲のBBOX指定に対して、サーバが断る方法として、単独でMAXGEOWIDTHを設定することを示しました。

WCSサーバとして使う場合に特有の話です。WMSではMAXSCALEDENOMを使った方が良いかと思います。

また、こういった問題がある以上、WCSとWMSの両方に対応するレイヤは作らない方が良いと思います（もともとそんなことしないと思いますが）。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

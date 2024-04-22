---
title: "グリッドレイヤを作るならcanvasレイヤがいいと思う"
emoji: "😀"
type: "tech"
topics: [Leaflet,GIS]
published: true
---

# はじめに

https://github.com/boiledorange73/Layer.JPGrid/ にて地域メッシュコードを表示するレイヤを表示しています。また、これの使用例を https://boiledorange73.github.io/Layer.JPGrid/ で見ることができます。

地域メッシュコードについての詳細情報は https://ja.wikipedia.org/wiki/%E5%9C%B0%E5%9F%9F%E3%83%A1%E3%83%83%E3%82%B7%E3%83%A5 をご覧下さい。

今回は、これの若干の解説をしたいと思いますが、地域メッシュコードレイヤは、非タイル canvas を使っています。

## なぜ 非タイル canvas ?

なぜ 非タイル canvas を使ったレイヤを使うかと言うと、グリッドを``div``要素などで埋める方法より早いからです。ていうか、グリッドを DOM で埋めると、イベント処理などで、いろいろ余分な処理を含むことになるので、確実に遅くなります。

忘れましたけど「確実に」どころか「とんでもなく」遅くなったような気がします。

対して、1枚の canvas に必要な範囲だけグリッドを描画する方が、実用的なコストで利用できるはずです (少なくとも私の環境では)。

それと canvas 要素は、タイル状に敷き詰めるのでなく、地図表示範囲全体にわたって、一つの要素にしています。

# 描画ルーチンを見る

https://github.com/boiledorange73/Layer.JPGrid/blob/main/BO.L.Layer.Grid.js

の ``_draw()`` を見ていきます。

## 概要

* canvasを地図表示範囲にあわせる
* canvas全体をクリアする
* 表示範囲に対応する行範囲、列範囲を計算する
* グリッド位置をピクセル単位で計算する
* グリッドを描画する
* セルごとに内部を描くルーチンを呼ぶ

## canvas要素をメインにあわせる

canvas要素は、地図の移動や拡大縮小に合わせて、左上位置やサイズが変わっています。再描画する前に、canvas要素を地図要素に合わせ直します。

``_fit_canvas()``でやっています

```js
        "_fit_canvas": function _fit_canvas() {
            if( !this._map ) {
                return null;
            }
            var topLeft = this._map.latLngToLayerPoint(this._map.getBounds().getNorthWest());
            var size = this._map.latLngToLayerPoint(
                this._map.getBounds().getSouthEast()
            )._subtract(topLeft);
            L.DomUtil.setPosition(this._canvas, topLeft);
            this._canvas.width = size.x;
            this._canvas.height = size.y;
            return this;
        },
```

## canvas全体をクリアする

わざわざ書かなくていいかもしれないけど、いちおう書いておきます。

```js
            // clears all
            ctx.clearRect(0, 0, this._canvas.clientWidth, this._canvas.clientHeight);
```

## 表示範囲に対応する行範囲、列範囲を計算する

最初に地図の緯度経度ベースの表示範囲 (``vb``) を得ます。

```js
            var vb = this._map.getBounds();
```

次に、``options``から、原点緯度経度、対角点緯度経度、行数、列数を得ます。

```js
            // r = rows*(vb.lat-org.lat)/(diag.lat-org.lat)
            var org = this._origin;
            var dag = this._diagonal;
            var dlt = {"lat": dag.lat - org.lat, "lng": dag.lng - org.lng};
            var rows = this._rows;
            var cols = this._columns;
```

地図の緯度経度ベースの表示範囲 (``vb``) を囲む行範囲と列範囲を求めます。
まずは、行範囲です。行インデックスの最小値が原点に近い側で、最大値が対角点に近い側となります。

```js
            // rows in the view
            var rmin = parseInt(rows * (vb.getSouth()-org.lat)/dlt.lat);
            var rmax = parseInt(rows * (vb.getNorth()-org.lat)/dlt.lat);
            var r;
            if( rmin > rmax ) {
                r = rmin;
                rmin = rmax;
                rmax = r;
            }
```

上記の``rmin``、``rmax``の行範囲は、地図表示範囲全体の行範囲なので、``options``で指定されている行範囲より広い範囲なら、``rmin``、``rmax``の行範囲を狭めます。

```js
            // forces rows into extent
            if( rmin < 0 ) {
                rmin = 0;
            }
            if( rmax > rows - 1 ) {
                rmax = rows - 1;
            }
```

列でも同じことします。

```js
            // columns in the view
            var cmin = parseInt(cols * (vb.getWest()-org.lng)/dlt.lng);
            var cmax = parseInt(cols * (vb.getEast()-org.lng)/dlt.lng);
            var c;
            if( cmin > cmax ) {
                c = cmin;
                cmin = cmax;
                cmax = c;
            }
            // forces columns into extent
            if( cmin < 0 ) {
                cmin = 0;
            }
            if( cmax > cols - 1 ) {
                cmax = cols - 1;
            }
```


## グリッド位置をピクセル単位で計算する

グリッド位置を計算します。

まず、地図表示範囲の左上隅の、世界地図上のピクセル単位の位置 ``topLeft`` を得ます。

``latLngToLayerPoint()``の返す位置は、世界地図上のピクセル単位の位置です。

でも描画で欲しいのは、地図表示範囲の左上を原点とした、ピクセル単位の位置 (以下「相対ピクセル位置」とします) ではないので、``topLeft`` で引き算をして、相対ピクセル位置を得ています。

```js
            // topLeft's position (pixel)
            var topLeft = this._map.latLngToLayerPoint(this._map.getBounds().getNorthWest());
```

そして、グリッド位置(緯度経度)を配列に保存して、相対グリッド位置を計算して、配列に保存する、というループを実行してます。


```js
            // calculates all points (rmin,cmin to rmax+1,cmax+1)
            var grid_latlngs = [];
            var grid_points = [];
            var rcnt = rmax-rmin+1;
            var ccnt = cmax-cmin+1;
            var rix, r;
            for( rix = 0, r = rmin; r <= rmax+1; rix++, r++ ) {
                // new row
                grid_latlngs[rix] = [];
                grid_points[rix] = [];
                // lat
                var lat = r / rows * dlt.lat + org.lat;
                var cix, c;
                for( cix = 0, c = cmin; c <= cmax+1; cix++, c++ ) {
                    // lng
                    var lng = c / cols * dlt.lng + org.lng;
                    // latlng
                    grid_latlngs[rix][cix] = {"lat":lat, "lng":lng};
                    // latlng to point (may be used by this.drawOne)
                    grid_points[rix][cix] =
                        this._map.latLngToLayerPoint(grid_latlngs[rix][cix]).subtract(topLeft);
                }
            }
```

上記スクリプトの中の最後に緯度経度位置から相対ピクセル位置の計算が出ています。

```js
                        this._map.latLngToLayerPoint(grid_latlngs[rix][cix]).subtract(topLeft);
```

```latLngToLayerPoint()``` でで世界地図上のピクセル単位の位置を得て、そこから左上隅のピクセル位置を引くことで、相対ピクセル位置を得ています。

## グリッドを描画する

グリッド描画は、描画範囲の端から端までの横線を必要本数描画して、縦線を描画しています。

セルごとに四角形の境界線を描いていっていないのは、その方法だと、境界線にボーダーを施すと、見た目が悪くなるためです。


```js
            // draws all
            // draws border of border
            if( this.options.borderBorderWidth > 0 ) {
                this._draw_border(
                    ctx, grid_points,
                    rmin, cmin, rmax+1, cmax+1,
                    this.options.borderWidth+2*this.options.borderBorderWidth,
                    this.options.borderBorderStyle
                );
            }
            if( this.options.borderWidth > 0 ) {
                this._draw_border(
                    ctx, grid_points,
                    rmin, cmin, rmax+1, cmax+1,
                    this.options.borderWidth,
                    this.options.borderStyle
                );
            }
```

## 中身を描画する

``drawOne`` を呼び出しています。呼出しの際には、対応するセルの境界のポリゴンを、緯度経度単のものと、相対ポイント単位のものとを生成して、``drawOne``に渡しています。

```js
            // draws each cell.
            if( this.drawOne && !this.options.hideInternal ) {
                for( rix = 0, r = rmin; r <= rmax; rix++, r++ ) {
                    for( cix = 0, c = cmin; c <= cmax; cix++, c++ ) {
                        var ring_ll = [
                            grid_latlngs[rix][cix],
                            grid_latlngs[rix+1][cix],
                            grid_latlngs[rix+1][cix+1],
                            grid_latlngs[rix][cix+1],
                            grid_latlngs[rix][cix]
                        ];
                        var ring_p = [
                            grid_points[rix][cix],
                            grid_points[rix+1][cix],
                            grid_points[rix+1][cix+1],
                            grid_points[rix][cix+1],
                            grid_points[rix][cix]
                        ];
                        this.drawOne(ctx, r, c, [ring_p], [ring_ll]);
                    }
                }
            }
```

## 地域メッシュコードの場合

これより前のコードは Grid レイヤで、地域メッシュコードレイヤは、これを継承した JPGrid レイヤです。

https://github.com/boiledorange73/Layer.JPGrid/blob/main/BO.L.Layer.JPGrid.js

ここでは、このレイヤの ``drawOne`` を見ます。

```js
        "drawOne": function drawOne(ctx, r, c, poly_p, poly_ll) {
            // box (latlng)
            var box_ll =  poly2box_ll(poly_ll);
            var latc = 0.5*(box_ll[0]+box_ll[2]);
            var lngc = 0.5*(box_ll[1]+box_ll[3]);
            // text
            var text = BO.UT.JPGrid.latlng2jpgridcode(latc, lngc, this.options.level);
            // box (pixel)
            var box_p = poly2box_p(poly_p);
            var xc = 0.5*(box_p[0]+box_p[2]);
            var yc = 0.5*(box_p[1]+box_p[3]);
            var pw = box_p[2]-box_p[0];
            var ph = box_p[3]-box_p[1];
            // draws
            ctx.font = "" + this.options.fontSize + "px sans-serif";
            var tmesure = ctx.measureText(text);
            var tw = tmesure.width;
            var th = tmesure.actualBoundingBoxAscent + tmesure.actualBoundingBoxDescent;
            if( pw >= tw && ph >= th ) {
                ctx.textBaseline = "middle";
                ctx.lineWidth = this.options.textBorderWidth;
                ctx.strokeStyle = this.options.textBorderStyle;
                ctx.strokeText(text, xc-0.5*tw, yc);
                ctx.fillStyle = this.options.borderStyle;
                ctx.fillText(text, xc-0.5*tw, yc);
            }
        },
    });
```

ざっとしか説明しませんが、ポリゴンからボックスを求めて、それの中心点を求めて、その中心点からメッシュコード文字列を生成して、canvasに書き込んでいます。

# おわりに

いかがだったでしょうか。

canvas要素を使った Leaflet レイヤを作ってみました。グリッドの場合は、canvas を使って自前で描画する方が、DOM要素を多数埋め込むよりも、高速なはずです。

少なくとも、地域メッシュを表示するのは、うまくいきました。

今回は地域メッシュコードのレイヤでしたが、他のグリッドレイヤにも応用できると思います。

最後に、繰り返しになりますが https://boiledorange73.github.io/Layer.JPGrid/ で、地域メッシュコード地図を見ることができます。実用的な速度で動作しているものと信じています (断定してない)。ふと地域メッシュコードが知りたくなった時に使えるものと思います。

## 他サイトの地域メッシュコードを見るアプリ

よく作ったものと思うけど、先に作っておられる方々がいらっしゃいましたので、末尾ながら、ご紹介します。

  * https://www.arcgis.com/apps/instant/lookup/index.html?appid=ec8abf80f76c4417b01561e303ed2d32
  * https://goinc-oss.github.io/mesh-viewer/

ていうか、私が何のために作ったのか分からなくなってきたじゃねえか orz

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

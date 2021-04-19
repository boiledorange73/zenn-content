---
title: "WMTSで地域を絞り込んだうえでPostGISレイヤを表示する"
---
# はじめに

「QGISでPostGISのデータを見てみよう」で、QGISを使ってPostGISのデータを見てみました。

このときは、国土数値情報（行政区域）の広島県だけを使っていましたが、今度は、全国をインポートしたいと思います。

しかし、全国のポリゴン全体をQGISで表示するには、とんでもなく時間がかかります。

全体を表示すると、次のようになります。

![全国ポリゴンの全体を表示したところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/1.png)

どうも無駄に時間をくっている気がします。あと、ここで塗りつぶし色を変更したら、もう一度データ取得からやり直しになりますので、かなり面倒です。

見たい箇所だけに絞ろうとすると、たとえば「広島県の範囲だけ見たい」と考えた場合に、経度緯度の範囲が簡単に分かればいいのですが、計算するのもまた面倒です。

そこで、QGISから、WMSに基づく地図配信サービスを併用して、地域を絞り込んだうえでPostGISデータを見る方法を紹介します。

## WMTS?

WMTSはWeb Map Tile Serviceの略で、地図タイル画像のURLや空間参照系の情報のメタデータを定義しています。https://www.ogc.org/standards/wmts から、仕様書をダウンロードできます。

ここでは、[地理院地図](https://maps.gsi.go.jp/)の地図タイル画像に関するWMTSメタデータを使用します。

サービスポイントは二種類あり、それぞれ次の通りです。

|名称|URL|
|----|---|
|地理院地図(軽量版)|https://gsi-cyberjapan.github.io/experimental_wmts/gsitiles_wmts_light.xml|
|地理院地図|https://gsi-cyberjapan.github.io/experimental_wmts/gsitiles_wmts.xml|

通常は、軽量版を使うと良いと思います。

# QGIS 3.16 からWMTSを使う方法

## WMTSサーバを登録する

とりあえずメニューから「レイヤ」→「WMS/WMTSレイヤの追加」を選択します。

![WMS/TMSレイヤの追加を選択しようとしている図](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/2.png)

「WM(T)Sサーバからレイヤを追加」ダイアログが現れます。

現時点では、WMSサーバは全く登録されていませんので、登録しましょう。
「新規」ボタンをクリックします。

![「WM(T)Sサーバからレイヤを追加」ダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/3.png)

「新しいWMSコネクションの作成」ダイアログが表示されます。通常、入力に必要なのは、「名称」と「URL」です。

名称は、ユーザが識別するために使用するもので、自由に記述できますが、自分にとって分かりやすくして下さい。URLは、サービスポイントURLを指定します。

今回は、名称を「地理院地図 WMTS Light」、URLを``https://gsi-cyberjapan.github.io/experimental_wmts/gsitiles_wmts_light.xml``とします。

![「新しいWMSコネクションの作成」ダイアログで名称とURLを入力したところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/4.png)

その他の設定は、とりあえず無視してOKです。

登録がうまくいくと、ひとつ前のダイアログに、名称が表示されていると思います。

![「地理院地図 WMTS Light」というエントリが生成されたことを示す図](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/5.png)

## 接続する

「WM(T)Sサーバからレイヤを追加」ダイアログで「接続」ボタンをクリックします。
表にずらっと何かが表示されていますが、これはレイヤ一覧です。各行が一つのレイヤに対応しています。最後の行が青地白文字になっていますが、このレイヤが選択されていることを示しています。
複数選択はできません。

![レイヤ一覧が表示され標準地図レイヤが選択されているところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/6.png)

選択したうえで「追加」ボタンを押すと、レイヤ追加が実行されます。「閉じる」をクリックすると、レイヤ追加ダイアログが閉じられます。
これで、地理院地図が表示されます。

![地理院地図を表示させているところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/7.png)

## PostGISはできるだけ表示しないように操作する

QGISを操作して、見たい範囲だけを表示するようにします。

![中国地方全体が表示される範囲を表示しているところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/11.png)

ここまで来たら、PostGISレイヤを表示させます。

![PostGISレイヤを追加したところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/12.png)

別の範囲に移りたい際には、いったん「レイヤ」ペインのPostGISレイヤのチェックボックスをオフにして、PostGISレイヤを表示しないようにします。

![PostGISレイヤを表示しないようにしたところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/13.png)

この状態で移動します。

![和歌山県の一部が表示される範囲を表示しているところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/14.png)

あらためて、PostGISレイヤを表示させます。

![PostGISレイヤを表示させたところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/15.png)

## レイヤの順序を入れ替える

地理院地図と市区町村ポリゴンの二つのレイヤを持っている場合には、順序に気を付けて下さい。次のように地理院地図の方が上位に来ていると、市区町村ポリゴンは表示されません。

![地理院地図を表示させているところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/8.png)

順序を次のように変更すると、市区町村ポリゴンが表示されるようになります。順序変更はドラッグで可能です。

![地理院地図を表示させているところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/9.png)

## 全体表示をしてみる

ある程度時間がかかりますので注意して下さい。

「レイヤ」ペインのPostGISレイヤを右クリックでおさえて、コンテキストメニューを出し、「レイヤの領域にズーム」を選択します。

![「レイヤの領域にズーム」を選択したところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/16.png)

しばらく、プログレスバーが動いて頑張ってロードします。しばらく待ちましょう。

![ウィンドウ下のプログレスバーが動いて頑張っているところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/qgis-wmts/17.png)

## 範囲を絞り込んでも全体表示と同じパフォーマンスになる場合

たぶん、ジオメトリに対して空間インデックスを貼っていないためです。

``CREATE INDEX <インデックス名> ON <テーブル名> USING GiST (<ジオメトリカラム名>);``を実行して下さい。

# おわりに

それなりのサイズの地理空間データを扱うまでは、たぶんこの記事について、実感がわかないと思いますが、そのうち、ちょっと表示したいのにロードがなかなか終わらない問題に直面することと思います。その際に思い出して頂ければいいかな、と思っています。

# 出典

都道府県ポリゴンには国土数値情報（行政区域）を使用しました。
地理院地図を使用しました。

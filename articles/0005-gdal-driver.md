---
title: "JSONなラスタをGDALでドライブ"
emoji: "😀"
type: "tech"
topics: [GDAL]
published: true
---
# はじめに

http://aginfo.cgk.affrc.go.jp/rstprv/index.html.ja では、10m DEMを「切り取り」できるのですが、生ラスタをそのままJSONにしてしまった書式でダウンロードできます。

仕様については http://aginfo.cgk.affrc.go.jp/rstprv/docs/rasterjson/index.html.ja 参照。

GDALドライバを作って、この書式をGDALに対応させてやる、というのが今回の趣旨です。

## GitHubに上げています

https://github.com/boiledorange73/gdal_rasterjson にあります。

## GDAL ?

GDAL/OGR ( http://www.gdal.org/ ) は、ラスタデータ、ベクタデータに対して、投影変換等の幾何的な変換を施したり、別の書式で出力したり、といった、（さまざまな意味での）変換を行うためのライブラリ+CUIプログラム群です。

GDALがラスタデータ処理の部分で、OGRがベクタデータ処理の部分ですが、統合されている(libgdalに全て入っている)ので、相互に必要なものをつまんでいけます。

Advent Calendarの一昨日のエントリ「GDALをC++ライブラリとして使ってみた」( http://www.k.nakao.name/noisefactory/blog/2015/12/03/using-gdal-as-a-c-plus-plus-api/ ) と「続GDALをC++ライブラリとして使ってみた」 ( http://www.k.nakao.name/noisefactory/blog/2015/12/04/using-gdal-as-a-c-plus-plus-api-2/ ) で、C++を使った例を提示して下さってます。

## GDALドライバ ?

GDALは、新しい書式に対応するために、その書式の読み込み、書き出しを行う「ドライバ」を組み込むことができます。読み込みと書き出しさえ行えるようにすれば、特別な書式を扱うことができるようになります。

開発チュートリアルは http://www.gdal.org/gdal_drivertut.html にあります。

## まともに解説できねえorz

GDALドライバを、他のドライバを参考にしながらチマチマやっているため、当人さえ理解できていない部分があります。

また、このコードは、けっこう前に書いたので、すっかり忘れています。

「２週間前の自分は他人」なのです。
何度もこれで痛い目見てるっていうのに…。

# JSONパーサを作る

## 中途な文の残骸が無いと困る

JSONパーサについては、JSON-C等各種あったのですが、文法通りの文書でないとエラーになります。というと当然なのですが、``RasterJSONDataset::Identify``は、データを完全には渡さずに最初の1024バイトだけ渡しています。その中途半端なデータ列だけで、このドライバがサポートする書式であるかどうかを判断します。バイナリ書式だと、先頭数バイトに、その書式を識別するためのバイト列が埋め込まれることが多いので、そこだけで判断できます。

しかし、JSONの場合は、そうはいきません。

プロパティ``type``の値が文字列リテラル``raster``であることで識別していますが、JSON(というか文脈自由文法)である以上、先頭から"{"がはじまるとは限りません(空白があるかも知れない)。

そのうえ、1025バイト以上の位置でルート要素の``}``が出現する場合、1024バイトまでしか読んでいないので、範囲内には``}``が存在しないので、文法エラーとなります。

そのため、読めるところまで読んで、パースできるところまでパースして、パースできたところで``"type": "raster"``があればOKにしなければなりません。

## bison使いたかったが

bison等はLR法([WikiPedia](https://ja.wikipedia.org/wiki/LR%E6%B3%95)記事)です。これは、BNFとイベント規則の組み合わせの集合からなっていますが、イベントは、その規則が確定した際に実行されます。このため、途中で切れた不完全な文を読み込まそうとすると、最上位の規則が確定するまで結果が返りません。

## JSONはLL(1)でOK

じゃあLL法([WikiPedia](https://ja.wikipedia.org/wiki/LL%E6%B3%95)記事)で文法書いてしまおう、と考えにいたる。バックトラックしないから、今回のように、不完全な文を相手にする場合にはピッタリです。
JSON（http://www.json.org/json-ja.html) に文法が掲げられています）がLL(k)で受理できるか、という点が気になりますが、LL(1)で受理できます。

[jsonpull.c](https://github.com/boiledorange73/gdal_rasterjson/blob/master/jsonpull/src/jsonpull.c) にこっそり書いてあります。

それと、他のライブラリ等から引っ張ってきた関数と名前がバッティングすると困るので、jsonpull/src/*.{h|c} を、 RasterJSON.cpp に展開する暴挙に出ています。``cpp``(プリプロセッサ)でお楽しみ下さい。

# JSON出力というより書式付き浮動小数点出力

https://github.com/boiledorange73/gdal_rasterjson/blob/master/jsonpull/src/textencoder_appendf.c

このファイルの一番最後に ``TextEncoder_AppendF`` というのがありますが、ここからいろいろゴニョゴニョしています。

``AnalyzedFormat_Analyze``で、アタマから順に、書式部分（``%``からはじまる部分）かリテラル部分（書式部でない部分）を要素に持つリストを生成します。リテラル部分はそのまま出力するだけです。
書式部分については、整数部の桁数は対数``mylog10``（log2を出して、基数変換しちえます）で算出し、浮動小数点数の小数部については、``0.5*mypow10(-precision)``（有効桁数の1つ小さい桁が5になる）を足したうえで``myfloor``（本当はfixですねこれ…）で切り捨てて、四捨五入としています。

なお、``%p``、``%a``は対象外ですのでご注意下さい。

最終的な出力は、[textencoder.c](https://github.com/boiledorange73/gdal_rasterjson/blob/master/jsonpull/src/textencoder.c)にある``TextEncoder_AppendChar``を呼び出して、バッファに書き込んでいます。

# さて本題

[RasterJSON.h](https://github.com/boiledorange73/gdal_rasterjson/blob/master/RasterJSON.h)と[RasterJSON.cpp](https://github.com/boiledorange73/gdal_rasterjson/blob/master/RasterJSON.cpp)とがドライバ本体です。

クラスは、``RasterJSONDataset``（データセット）と``RasterJSONRasterBand``（バンド）です。

読み込みは``RasterJSONDataset::Open``、書き出しは``RasterJSONDataset::CreateCopy``で、それぞれ行います。

``GDALRegister_RasterJSON``で、このドライバの説明、オプション等が指定できます。

## Open

JSONパーサJsonNodeを生成して、そこから各プロパティが適切かチェックして、``JsonNode *pJsonRoot``にルート要素へのポインタを入れ、``JsonNode *pJsonValues``に``values``プロパティから得られた配列へのポインタを入れておきます。

```cpp
    JsonNode_GetReal(JsonNode_ArrayGet(json_target, 4), &(poDS->adfGeoTransform[0]));
    JsonNode_GetReal(JsonNode_ArrayGet(json_target, 0), &(poDS->adfGeoTransform[1]));
    JsonNode_GetReal(JsonNode_ArrayGet(json_target, 1), &(poDS->adfGeoTransform[2]));
    JsonNode_GetReal(JsonNode_ArrayGet(json_target, 5), &(poDS->adfGeoTransform[3]));
    JsonNode_GetReal(JsonNode_ArrayGet(json_target, 2), &(poDS->adfGeoTransform[4]));
    JsonNode_GetReal(JsonNode_ArrayGet(json_target, 3), &(poDS->adfGeoTransform[5]));
```

これは、JSONのtransformプロパティとpoDS->adfGeoTransformの並び順とが違う点に注意が必要です。

## Read

``RasterJSONDataset::ReadBlock`` にある通り、``pJsonValues``から引き出してバッファに複写しています。

RasterJSONRasterBand::IReadBlock

## Identify

これは仮想関数の実装でもオーバライドでもオーバロードでもない、普通のメンバ関数です。

1024バイトの文字列から、このドライバがサポートするデータかどうかを確認します。

1024バイト以上だとエラーとなりますが、無視して、残骸から``"type": "raster"``があればTRUE（真）、なければFALSE（偽）としています。

## CreateCopy

``RasterJSONDataset::CreateCopy``は、書き出し処理を行っています。とにかくバッファ（線型リンクリスト）にJSON文字列を書き出して、最後に``while``ループ内で``VSIFWriteL``を実行していくようにしています。

# プラグインとしてコンパイルする

これはFreeBSD上で確認したもので、たぶんUNIXライクなOSなら大丈夫だろうと思いますが、Windowsについては分かりません。

## コンパイル

普通に.soファイルを作ります。この際、ファイル名を``gdal_(フォーマット名).so``とします。フォーマット名は大文字小文字の区別がありますので、ご注意ください。RasterJSONというフォーマットのドライバをGNU C++でコンパイルする場合には、次のようになります。

```bash
c++ -fPIC -shared RasterJSON.cpp -o gdal_RasterJSON.so `gdal-config --cflags` `gdal-config --libs` (ほかのオプション)
```

## インストール先

``$(PREFIX)/lib/gdalplugins``というディレクトリができていて、ここに投入すれば読みに行ってくれるようになっています。rootになって、次のようにすれば入ります。

```bash
cp gdal_RasterJSON.so "`gdal-config --prefix`/lib/gdalplugins"
```

## 参考

* https://trac.osgeo.org/gdal/wiki/ConfigOptions#GDAL_DRIVER_PATH にプラグインに関する説明がありました（環境変数のGDAL_DRIVER_PATHの説明ですが）。
* [OGRSFDriverRegistrar::AutoLoadDrivers()](http://gdal.org/1.11/ogr/classOGRSFDriverRegistrar.html#ae98efe100c0ecea4f6868e008838f878)にも同じような説明があります。
* 同じことを https://github.com/boiledorange73/gdal_rasterjson/blob/master/README に書いています。

# おわりに

自分で何書いたかわからん部分も結構あって、文書としては、とても非常にたいへんおかしいものとなっていますが、とりあえず動きます、本当です、信じて下さい（あくまで1.x用でして、2.xでの動作は確認もしていません）。

このドライバを作成するにあたって費やされた労力のほとんどが、JSONパーサ（書いてませんが当然字句解析もセット）、書き出し（特に、浮動小数点の文字列化など）です。かなり面倒だったのですが、そこさえクリアすればOK。多数のGDALツールやMapServer等のGDALライブラリを使った様々なアプリケーションで恩恵を受けることができます。

このプログラムの問題点としては、メモリの使い方が下手であること。``CreateCopy``で順次バッファ内を吐き出しているわけではない点がまず挙げられます。
あと、``Open``で生成したJSONツリーをそのまま保持している点も、できればどうにかしたい部分もあります。レギュラーなデータサイズなら、``ReadBlock``において引数からファイルポインタの頭出しが可能で、メモリにデータ全体を置いておく必要がなくなるのですが、そういうことができていません。とはいえ、これは面倒だからやりたくないです。だれかやって。

また、とりあえずプラグインドライバとしてもコンパイルできるようになりました。当初はソースに組み込まないといけなかったので、少しですが面倒さが減りました。

あと、ガーッと書いたため、ソース内のコメントが全く足りていないので、適当にコメントを書いてコミットして下さい、それぐらい自分でやれよトロいなとか言われても何言われてもいいです。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

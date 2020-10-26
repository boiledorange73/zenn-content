---
title: "OverPass APIからデータを落としてPostGISに入れてみた"
emoji: "😀"
type: "tech"
topics: [PostGIS, OSM]
published: false
---
# はじめに

OpenStreetMapのデータを得られる OverPass API からデータが取れて、PostGISに叩き込んでみたので、どうやったかを記述します。

# OverPass API

http://wiki.openstreetmap.org/wiki/JA:Overpass_API に、読めばわかるように示されています。

クエリを投げるフォームを用意してくれています。http://overpass-api.de/query_form.html

## 自信が全くないクエリ解説

クエリはXMLベースで、要素の並び順にあわせて実行していきます。地理的範囲によるフィルタリングはbbox-queryで指定します。属性値によるフィルタリングは has-kv を使いますが、そこまで学習できていません。

```xml
<!-- 指定ボックス内の全てのノードを引っ張る -->
<bbox-query s="34.50" n="34.51" w="133.39" e="133.40"/>
<print/>
```

recurseでノードに対応したwayを引っ張ったり（type="node-way"）、参照を引くためのものです。

クエリやrecurseを実行するごとに以前のものがクリアされていきます。

たとえば、次のクエリを見てみましょう。

```xml
<!-- 指定ボックス内の全てのノードを引っ張る -->
<bbox-query s="34.50" n="34.51" w="133.39" e="133.40"/>
<!-- ノードに対応したウェイを引っ張る -->
<recurse type="node-way"/>
<print/>
```

bbox-queryによってnode要素が引っ張ってこられ、続いてreurseによってノードに対応したウェイを引っ張ってかれますが、node要素（位置情報がある）が消え、way要素だけになります。way要素はnode#idへの参照を持っていますが、位置情報自体は持っていないので、単独では位置が特定できません。

``<union>``でくくると、その子要素は全て結合されます。
次を見て下さい。

```xml
<!-- 1, 2 をまとめる -->
<union>
  <!-- 1. 指定ボックス内の全てのノードを引っ張る -->
  <bbox-query s="34.50" n="34.51" w="133.39" e="133.40"/>
  <!-- 2. ノードに対応したウェイを引っ張る -->
  <recurse type="node-way"/>
</union>
<print/>
```

これで、node要素とway要素の両方を持ったデータを得ることができます。

# osm2pgsql

得られたデータをPostGISで使用するために、osm2pgsqlというツールが提供されています。http://wiki.openstreetmap.org/wiki/Osm2pgsql あたりを参照。

GitHubに上げてくれているので、git clone https://github.com/openstreetmap/osm2pgsql で引っ張ってきます。

GEOSとProj.4が必要ですが、それさえあれば、順調に ccmake, gmake でビルドできました。

## osm2pgsqlでハマったこと

### 直接データベースに書き込みます

shp2pgsqlはシェープファイルをPostgreSQLのINSERTのリストまたはダンプ形式のデータを出力し、データベースは直接扱いません。パイプでpsqlに渡すか、ファイルにリダイレクトして、改めてpsqlに実行させるか、のいずれかです。

osm2pgsqlは、直接データベースに書き込みます。

これは、追加モードの場合に、以前にインポートしたデータと今回インポートするデータとで衝突する際の対応をしてくれるためです。

### ファイル名のサフィックスに注意

クエリを投げるフォームを使うと、少なくともFireFoxでは、そのまま保存すると interpretor というファイル名になります。これをそのままosmファイルとして指定すると、

  Osm2pgsql failed due to ERROR: Unknown file format 'auto'.

というメッセージを吐いて失敗します。

これは、ファイル名のサフィックスで書式を振り分けているためのようで、たとえば "interpritor.osm" というように、サフィックスに ".osm" を付けるとOKです。

### スリムモードでないとなんかうまくいかない

osmの地物データのうち、位置情報を含んでいるのはノード(node要素)であって、ラインストリング、ポリゴンに相当するもの(way要素)は、ノードのIDを参照しています。このため、PostGISの地物表現は、地物ごとに位置情報を埋め込んでいます。そのため、まずはノードを読んで、参照を読みながら、地物を作っていくようになっています。

デフォルトでは、どうもノードをメモリに置いておくようです。ちょっとしたファイルでもノード数は結構多いのか、キャッシュが足りないと言われ、インポートに成功したためしがありません。

osm2pgsqlには「スリムモード」というモードがあって、一時的な（名前付き）テーブルをデータベース内に作成して、そちらにノードを書き込んで、あらためて目的のテーブルにポイント、ラインストリング、ポリゴンを書き込むようにします。

ハナからあきらめてスリムモードにした方が良いかなと思います。


## osm2pgsqlのオプションなど

osm2pgsql -s -l -d (データベース名) (osmファイル名)

としています。

  - -s スリムモード
  - -l 経度・緯度で書き込み (デフォルトはEPSG:3857, WGS84球面メルカトル)

## cygwinでやってみる

http://wiki.openstreetmap.org/wiki/Osm2pgsql#Binary を参照。cygwin環境版を使えと言われたのでそうすることにしました。

https://vanguard.houghtonassociates.com/browse/OSM-OSM2PSQL-95/artifact から得られるzipファイルを展開すると、cygwin-package というフォルダができるので、そのフォルダを適当なところに置きます。とりあえずルートディレクトリの直下に置きます。

シェルを動かすと、PATHに、展開したディレクトリへのパスを追加します。この状態で実行すると、/usr/local/share/osm2pgsql/default.style が無いと怒られたので、ディレクトリを掘って、default.styleをcygwin-packageから複写しました。まとめると、次のようになるでしょう。

```bash
$ PATH=$PATH:/cygwin-package
$ mkdir -p /usr/local/share/osm2pgsql
$ cp /cygwin-package/default.style /usr/local/share/osm2pgsql
$ osm2pgsql.exe -H localhost -P 5432 -W -U postgres -s -d db interpreter-1.osm
```

osm2pgsqlの -W はパスワードの入力を受け付けるためものですが、パスワードがttyに表示されるので、驚かないでくださいと言われても驚く。

また、-l をつけると、``Projection code failed to initialise`` と怒られるので、EPSG:3857のデフォルトにしておきました。PostGISのST_Transformを使えばどうにかなるでしょう。

# おわりに

OSMのデータをいただいて、PostGISに叩き込んでみました。

Overpass APIは、今日触ったところで、本当にこれで説明になっているのか分からないので、謹んでつっこみを歓迎いたします。

次回は、このデータを使って何かやります。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

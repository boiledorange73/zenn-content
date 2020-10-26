---
title: "標準装備のQGISで三次元表示ができるよ!"
emoji: "😀"
type: "tech"
topics: [PostGIS, QGIS]
published: false
---
# はじめに

以前から、PostGISの三次元ジオメトリのビューアが欲しかったのです。三次元を実感するためには絶対に必要だと思っていました。特にPostGISのEWKBを直で読んでくれたらうれしいと感じていました。しかし、以前から探してはあきらめる、を繰り返していました。

今回、[FacebookのQGIS User Group Japan](https://www.facebook.com/groups/QGIS-User-Group-Japan-1270223339681777/)に、QGISの三次元マップビューが実装された、という記事 "[New 3D features in QGIS 3.4](https://www.lutraconsulting.co.uk/blog/2018/10/17/qgis3d-new-features-qgis3-4/?fbclid=IwAR0VMarXQ8CAPKG1us6IRCmSSqkyiUN3ji8DJPeOa_jG_t6xoIF5iEuDRPA)"が紹介されていました。

もうこんなの試すしかない、と思い、ざっと試してみました。

## 地形表示と地物表示

地形表示と地物表示に話を分けます。

地形表示は、DEMのラスタデータを引っ張ってきて地形とし、これに別の地図画像をドレープするものです。

地物表示は、三次元(XYZの頂点からなる)地物を表示する者です。

# 地形を反映した地図画像を表示する

https://www.lutraconsulting.co.uk/blog/2018/03/01/working-with-qgis-3d-part-1/ 参照。

## DEMの読み込み

ラスタデータベース配信サービス https://aginfo.cgk.affrc.go.jp/rstprv/index.html.ja では、国内の10m DEMを任意の範囲で切り取って取得することができますので、このへんから拾ってきて下さい。

WCSでやろうとするとQGISがダウンしてしまいました。

得られたDEMはラスタレイヤとして追加しておいて下さい（表示はしてもしなくても構いません）。レイヤとして追加しておかないと3Dマップビューで地形として利用できません。

## OSMタイルの登録

「ブラウザ」ペインの"XYZ Tiles"を右クリックして「新しい接続」を選択します。なお下図は既に"OpenStreetMap"があるのですが気にしないで下さい。

![「新しい接続」を選択しようとしているところ](https://storage.googleapis.com/zenn-user-upload/371aji2kfst31v92b2hhy267bx55)

「XYZ接続」ダイアログで、名前（任意）、URL、有効ズームレベル範囲を指定します。

![XYZ接続ダイアログ](https://storage.googleapis.com/zenn-user-upload/areujo0jid4ke7nyxst6108tfqrw)

## OSMタイルの表示

ひとたび登録しておくと、あとは、"OpenStreetMap"をダブルクリックするか、右クリックから「選択したレイヤをキャンパスに追加」を指定するだけです。

![「選択したレイヤをキャンパスに追加」を実行しようとしているところ](https://storage.googleapis.com/zenn-user-upload/joddld06dg36wowstookverkg9c0)

## 三次元表示

### 3Dマップビューを立ち上げる

「ビュー」メニューから「新しい3Dマップビュー」を選択します。

![「新しい3Dマップビュー」を選択しようとしているところ](https://storage.googleapis.com/zenn-user-upload/nkx4mv5sm7x4q6eoer74p800rssk)

操作してみて下さい。たとえば下図のように視点を変更することができます。なお、起伏は現時点ではありませんが気にしないで下さい。

![起伏が無い3D地図](https://storage.googleapis.com/zenn-user-upload/4e9b6z5356c2vt9w6jjamekri18f)

### demext.tifが標高データであることを指示する

3Dマップビューの上で設定アイコンをクリックします。

![3Dマップビューの設定アイコン](https://storage.googleapis.com/zenn-user-upload/m1shvs3hxzwmj3eukmhr9dv1q5vt)

3Dコンフィギュレーションが出ます。「高さ」でdemext.tifを指定すると、demext.tifが標高データとして反映されます。

なお"demext.tif"が存在しない場合には、DEMデータを得て、QGISにレイヤとして登録して下さい。

![3Dコンフィグレーションで鉛直スケールを選択しようとしているところ](https://storage.googleapis.com/zenn-user-upload/fpb3ijrl1vjq4pkaexmb2pstnod4)

高さのデータソースを指定すると、次のように地形が表示されます。

![3D地図](https://storage.googleapis.com/zenn-user-upload/b50hyrzduya5dqj4ajwhc03tm9x5)

蔵王山とかが美しく表示されました（どこやねん）。

# 立体地物を作成する

簡単に地物を一つ作ってみましょう。

## データを作る

### ST_Extrudeを使わない場合

とりあえず、次のようにするとデータを叩き込めます。``POLYHEDRALSURFACE``というタイプのジオメトリです。「多面体の表面」という意味です。Z値を持つマルチポリゴンと同じ構造ですが、「立体の表面を形成するもの」と「板の集合でたまたま立体の面を形成している」とは意味が違いますが、妥当性を厳密に評価する機能が無い限りは実質的な違いはありません。

```sql
CREATE TABLE cube (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(POLYHEDRALSURFACEZ, 3857)
);
```

```sql
INSERT INTO cube(geom)
SELECT 'SRID=3857;POLYHEDRALSURFACE(((14848227 4096370 0,14848227 4096390 0,14848247 4096390 0,14848247 4096370 0,14848227 4096370 0)),((14848227 4096370 10,14848247 4096370 10,14848247 4096390 10,14848227 4096390 10,14848227 4096370 10)),((14848227 4096370 0,14848227 4096370 10,14848227 4096390 10,14848227 4096390 0,14848227 4096370 0)),((14848227 4096390 0,14848227 4096390 10,14848247 4096390 10,14848247 4096390 0,14848227 4096390 0)),((14848247 4096390 0,14848247 4096390 10,14848247 4096370 10,14848247 4096370 0,14848247 4096390 0)),((14848247 4096370 0,14848247 4096370 10,14848227 4096370 10,14848227 4096370 0,14848247 4096370 0)))';
```

### ST_Extrudeを使う場合

上で提示した多面体サーフェスをポリゴンから作ってみましょう。

PostGIS SFCGALエクステンションにある``ST_Extrude()``で、ポリゴンから「押し出す」ことで立体を作ることができます。データベースにPostGIS SFCGALエクステンションを導入しなければなりません。

```sql
CREATE EXTENSION postgis_sfcgal;
```

```sql
CREATE TABLE bottom (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(POLYGON, 3857)
);
```

```sql
CREATE TABLE cube (
  gid SERIAL PRIMARY KEY,
  geom GEOMETRY(POLYHEDRALSURFACEZ, 3857)
);
```

```sql
INSERT INTO bottom(geom)
SELECT 'SRID=3857;POLYGON((14848227 4096370,14848247 4096370,14848247 4096390,14848227 4096390,14848227 4096370))'::GEOMETRY;

INSERT INTO cube (geom)
SELECT ST_Extrude(geom,0,0,10)
FROM bottom;
```

### QuickWKTを使う場合

QGISのWKTプラグインは``POLYHEDRALSURFACE Z``には対応していないので、``MULTIPOLYGON Z``に置き換えてみて下さい。前述の通り、意味は違いますが、表示だけなら、実質的に違いはありません。

```
MULTIPOLYGON Z(((14848227 4096370 0,14848227 4096390 0,14848247 4096390 0,14848247 4096370 0,14848227 4096370 0)),((14848227 4096370 10,14848247 4096370 10,14848247 4096390 10,14848227 4096390 10,14848227 4096370 10)),((14848227 4096370 0,14848227 4096370 10,14848227 4096390 10,14848227 4096390 0,14848227 4096370 0)),((14848227 4096390 0,14848227 4096390 10,14848247 4096390 10,14848247 4096390 0,14848227 4096390 0)),((14848247 4096390 0,14848247 4096390 10,14848247 4096370 10,14848247 4096370 0,14848247 4096390 0)),((14848247 4096370 0,14848247 4096370 10,14848227 4096370 10,14848227 4096370 0,14848247 4096370 0)))
```

# 三次元地物を表示する

https://medium.com/the-pointscene-diaries/qgis-3d-buildings-tutorial-1e0111fcd766 を参考にしました。

## 前提としてレイヤ追加

OSMのタイル地図画像、DEMとしてdemext.tif、地物データとしてcubeとが全てロードされているとします。なおdemextは表示させていません。

![cubeとOSMによる２Dの地図](https://storage.googleapis.com/zenn-user-upload/f1i5xb4gcu71a5phvpe2n9z22qzb)

## レイヤの三次元表示を有効にする

レイヤ→プロパティを順に選択して、レイヤのプロパティを表示します。
左側のペインから「3Dマップビュー」を選択します。
「3Dレンダラを有効にする」チェックボックスにチェックを入れます。

![レイヤプロパティで「3Dレンダラを有効にする」を有効にしたところ](https://storage.googleapis.com/zenn-user-upload/oxr6tprv90r4wokfebxiir5thtmo)

あと、色を変えたいなら「拡散」「アンビエント」等をいじってみて下さい。

## 3Dマップビューを表示する

3Dマップビューを上げると、三次元で地物が表示されます。

![地物を表示するが地形起伏のない3D地図](https://storage.googleapis.com/zenn-user-upload/jlx3roqgc9kgx6a2v6hfemqgm4bp)

DEMがあれば、地形も出せます。ただし、地物の鉛直方向の位置を決める際に、地形は考慮されないようです。

![地物を表示する3D地図](https://storage.googleapis.com/zenn-user-upload/0hb0s83g2s9adui357afzvjp7ozh)

ちょっと分かりにくいとは思いますが、地面が小高い丘の斜面になったぶん、箱型地物をめり込ませることができました。

# おわりに

いかがでしたでしょうか。

QGISで、地形表示は（DEMを持っているなら）それを3Dマップビューのプロパティで指定するだけで表示でき、地物表示はQGIS本体側のレイヤプロパティで「3Dレンダリングを有効にする」チェックボックスを入れるだけ、という簡単な手順でチャカチャカっと3Dマップビューが見られるようになりました。また、PostGISの多面体サーフェスを問題無く読み込んでくれています。

QGIS 3.4から標準装備についてくるようになってくれたので、今後は三次元データを（扱う必要があるなら）今よりも積極的に使えるようになるのではないでしょうか。

# 出典

* 3Dマップビューのドレープには、OpenStreetMapを使用しました。(C) OpenStreetMap contributors
* 地形データにはラスタデータ配信サービスを使用しました。出典: 農研機構 (https://aginfo.cgk.affrc.go.jp/rstprv/termsofuse.html.ja)

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

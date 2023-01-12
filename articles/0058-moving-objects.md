---
title: "Moving Objectsをやってみる"
emoji: "😀"
type: "tech"
topics: [GIS, PostGIS, WebSocket]
published: false
---

# はじめに

"PostGIS Day 2022" https://info.crunchydata.com/postgis-day-2022 の最後だけ参加したのですが、"Web Sockets and Real-Time Updates for Postgres with pg_eventserv" [YouTube](https://www.youtube.com/watch?v=Z_nOzHmpY8M) が紹介されていました。

文書による詳細説明としては https://www.crunchydata.com/blog/moving-objects-and-geofencing-with-postgres-postgis あたりを参考にして下さい。

ウェブアプリを開くと、3個の点があって、左端の矢印をクリックすると、その方向に合わせて点がリアルタイムで動きます。また、他の人が操作すると、それもリアルタイムで動きます。結構サクサク動いてくれます。

そして、とりあえずテストの URL が出るとみんな触りだす。みんなこういうのが好きなのね。

さらに、INSERT で新しい点を突っ込むと、それもウェブアプリに反映されます。

これを自分の手元で実行してみた記録です。サンプルとして公開されているので、そんなに難しくないです。

# とりあえずビルドする

## pg_eventserv

```tcsh
% git clone https://github.com/CrunchyData/pg_eventserv
% cd pg_eventserv
% gmake build
% cd ..
```

## pg_featureserv

pg_featureservはgoコンパイラが必要です。

```tcsh
% git clone https://github.com/CrunchyData/pg_featureserv
% cd pg_featureserv
% go build
% cd ..
```

## データベース作成

pg_eventserv/examples/moving-objects に移動します。

```
% cd pg_eventserv/examples/moving-objects
% vi moving-objects.sql
```

moving-objects.sql を編集します。"PERFORM SELECT" となっている箇所を "PERFORM" とします。

```sql
-- PERFORM SELECT pg_notify(channel, payload);
PERFORM pg_notify(channel, payload);
```

空間データベースを生成して、moving-objects.sql を実行して、テーブル、トリガ、関数等を作成します。

```tcsh
% createdb (db名)
% psql -d (db名) -c "CREATE EXTENSION postgis"
% psql -d (db名) -f moving-objects.sql
% cd ../../..
```

# サービスの起動

pg_featureserv と pg_eventserv を起動しなければなりません。

## ファイアウォール

その前に、ファイアウォールを使っていて、外部からアクセスしたい場合には、ポート **7700** と **9000** を開けておかなければなりません。pg_eventserv が 7700、pg_eventserv が 9000 を、それぞれ使います。

## pg_featureserv

```tcsh
% cd pg_featureserv
% setenv DATABASE_URL "postgres://(ユーザ名):(パスワード)@(ホスト名)/(db名)"
% ./pg_featureserv
```

## pg_eventserv

```tcsh
% cd pg_eventserv
% setenv DATABASE_URL "postgres://(ユーザ名):(パスワード)@(ホスト名)/(db名)"
% ./pg_eventserv
```

# ブラウザで開く

## Moving Objectsのクライアント

pg_eventservのサンプルとしてあります。

デフォルトではホスト名がlocalhostなので、変更する場合には、**2箇所**変更します。

```
cd pg_eventserv/examples/moving-objects/
vi moving-objects.js
```

```js
var wsHost = "ws://(ホスト名):7700";
...
var fsHost = "http://(ホスト名):9000";
```


``moving-objects.html``を開くと次のような画面が出ます。

![Moving Objectsウェブアプリ実行例](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0058/01-moving_objects.png)

# pg_featureserv は WFS にも対応しているっぽいが Read Only

pg_featureserv は WFSにも対応しているっぽくて、QGISで WFS レイヤが作れます。

QGISでWFS接続で http://(ホスト名):9000/ とすると、確かに見ることができます。

![QGISでWSFとして接続した例](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0058/02-qgis.png)

ただし、このWFSレイヤは編集可能になりません。ということで、Read OnlyではあるもののWFSに対応しているようです。

# Moving Objectの仕組み

## PostGISからブラウザ

PostGISからブラウザへのデータの流れは、pg_eventserv が提供する WebSocket を介して繋がります。

pg_eventserv の行っていることをもう少し詳しく言うと、PostgreSQLの機能である NOTICE で出た文字列を受け取って、それを WebSocket に流しています。

## ブラウザからPostGIS

pg_featureserv は、関数のAPI化もできます。

http://(ホスト名):9000/ にアクセスすると、"Collections" (テーブル)と"Functions"の一覧を見ることができます。

そして、"Functions"で示される関数はURLをたたくと呼び出せます。

たとえば

http://(ホスト名):9000/functions/postgisftw.object_move/items.json?direction=up&move_id=1

とすると、"Object 1" (赤い点)が北に1度移動します。

この関数は、たまたま UPDATE を実施する関数でしたが、何かの結果を得る関数でも問題ありません。

## 「ブラウザからPostGIS」と「PostGISからブラウザ」を繋げる

postgisftw.object_move関数を実行すると、"move_objects.objects"というテーブルの値が書き換わります。

このテーブルに**トリガを仕込んで**おき、値が書き換わると**NOTICEを発行**します。INSERT, DELETEもトリガを仕込んでいます。

objectsに変化があった際に発行される NOTICE を pg_eventserv が受け取って WebSocket に流すことになり、ウェブアプリのクライアントに反映されます。

ただし、NOTICE は文字列型でしか渡らないので、文字列に変換してから NOTICE に渡すことになります。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

---
title: "MySQLのaxis orderは「バグってる」わけではない"
emoji: "😀"
type: "tech"
topics: [MySQL, GIS]
published: false
---
# はじめに

随分と定着した FOSS4G Hokkaido の 2018年版 ( https://foss4g.hokkaido.jp/ ) があったそうで、相変わらず行かなかったのですが、Youtubeでの録画配信 ( https://www.youtube.com/watch?v=TvnlZJdFdX4 )を拝見しました。

MySQL 8のWKB/WKTのaxis orderに関して、ここの話が講演の中でご紹介頂きましたが、「バグってる」というのに違和感を感じました。

これは、私の二件の記事の書き方が悪かったものですので、弁明のようなかんじで、ざっと書いてみました。

## まず分けて考えよう

ちょっとややこしい話になってしまっているので、整理します。

WKB/WKTは、インタフェースの話です。要素間で仕様が合致していないとひどいことになる、というものです。そして、内部表現は、あくまで内部のことですので、どうやっていても他からあれこれ言うのは基本的に避けるべきだろうと思います。

[PostGISユーザがMySQL 8の空間拡張機能を少し触ってみた](h0043-mysqlgis)の「axis orderが違う」節については、WKB/WKTの話ですので、インタフェースの話です。で、ここではほとんど突っ込みを入れていませんが、完全にノーマークでした。

[MySQLの空間拡張機能のaxis orderについて](0045-mysqlgis-axisorder)は、内部表現の話です。開発チームが決定し、適宜改変していくものであり、あれこれ言うべきでないところです。とはいえ、なんとなく不安感を感じるところだったので開発チームで混乱するよ、という意味で書いています。「出過ぎたマネ」「余計なおせっかい」と言われたら「その通り」だと思います。

# WKB/WKTのaxis orderを変えることの是非

## どちらも誤りではない

一つ目の座標、二つ目の座標をどう取るかについて、二つの考え方があります。
左右/東西 を一つ目の座標とし、上下/南北を二つ目の座標とする考え方があります。この記事では「古い順序」とします。
一つ目の座標、二つ目の座標にそれぞれ何を取るかは、EPSGの登録に基づく、とする考え方があります。この記事では「新しい順序」とします。

この二つのどちらを選択しても、それを簡単に誤りであると言うべきではない、と個人的には考えます。
OGC標準を正義とする場合には「新しい順序」が正義です。これは間違いありません。「新しい順序」を選ぶべきです。でも、他のソフトウェアが全て「古い順序」の方を採用している状況だと、正義を叫んでも無力です。

### 「新しい順序」の根拠

OGCが公開している"Axis Order Policy and Recommendations"という文書に根拠を求めれば良いようです。
現版は"Revision to Axis Order Policy and Recommendations" (OGC Project Document 08-038r7)で、https://portal.opengeospatial.org/files/?artifact_id=76024 にdocxファイルがあります。

## じゃあどうすれば良いのかは答えないつもりか

はい答えません。
そもそも、そんな権限ありません。

その代わりに、昔話をしたいと思います。

## かつてaxis order変更の事例があった

Web Map ServiceというOGC標準があります。サーバが、GISソフト等のクライアントからHTTP(s)+クエリストリングでリクエストを受け取り、その通りの地図画像を作成して返すというものです。

あるサイトで、MapServerを使ったWMS準拠サービスを提供していましたが、バージョンを上げた途端、平面直角座標系（「古い順序」と「新しい順序」で差がある）で「壊れた」としか言いようがないような挙動を取るようになったと報告を受けました。

この問題はMapServer 6以降で、WMS 1.3.0準拠のサービスを出す際に、「古い順序」から「新しい順序」に変更したためです（なお、1.1.x準拠で出す場合には、「古い順序」のままです）。なお、やはりあまりにインパクトが強いだろう、と判断したMapServer開発チームは、当面は4000番台は「古い順序」のままとしました。現在では、MapServerは4000番台も含めて「新しい順序」に変更されています。

これは、サーバ側が「古い順序」から「新しい順序」にシフトし、クライアント側が後で追随した事例になります。

ここで余談ですが、レポートを受けた際には、もう本番サーバのアップデートをしてしまっている状態だったので、かなり焦りました。UTMは「古い順序」と「新しい順序」で差が出ないのと、EPSG:4326等は「古い順序」と「新しい順序」で差が出ないようにしていた（過去形）ので、余計に原因が分かりにくくなっていました。

その後、クライアントは、たとえばQGISの場合には、基本的に「新しい順序」に変更し、WMSサーバの新規登録時に「軸方位を逆にする」チェックボックスを導入し、サーバが「古い順序」になっている場合にチェックを入れると「古い順序」に対応できるようにしています。

対するサーバは、MapServerについてですが、WMS 1.1.1でリクエストを出すと「古い順序」となるようにしています。クライアントに手を入れられない場合には、WMSサーバの新規登録時にURLに"version=1.1.1"と、WMS 1.1.1でリクエストを出すよう強制すれば、ワークアラウンドとなります。

この事例については、サーバ側、クライアント側、共に歩み寄れたものと思います。個人的には「美しく決まった」と見ています。

が、こううまくいくとは限りません。WMSの場合には、1.1.1でのaxis orderは未定だったとかの抜け道があったのと、どちらも持ちつ持たれつの関係だったのが良い方に働いたのでしょうけど、MySQLは、RDBMSとしての実績は十分にあるのですが、ことspatialについてはニューカマーと言っていいでしょう（結構前から空間機能はあったけど）。MySQLの方だけが臥薪嘗胆の時期ととらえて、歩み寄らなければいけないかも知れません。

# まとめ

* 「古い順序」「新しい順序」どちらも間違いじゃないと考えた方が良い
* 「古い順序」から「新しい順序」に変わった事例はある
  * クライアント、サーバ双方が歩み寄った（と私は見ている）けど、そううまくいくとは限らない
* おざきさんの「yellow_73さんマジリスペクト」は、こそばゆくて鼻膨らませながら３度ほど繰り返し再生

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。
---
title: "QGISでPostGISのデータを見てみよう"
---
# はじめに

## postgresql.conf および pg_hba.conf の変更

リモートのPostGISに接続する場合には、サーバ側の postgresql.conf の変更 (listen_address) や pg_hba.conf の変更は行っておいて下さい。

ローカルだと問題ありません。

# QGIS

QGISはオープンソース(GPL)のGISです。本家サイト http://qgis.org/ からダウンロードします。

Windowsについては、他のオープンソースの地理空間情報ツール、ライブラリを依存関係を考慮してダウンロードしてくれる OSGeo4W http://trac.osgeo.org/osgeo4w/wiki/OSGeo4W_jp からも導入可能です。

# localhost の public.db.t1 に接続する

立ち上げたら、何も表示されていません。まずは PostGIS のデータベースに接続します。

まず、「レイヤ」メニューからPostGISレイヤの追加を選択します。

![「レイヤ」メニューを表示しているところ](https://storage.googleapis.com/zenn-user-upload/otxrgrlvigp8d4gpcyxzj1kjmzla)

## localhost の db に接続するための設定を行う

「PostGIS テーブルを追加」ダイアログには何もありません。ここで「新規」をクリックします。

![初期状態の「PostGIS テーブルを追加」ダイアログ](https://storage.googleapis.com/zenn-user-upload/bpieyseg2mhsre48j828k8o6erwd)

「新規 PostGIS 接続を作成」ダイアログが開きます。

![初期状態の「新規 PostGIS 接続を作成」ダイアログ](https://storage.googleapis.com/zenn-user-upload/q3t5sx5pqokrxjpniqb9jof39dly)

* 名称は自由に記入できます。ここでは db@localhost としています。
* ホスト、データベースは、サーバの状況に応じて記入して下さい。ここでは ホストは localhost とし、データベースは db としています。
* ポートは、デフォルトでは 5432 です。変更してある場合は、サーバの設定にあわせて下さい。
* SSLモード、ユーザ名、パスワード等は、サーバの状況に応じて記入して下さい。

![記入を済ませた後の「新規 PostGIS 接続を作成」ダイアログ](https://storage.googleapis.com/zenn-user-upload/mmjw7tyizbbr8nfrlow8hoysylz5)

この接続設定は保存されるので、次回以降はこの操作は不要です。

記入が終了すると、「PostGIS テーブルを追加」ダイアログのプルダウンに「db@localhost」というのが出ていると思います。複数の設定登録が可能です。

## 接続してレイヤを追加する

「PostGIS テーブルを追加」ダイアログのプルダウンに「db@localhost」が出ているのを確認して「接続」ボタンをクリックすると、下にジオメトリ型のカラムを持つテーブル一覧が表示されます。

![「PostGIS テーブルを追加」ダイアログのジオメトリ型のカラムを持つテーブル一覧](https://storage.googleapis.com/zenn-user-upload/uk5u98v1zb00qiv5jslnldgno8q5)

ここでは、t1を選択して、「追加」をクリックします。そうすると、メインウィンドウに、地図が表示されます。

![レイヤ追加直後の状態](https://storage.googleapis.com/zenn-user-upload/db5w25gmz881kny7tjbrabymsroc)

# QGISの基本操作

## 移動と拡大縮小
ウィンドウ上部にあるツールバーのうち、次に示すものに注目して下さい。

![操作モード切替ツールバー](https://storage.googleapis.com/zenn-user-upload/l9oaz5xlab36hrmhe5f5bh7l2sgw)

* 左から2番目にある「手のひら」（左端にあるピンチ操作アイコンではない）を選択すると、「パン」モードになり、マウスドラッグで地図を動かせます。
* 左から4番目の「むしめがね（＋）」で拡大モードに入ります。ドラッグした範囲にあわせて拡大してくれます。
* 左から5番目の「むしめがね（－）」は縮小モードで、拡大モードと同じ逆の操作で、逆の結果となります。

マウスホイールで拡大縮小が行えますので、パンとマウスホイールで概ねの操作は可能です。

## 指定したジオメトリのレコードを見る

「レイヤ」ペインで"t1"レイヤを選択し、「地物情報表示」モードに切り替え、ジオメトリをクリックすると、そこが選択され、「地物情報」ペインにレコードのデータが表示されます。

![「地物情報表示」の操作順序](https://storage.googleapis.com/zenn-user-upload/y0aqqbc0ax29tko0f1xraawx5qpf)

ペインはウィンドウにすることができるので、見にくい場合には、そのようにして下さい。

![ペインを切り離したところ](https://storage.googleapis.com/zenn-user-upload/bcvo2s2f4hulmvp89jr3xpi02kx1)

# ラベルを表示する

市区町村の区画だけ見ていても、どこが何市だ？といった情報はこの時点では表示できていません。

そこで、市区町村名のラベルを表示します。

市区町村名は、t1テーブルでは n03_004カラムにあります。

まず、レイヤペインでt1レイヤをマウス右ボタンで押さえ「プロパティ」を選択します。

![レイヤペインからレイヤのプロパティを選択しようとしているところ](https://storage.googleapis.com/zenn-user-upload/bfskzqd1cmijgc8bckw54jx1iytf)

「レイヤプロパティ」ダイアログが表示されます。今回はラベルを表示するので、「ラベル」タブを選択します。

![レイヤプロパティダイアログでラベルタブを選択しているところ](https://storage.googleapis.com/zenn-user-upload/7iqugpmvvg2ci1t4djkr9zh44is2)

「このレイヤのラベル」のチェックボックスにチェックを入れ、プルダウンからカラムを選択できるので、n03_004を選択します。

必要なら色や文字サイズ等の設定も変えられます。

![ラベルに使うカラムをn03_004に指定しているところ](https://storage.googleapis.com/zenn-user-upload/348o7k5df2nbg70rts8i5rkoi4w0)

「OK」で確定させると、地図上に市区町村名が表示されます。

![地図に市区町村名が表示されている図](https://storage.googleapis.com/zenn-user-upload/poe37fxwk4209k2ykt5ez5mb201d)

同一市区町村名が複数出てきていて見にくいかも知れませんが、とりあえずは市区町村名が表示されたことだけ確認して下さい。

# おわりに

QGISを使って、PostgreSQLサーバへの接続設定の作成、テーブルを指定した地図の表示、ジオメトリを選択してレコードデータを表示し、地図上にラベルを表示するところまで行いました。

PostGISはバックエンド担当なため、ここまで地図が全く出てこなかったのですが、QGISを使うと地図が表示できることが確認できたと思います。

QGISの使い方については、それなりにリソースがあるので、そちらを参考にして頂いた方がいいと思います。

# 出典

本記事では、国土交通省国土政策局が発行する国土数値情報（行政区域）を利用しました。

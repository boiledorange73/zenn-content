---
title: "シェープファイルのデータをインポートしてみよう GUI編"
---
# はじめに

「シェープファイルのデータをインポートしてみよう コマンドライン編」では、シェープファイルのデータをPostGISにインポートするために、コマンドラインからshp2gpsqlを使用しました。

PostGISには、GUIのインポータ/エクスポータも用意されているので、ここでは、それを使ってインポートしてみましょう。

# GUIインポータ/エクスポータ

PostGISのビルド時に、``--with-gui``を指定することで、GUIのインポータ/エクスポータもビルドされます(GTK+2.0が必要)。スタックビルダから入れた場合には、特に何もしなくてもついてきます。

shp2pgsqlは、シェープファイルからSQLファイルに変換する「コンバータ」ですが、こちらは、PostGISにインポートするところまでやってしまう「インポータ」です。振る舞いに違いがあることに注意して下さい。

# とりあえずデータを持ってくる

実験用データとして、国土数値情報 http://nlftp.mlit.go.jp/ksj/index.html から「行政区域」をもらってきましょう。以下の例では広島県のみを引っ張ってきています。ZIPでもらえます。これをunzipすると、xmlファイル（JPGIS 1.2形式)のほかに、shp, dbf, shx 等のファイルがあります。

# 順を追ってみていく

## 初期画面
"Shapefile and DBF Loader Exporter"を開くと、次のようなウィンドウが現れます。

![インポータ/エクスポータ初期状態](https://storage.googleapis.com/zenn-user-upload/0mrv97mmjd140k5axvt36v1koxtx)

## 接続設定

まず、接続設定を記入します。"View connection details."をクリックすると、次のようなダイアログが出るので記入します。pgAdminIIIのプラグインとして立ち上げた場合は、確認だけです。

![接続設定ダイアログ](https://storage.googleapis.com/zenn-user-upload/omfp8cbohp7efksg36dy80w6nxfk)

設定を記入したらOKを押してもとのウィンドウに戻ります。

データべスへのログインを試み、成功するとインポート作業に移ることができます。

## ファイル選択

"Add File"をクリックして、ファイル選択ダイアログを開き、インポートしたいファイルを選択します。

![ファイル選択ダイアログ](https://storage.googleapis.com/zenn-user-upload/zjfypm9rsph0w0nrxn8g1jinvw87)

もとのウィンドウに戻ると、フイルが選択されたことが確認できます。

![ファイルが選択されたところ](https://storage.googleapis.com/zenn-user-upload/0t9lamwri2i8lq05omrbuk9xyts6)

ウィンドウの横幅が急に広がっちゃってますが、気にしないでください。

## テーブル名設定

デフォルトでは、インポート先テーブル名は、ファイル名にされてしまいます。ここで、変更しちゃいましょう。

テーブル名の箇所を直接クリックして、適切な名前に変更して下さい。

![テーブル名を変更しているところ](https://storage.googleapis.com/zenn-user-upload/upbcwrlgvwct328mdlstxmi6vd2a)

それから、Enterを確実に押して、確定させてください。確定させないままウィンドウのフォーカスを外すとキャンセルされます。

![テーブル名変更を確定させたところ](https://storage.googleapis.com/zenn-user-upload/azibpav8q56f4r0jxsuwpamfthwc)

## SRIDを指定する

デフォルトでは、SRIDは0になっています。SRIDを指定しておかないと、あとあと面倒になるので、4612に変えておきましょう。

やり方はテーブル名指定と同じです。SRIDの箇所をクリックして編集可能にし、4612と入れます。

![SRIDを変更しているところ](https://storage.googleapis.com/zenn-user-upload/31ptrwnodsyt2qd76ix2kjs8hhlr)

Enterを確実に押して、確定させます。

![SRID変更を確定させたところ](https://storage.googleapis.com/zenn-user-upload/ttwhyjef0l3xyxgihck2vzniucc3)

## 文字コードを変える

"Options ..."をクリックして、次のダイアログを出し、先頭の"DBF file character encoding" を "UTF-8"から"cp932"にして下さい。

これはデータによって異なりますが、たいていの国内のデータはcp932と見ていいです。私見ですが、UTF-8もEUC-JPもISO-2022-JPも見たことないです。

![オプションダイアログでエンコーディングをcp932にしたところ](https://storage.googleapis.com/zenn-user-upload/8fmbjc7t4lrnoiv2bj9e7ws871vx)

## インポート

もとのウィンドウに戻って、"Import"ボタンをクリックします。

次のように、メッセージペインにインポートが成功した旨が表示されれば成功です。

![インポート成功](https://storage.googleapis.com/zenn-user-upload/003katik8zp02m7reso1y0a2g8er)

## 文字コード指定がおかしい場合

文字コードセットをUTF-8のままでじっこうした場合のログウィンドウは、次のようになります。

![エンコーディングエラー](https://storage.googleapis.com/zenn-user-upload/zaoahoqdfpmouqukdleejvsu4x4f)

``Unable to convert data to UTF-8 (iconv reports "Illegal byte sequence"). Current encoding is "UTF-8".``「UTF-8への変換ができません (iconvが「不正なバイト列」を報告しました)。現在のエンコーディングは"UTF-8"です」と言われています。

# おわりに

ここでは、シェープファイルのデータを、GUIツールを使ってインポートしてみました。

shp2pgsqlにGUIのガワをかぶせたにすぎないので、慣れると差が無いです。むしろスクリプトやバッチファイルで使うにはshp2pgsqlの方が便利です。ただ、GUIインポータの方が、インポート先テーブル名やSRID等が表として示してくれるので、途中の確認は容易になるかと思います。

shp2pgsqlとGUIインポータのいずれを使っても大丈夫ですので、お好きな方を使って下さい。

---
title: "インストールしてみよう Windows編"
---

# はじめに

まずはPostGISをインストールしなければなりません。

ここでは、Windows 10上で、PostgreSQLに同梱されているスタックビルダを用いたインストールを紹介します。


# PostgreSQLのダウンロード

PostgreSQLをインストールしたうえで、同梱されているスタックビルダを使ってPostGISをインストールします。

https://www.postgresql.org/download/windows/ で "Interactive installer by EDB" の節にある "Download the installer" をクリックし、EDB社サイトに行き、ダウンロードします。

ダウンロードしたexeファイルを走らせて、インストールを行ってください。

# PostgreSQLのインストール

最初にインストール先を選択します。デフォルト通りでいいと思います。

![インストール先選択ダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/pg-02.png)

次に同梱されているソフトウェアからインストールするものを選択します。全て入れることをお勧めします。最低でも "Stack Builder"は入れて下さい。

![同梱ソフト選択ダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/pg-03.png)

データディレクトリを選択します。これもデフォルト通りでいいと思います。

![データディレクトリ選択ダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/pg-04.png)

PostgreSQLインストール時には"postgres"という名前のユーザが作られますが、そのパスワードを設定します。このパスワードは忘れないようにして下さい。

![postgresパスワード設定ダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/pg-05.png)

サーバが待機するポートを指定します。これもデフォルト通りでいいと思います。

![サーバ待機ポート指定ダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/pg-06.png)

ロケール（言語）を指定します。デフォルトである"[Default locale]"を指定するべきです。

![ロケール指定ダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/pg-07.png)

インストール内容を確認するためのダイアログが表示されるので、間違いが無ければ（多分無いですが）"Next"をクリックします。

![インストール内容確認ダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/pg-08.png)

もう一回"Next"をクリックしてインストールを開始します。

![インストール開始確認ダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/pg-09.png)

インストールが終了しました。"Stack Builder may be used ..." と出ている箇所のチェックボックスがオンにすると、直後にスタックビルダが実行されるので、そうして下さい。

![インストール終了ウィンドウ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/pg-11.png)


# PostGISのインストール

PostgreSQLをインストーラから入れた場合には、上記の通り、インストール終了ウィンドウでチェックボックスをオンにすると、スタックビルダが起動できるので、起動させて下さい。

一旦PostgreSQLのインストールを終えてしまったなら、WindowsメニューのPostgreSQLフォルダ内にあるので、これをクリックして下さい。

![Windowsメニューを表示しているところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-01.png)

次のようなウィンドウが表示されます。

![スタックビルダの「ようこそ」画面](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-02.png)

# さっそくやってみましょう

最初にインストール先のサーバを指定します。通常なら、次の画面のように、5432番ポートにつなぐよう指定します。その後「次へ」ボタンをクリックします。

![インストール先設置](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-03.png)

インストールしたいアプリケーションとして、"Spatial Extensions"内にある"PostGIS"を選択します。その後「次へ」ボタンをクリックします。

![インストールしたいアプリケーションを指定したところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-05.png)

選択されたパッケージの表示を確認します。その後「次へ」ボタンをクリックします。

![選択されたパッケージ表示画面](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-06.png)

ダウンロードが行われます。

![ダウンロード中に表示される画面](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-07.png)

ダウンロードが終了したら「次へ」ボタンをクリックします。

![ダウンロード終了時の画面](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-08.png)

PostGISのセットアップが始まります。まずはライセンス (GPL)を確認したうえで"I Agree"ボタンをクリックします。

![PostGISセットアップの初期画面](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-09.png)

インストールする要素を選択します。通常は"PostGIS"だけにチェックを入れます。その後 "Next"ボタンをクリックします。

![インストール要素選択画面](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-10.png)

次にインストール先フォルダを指定します。通常は、PostgreSQLのインストール先です。その後 "Next"ボタンをクリックします。

![インストール先フォルダ指定画面](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-11.png)

インストールが終わると、数件のシステム設定変更を行うか確認されます。

環境変数``PROJ_LIB``（PROJのインストール先のパス）を設定するか尋ねられます。通常は「はい」をクリックします。``PROJ_LIB``を別に指定する場合には「いいえ」をクリックします。

![PROJ_LIBを設定するかどうかを確認するダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-13.png)

環境変数``GDAL_DATA``（GDALで使用する設定ファイル類のインストール先のパス）を設定するか尋ねられます。通常は「はい」をクリックします。``GDAL_DATA``を別に指定する場合には「いいえ」をクリックします。

![GDAL_DATAを設定するかどうかを確認するダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-14.png)

``POSTGIS_GDAL_ENABLED_DRIVERS``（有効なGDALドライバ）を設定するか尋ねられます。デフォルトでは全てのドライバは無効になります。「はい」をクリックすると、一部のドライバ（GeoTIFF, PNG, JPEG, XYZ, DTED, USGSDEM, AAIGrid）が有効になります。「いいえ」をクリックすると、デフォルト通り、全てのドライバが無効になります。GDALドライバを多く導入すると便利になる反面セキュリティホールになる可能性があります。PostGISをウェブサーバに繋げる場合には「いいえ」とした方が良いです。それ以外、特に一人だけで使用するような場合には「はい」としていいと思います。

![POSTGIS_GDAL_ENABLED_DRIVERSを設定するかどうかを確認するダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-15.png)

``POSTGIS_ENABLE_OUTDB_RASTERS``（データベース外ラスタを有効にするかどうか）を設定するか尋ねられます。これに``1``を設定すると、ラスタデータをテーブル内に格納せずに、データベースのデータとは全く別のファイルとして保存されます。デフォルトではデータベース外ラスタは無効です。

![POSTGIS_ENABLE_OUTDB_RASTERSを1に設定するかどうかを確認するダイアログ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-16.png)

これでPostGISのインストールが終了します。

![PostGISインストール終了画面](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-17.png)

PostGISインストールのウィンドウを閉じると、他にインストールするべきものが無ければ、スタックビルドも終了します。

![スタックビルダ終了画面](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/b1de0a18073af70946e0/install-sb/sb-18.png)

# おわりに

Windows 10上でのPostGISのインストール方法を示しました。しかしながら、現時点では、エクステンションの作成が可能になった状態です。

この次からは、実際にPostGISを使用します。


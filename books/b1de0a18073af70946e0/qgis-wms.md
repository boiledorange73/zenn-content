---
title: "WMSを使ってみる"
---
# はじめに

[WMTSで地域を絞り込んだうえでPostGISレイヤを表示する](qgis-wmts.md)でレイヤを追加した際に、WMSという語がちらっと出てきました。ここでは、QGISから、WMSに基づく地図配信サービスを併用して、地域を絞り込んだうえでPostGISデータを見る方法を紹介します。

## WMS配信サービスのURLについて

2020年4月1日に www.finds.jp は aginfo.cgk.affrc.go.jp に変更されています。
2016年4月1日に行われたFinds.jpサイトの大幅更新により、「基盤地図情報WMS配信サービス」は、他のWMS配信サービスと統合され、「地図画像配信サービス」の一部として公開されています。これにより、URL等が変更されています。この記事において以前の版で記載していたURLは無効になっていますので、ご注意下さい。

## WMS?

WMSはWeb Map Serviceの略で、HTTPを利用して、オンデマンドな地図画像配信を行うサービス仕様です。http://www.opengeospatial.org/standards/wms から、仕様書をダウンロードできます。

ここでは、[地図画像配信サービス](http://aginfo.cgk.affrc.go.jp/mapprv/index.html.ja)を使用します。[基盤地図情報](http://www.gsi.go.jp/kiban/)などを元データにした、WMSに基づく地図画像配信サービスを行っています。

# QGIS 2.4 からWMSを使う方法

## WMSサーバを登録する

とりあえずメニューから「レイヤ」→「WMS/WMTSレイヤの追加」を選択します。

![WMS/TMSレイヤの追加を選択しようとしている図](https://storage.googleapis.com/zenn-user-upload/qhcrpq99cwszqu1rn6jbx6pbd9nd)

「WM(T)Sサーバからレイヤを追加」ダイアログが現れます。

現時点では、WMSサーバは全く登録されていませんので、登録しましょう。
「新規」ボタンをクリックします。

![「WM(T)Sサーバからレイヤを追加」ダイアログ](https://storage.googleapis.com/zenn-user-upload/spcbq1vrjjrjmmvklv569o6cv58c)

「新しいWMSコネクションの作成」ダイアログが表示されます。通常、入力に必要なのは、「名称」と「URL」です。

名称は、ユーザが識別するために使用するもので、自由に記述できますが、自分にとって分かりやすくして下さい。

URLは、サービスポイントURLを指定します。今回は``http://aginfo.cgk.affrc.go.jp/ws/wms.php?``とします。

![「新しいWMSコネクションの作成」ダイアログで名称とURLを入力したところ](https://storage.googleapis.com/zenn-user-upload/xmorxv20udl2ka3nq8iuuoe6vgzu)

その他の設定は、とりあえず無視してOKだろうと思いますが、インターネットとの間にURL変換をするプロキシをかませている場合には、「capabilitiesで返答されたGetMap/GetTile URIを無視する」にチェックを入れると良いかと思います。

登録がうまくいくと、ひとつ前のダイアログに、名称が表示されていると思います。

![「Finds.jp WMS」というエントリが生成されたことを示す図](https://storage.googleapis.com/zenn-user-upload/bj9h7ktcq17stgc6g47wy31qlgoj)

## 接続する

「WM(T)Sサーバからレイヤを追加」ダイアログで「接続」ボタンをクリックします。

![「WM(T)Sサーバからレイヤを追加」ダイアログの「接続」ボタン](https://storage.googleapis.com/zenn-user-upload/7ledv0xiogt5jtbyyjthxntbo7o6)

中央部のリストにずらっと「何か」が表示されました。これらは、レイヤ一覧です。

![グループ化されたレイヤ一覧が表示されているところ](https://storage.googleapis.com/zenn-user-upload/urk489493gwcf5hwk3b2ngwzo95a)

基盤地図情報旧データ縮尺レベル25000を元データとしているレイヤグループは FGD25000 です。このグループには、行政区域、道路縁、等高線などのレイヤが存在します。

レイヤ行をクリックすると反転します。複数選択が可能ですが、今回はFGD25000グループのみ選択します。選択したうえで「追加」ボタンを押すと、レイヤ追加が実行されます。「閉じる」をクリックすると、レイヤ追加ダイアログが閉じられます。

![FGD25000を追加する順番を示した図](https://storage.googleapis.com/zenn-user-upload/fj1k91cr120xxsd8nmb5zl3z8bfj)

これで、日本全体が表示されます。

![日本全体を表示させているところ](https://storage.googleapis.com/zenn-user-upload/4ml8ou1ehskx9252ylzw2xtc2ux3)

## PostGISはできるだけ表示しないように操作する

QGISを操作して、見たい範囲だけを表示するようにします。

![広島県全体が表示される範囲を表示しているところ](https://storage.googleapis.com/zenn-user-upload/iku9dx7cpgrklpx5f8087kbg6rc8)

ここまで来たら、PostGISレイヤを追加します。

![PostGISレイヤを追加したところ](https://storage.googleapis.com/zenn-user-upload/ctsx8b4agljn54njjarnd2rdd5pi)

別の範囲に移りたい際には、いったん「レイヤ」ペインのPostGISレイヤのチェックボックスをオフにして、PostGISレイヤを表示しないようにします。

![PostGISレイヤを表示しないようにしたところ](https://storage.googleapis.com/zenn-user-upload/qqlowtc2osvbkwljvad1sxucrpbp)

この状態で移動します。

![和歌山県の一部が表示される範囲を表示しているところ](https://storage.googleapis.com/zenn-user-upload/t4hvxyb6bc6l86we8tq2x7kmv6z9)

あらためて、PostGISレイヤを表示させます。

![PostGISレイヤを表示させたところ](https://storage.googleapis.com/zenn-user-upload/0tftu9abn6v8x9kbdxa62x8qm6s6)

## 全体表示をしてみる

ある程度時間がかかりますので注意して下さい。

「レイヤ」ペインのPostGISレイヤを右クリックでおさえて、コンテキストメニューを出し、「レイヤの領域にズーム」を選択します。

![「レイヤの領域にズーム」を選択したところ](https://storage.googleapis.com/zenn-user-upload/evsdvxqsbohf8s9l8od6kf15a4lt)

しばらく、プログレスバーが動いて頑張ってロードします。しばらく待ちましょう。

![ウィンドウ下のプログレスバーが動いて頑張っているところ](https://storage.googleapis.com/zenn-user-upload/ll4dxcga9xly1dw53kcdaj88fgoj)

## 範囲を絞り込んでも全体表示と同じパフォーマンスになる場合

たぶん、ジオメトリに対して空間インデックスを貼っていないためです。

``CREATE INDEX <インデックス名> ON <テーブル名> USING GiST (<ジオメトリカラム名>);``を実行して下さい。

## 余談: QGIS 1.xではWMS 1.1.1を使用するべき

QGIS 1.xでは、``http://aginfo.cgk.affrc.go.jp/ws/kiban25000wms.cgi?VERSION=1.1.1&``としないと、平面直角座標系では正しく表示できない問題がありました。

MapServerでX,Yの順序が1.3準拠になっていなかったのが1.3準拠に変わり、対してQGIS 1.xでは1.3準拠でなかったことがありました。このため仕様の衝突があり、正しく表示できませんでした。

現在では解消しています。

# おわりに

前に説明したWMTSに基づくレイヤだけでなくWMSに基づくレイヤも使うことができることを示しました。また、QGIS 1.xについては、WMS 1.1.1を使用した方がいい場合があることも紹介しました。

# 出典

都道府県ポリゴン（基盤地図情報WMS配信サービスではない）には国土数値情報（行政区域）を使用しました。

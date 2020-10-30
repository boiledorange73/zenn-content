---
title: "基盤地図情報DEMをPostGISダンプ形式に変換する"
---
# はじめに

[PostGISラスタの仕様](pgisraster)、[PostGISラスタに加えることができる制約](constraint)等をもとに、基盤地図情報(http://www.gsi.go.jp/kiban/) の5mメッシュDEM、10mメッシュDEMをPostGISラスタのダンプ形式に出力するプログラムの紹介です。

## ソースコード

https://github.com/boiledorange73/fgddem2pgsql にあります。

## 必要なライブラリ

Expat (http://expat.sourceforge.net/) が必要です。

Shift_JIS文書に対応するために iconv が必要用に思うかも知れませんが、Shift_JISは無視しているので iconv は使っていません。

# fgddem2pgsql

```
fgddem2pgsql [-c|-a|-p] [-I] [-sd <スナップ値の逆数>] [-er <セル膨張率>] [-o <出力ファイル>] <テーブル名> <XMLファイル名> ...
```

* -c - テーブル作成モード。出力するスクリプト内に"CREATE TABLE"ステートメントと"COPY"ステートメントを書き込みます。-a, -pと共用できません。-c, -a, -pが指定されていない場合には、-cが指定されたのと同じになります。
* -a - データ追加モード。出力するスクリプト内に"COPY"ステートメントを書き込みます。テーブルは追加しません。-c, -pと共用できません。
* -p - 準備モード。出力するスクリプト内に"CREATE TABLE"ステートメントを書き込み、データ追加は行いません。-c, -aと共用できません。
* -I - 空間インデクスを追加します。
* -sd <スナップ値の逆数> - セルサイズ、左上隅位置をスナップする際に使用します。5mメッシュで18000、10mメッシュで9000となります。
* -er <セル膨張率> - セルサイズを膨らませるために使用します。''膨張セルサイズ=オリジナルセルサイズ*(1.0+<セル膨張率>)''です。10mメッシュで``1.0e-13''とするのがよいでしょう。5mメッシュでは、extentとsame_alignmentを両立するための値が発見できませんでした(3.7e-13がextentは成り立つがsame_alignmentが成り立たない、3.69e-13はextentが成り立たないがsame_alignmentが成り立つ)。
* -o <出力ファイル> - 出力ファイルを指定します。指定しない場合は標準出力に出力します。

fgddem2pgsql自体はワイルドカードは使用できません。だいたいのシェルはワイルドカードの機能は使えるので問題ありませんが、Windowsのコマンドプロンプトはダメです。

```csh
% fgddem2pgsql -c -sd 18000 -er 4.0e-13 -o snap.sql \
snap \
FG-GML-5133-63-00-DEM5A-20130702.xml \
FG-GML-5133-63-20-DEM5A-20130702.xml \
FG-GML-5133-63-01-DEM5A-20130702.xml \
FG-GML-5133-63-21-DEM5A-20130702.xml \
FG-GML-5133-63-11-DEM5A-20130702.xml
...

% fgddem2pgsql -c -o nosnap.sql \
nosnap \
FG-GML-5133-63-00-DEM5A-20130702.xml \
FG-GML-5133-63-20-DEM5A-20130702.xml \
FG-GML-5133-63-01-DEM5A-20130702.xml \
FG-GML-5133-63-21-DEM5A-20130702.xml \
FG-GML-5133-63-11-DEM5A-20130702.xml
...

% psql -d db -f snap.sql
...
% psql -d db -f nosnap.sql
...
```

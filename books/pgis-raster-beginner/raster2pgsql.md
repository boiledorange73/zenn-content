---
title: "PostGISラスタにGeoTIFFデータを格納する"
---
# はじめに

ラスタデータのフォーマットでよく使われる GeoTIFF から SQL を生成して、PostGISラスタにデータを格納してみます。

## gdalを使用します

PostGISをラスタ機能付きでビルドする場合にはgdalが必要なので、どこかにgdalが転がっていると思います。

gdalは、フォーマットの相互変換やリサンプリング、マージ等といったラスタデータ操作を行うライブラリとツール群からなります。

本章ではgdalのツールを使います。

## GeoTIFFデータの取得

[ラスタデータ配信サービス](http://aginfo.cgk.affrc.go.jp/rstprv/)で、任意の範囲のDEM (デジタル標高モデル)データをGeoTIFF等で得ることができます。

「切り取りアプリ」を使って、地図アプリケーション上から取得します。

本章で使用するデータは http://aginfo.cgk.affrc.go.jp/rstprv/rstext.html.ja?lon=133.38388888889068&lat=34.385666666666815&zoom=12&layers=BTT&clon=133.3838888888889&clat=34.385666666666665&anchor=cc&iw=600&ih=400&it=image/tiff から得ています。

# GeoTIFFデータの確認

データのダウンロードまでできたとして、``gdalinfo``で、データを確認してみます。

```
% gdalinfo demext.tif 
Driver: GTiff/GeoTIFF
Files: demext.tif
       demext.tif.aux.xml
Size is 600, 400
Coordinate System is:
GEOGCRS["JGD2000",
    DATUM["Japanese Geodetic Datum 2000",
        ELLIPSOID["GRS 1980",6378137,298.257222101004,
            LENGTHUNIT["metre",1]]],
    PRIMEM["Greenwich",0,
        ANGLEUNIT["degree",0.0174532925199433]],
    CS[ellipsoidal,2],
        AXIS["geodetic latitude (Lat)",north,
            ORDER[1],
            ANGLEUNIT["degree",0.0174532925199433]],
        AXIS["geodetic longitude (Lon)",east,
            ORDER[2],
            ANGLEUNIT["degree",0.0174532925199433]],
    ID["EPSG",4612]]
Data axis to CRS axis mapping: 2,1
Origin = (133.350555555559993,34.407888888888998)
Pixel Size = (0.000111111111100,-0.000111111111113)
Metadata:
  AREA_OR_POINT=Area
  TIFFTAG_RESOLUTIONUNIT=2 (pixels/inch)
  TIFFTAG_XRESOLUTION=72
  TIFFTAG_YRESOLUTION=72
Image Structure Metadata:
  INTERLEAVE=BAND
Corner Coordinates:
Upper Left  ( 133.3505556,  34.4078889) (133d21' 2.00"E, 34d24'28.40"N)
Lower Left  ( 133.3505556,  34.3634444) (133d21' 2.00"E, 34d21'48.40"N)
Upper Right ( 133.4172222,  34.4078889) (133d25' 2.00"E, 34d24'28.40"N)
Lower Right ( 133.4172222,  34.3634444) (133d25' 2.00"E, 34d21'48.40"N)
Center      ( 133.3838889,  34.3856667) (133d23' 2.00"E, 34d23' 8.40"N)
Band 1 Block=600x3 Type=Float32, ColorInterp=Gray
  Min=0.000 Max=347.400
  Minimum=0.000, Maximum=347.400, Mean=59.674, StdDev=95.060
  NoData Value=-9999
  Metadata:
    STATISTICS_MAXIMUM=347.39999389648
    STATISTICS_MEAN=59.674224998563
    STATISTICS_MINIMUM=0
    STATISTICS_STDDEV=95.060453340889
    STATISTICS_VALID_PERCENT=100
```

ピクセル数、空間参照系、ピクセルの原点に対応する地理的位置、解像度（単位/ピクセル）、バンド数とバンドのタイプ等が出てきます。

# データベースを作りエクステンションを導入する

``createdb``の後、SQLの``CREATE EXTENSION``で、エクステンションを導入します。この際、PostGIS 2.xでは``CREATE EXTENSION postgis``で、ラスタ機能も追加されましたが、PostGIS 3.xではPostGISエクステション導入の後に``CREATE EXTENSION postgis_raster``でラスタ機能を導入します。具体的には、次のようにします。

```
% createdb raster
% psql -d raster
raster=# CREATE EXTENSION postgis;
CREATE EXTENSION
raster=# CREATE EXTENSION postgis_raster;
CREATE EXTENSION
raster=# 
```

# とりあえず叩き込む

## SQLを生成する

PostGISにはraster2pgsqlという、SQLデータを生成するツールが同梱されています。shp2pgsqlみたいなものですね。raster2pgsqlを使ってみます。

次のようになります。

```
raster2pgsql (TIFFファイル名) (テーブル名) > (SQLファイル名)
```

出力ファイル名の指定ができないので、リダイレクトで保存して下さい。

## 出てきたSQLを眺める

生成されたSQLを見てみましょう。

まず、``BEGIN``があり、続いて``CREATE TABLE``があります。

```
CREATE TABLE "dem" ("rid" serial PRIMARY KEY,"rast" raster);
```

ここでは``dem``というテーブルを生成するようになっていますが、``raster2pgsql``を実行する際に指定したものですものです。カラムは"rid", "rast"だけです。

その後は ``INSERT``でデータを格納する文が並びます。そして最後に``END``があります。

## 入れるのは楽

そのまま``psql``に流せばOKです。

```
% psql -d (データベース名) -f (SQLファイル名)
```

# PostGISラスタの確認

## gdalツールで指定する「データソース」

一般的には「データソース」と「ファイル名」とは同じであると考えられているかと思います。

gdalで通常データソースを指定する所には、ファイル以外のチャネルに対応できるようにしています。PostGISのテーブルをデータソースとして開きたいときには、データソースを指定する箇所で、ファイル名の代わりに次のような文字列を指定します。

```
PG:dbname=(データベース名) table=(テーブル名) port=(ポート番号) user=(ユーザ名) password=(パスワード)
```

## gdalinfoによる確認

本章で示している例では、データベース名が``raster``で、テーブル名が``dem``ですので、そのように指定して``gdalinfo``でデータを見てみましょう。なお、下の実行結果例では警告が出ていますが、とりあえずは無視します。

```
% gdalinfo 'PG:dbname=raster table=dem'
Warning 1: Cannot find (valid) information about public.dem table in raster_columns view. The raster table load would take a lot of time. Please, execute AddRasterConstraints PostGIS function to register this table as raster table in raster_columns view. This will save a lot of time.
Driver: PostGISRaster/PostGIS Raster driver
Files: none associated
Size is 600, 400
Coordinate System is:
BOUNDCRS[
    SOURCECRS[
        GEOGCRS["JGD2000",
            DATUM["Japanese Geodetic Datum 2000",
                ELLIPSOID["GRS 1980",6378137,298.257222101,
                    LENGTHUNIT["metre",1]]],
            PRIMEM["Greenwich",0,
                ANGLEUNIT["degree",0.0174532925199433]],
            CS[ellipsoidal,2],
                AXIS["geodetic latitude (Lat)",north,
                    ORDER[1],
                    ANGLEUNIT["degree",0.0174532925199433]],
                AXIS["geodetic longitude (Lon)",east,
                    ORDER[2],
                    ANGLEUNIT["degree",0.0174532925199433]],
            ID["EPSG",4612]]],
    TARGETCRS[
        GEOGCRS["WGS 84",
            DATUM["World Geodetic System 1984",
                ELLIPSOID["WGS 84",6378137,298.257223563,
                    LENGTHUNIT["metre",1]]],
            PRIMEM["Greenwich",0,
                ANGLEUNIT["degree",0.0174532925199433]],
            CS[ellipsoidal,2],
                AXIS["latitude",north,
                    ORDER[1],
                    ANGLEUNIT["degree",0.0174532925199433]],
                AXIS["longitude",east,
                    ORDER[2],
                    ANGLEUNIT["degree",0.0174532925199433]],
            ID["EPSG",4326]]],
    ABRIDGEDTRANSFORMATION["Transformation from JGD2000 to WGS84",
        METHOD["Position Vector transformation (geog2D domain)",
            ID["EPSG",9606]],
        PARAMETER["X-axis translation",0,
            ID["EPSG",8605]],
        PARAMETER["Y-axis translation",0,
            ID["EPSG",8606]],
        PARAMETER["Z-axis translation",0,
            ID["EPSG",8607]],
        PARAMETER["X-axis rotation",0,
            ID["EPSG",8608]],
        PARAMETER["Y-axis rotation",0,
            ID["EPSG",8609]],
        PARAMETER["Z-axis rotation",0,
            ID["EPSG",8610]],
        PARAMETER["Scale difference",1,
            ID["EPSG",8611]]]]
Data axis to CRS axis mapping: 2,1
Origin = (133.350555555559993,34.407888888888998)
Pixel Size = (0.000111111111100,-0.000111111111113)
Corner Coordinates:
Upper Left  ( 133.3505556,  34.4078889) (133d21' 2.00"E, 34d24'28.40"N)
Lower Left  ( 133.3505556,  34.3634444) (133d21' 2.00"E, 34d21'48.40"N)
Upper Right ( 133.4172222,  34.4078889) (133d25' 2.00"E, 34d24'28.40"N)
Lower Right ( 133.4172222,  34.3634444) (133d25' 2.00"E, 34d21'48.40"N)
Center      ( 133.3838889,  34.3856667) (133d23' 2.00"E, 34d23' 8.40"N)
Band 1 Block=600x400 Type=Float32, ColorInterp=Gray
  NoData Value=-9999
```

これで、データの内容はともかく、位置や解像度等は正しいことが確認できます。

# QGISで見てみよう

QGISのメニューから「レイヤ」→「レイヤを追加」→「ラスタレイヤを追加」と選択すると、追加したいラスタファイル名を求めるダイアログが現れます。

ただ、ファイル名が求められているようです。

これだと、できないかも知れませんね。PostGISはファイルではないですから。

いや、できるかも知れません。

## QGISのデータソースもファイル以外に対応

というのも、QGISもgdalを持っています。ということは、ファイル名を指定するべきところで``PG:``で始まる接続文字列を入れると動くのではないか、と考えられないでしょうか？

ためしにリモートで接続してみました。この際にputtyでフォワーディングさせていたためポート番号は5432ではないし、ユーザ名もパスワードも必要となりました。

それでも無理ではありませんでした。次のように、ポート番号やユーザ認証情報もサポートされています。

```
PG:dbname=(データベース名) table=(テーブル名) port=(ポート番号) user=(ユーザ名) password=(パスワード)
```

ダイアログは次のようになります。

![PostGISラスタをデータソースに指定しているところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-raster-beginner/raster2pgsql/1.png)

なんか違和感を感じます。

## 表示できた

せっかくなのでPostGISラスタをデータソースとして指定してQGISで表示させた結果を示します。

![PostGISラスタをQGISで表示しているところ](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-raster-beginner/raster2pgsql/2.png)


# おわりに

本章では、GeoTIFFデータからSQLを生成してPostGISラスタに保存し、gdalinfoで確認し、QGISで表示する手順を紹介しました。

また、PostGISデータをgdalで使用する場合の接続文字列の概要と、gdalだけでなくQGISも接続文字列に対応していることを紹介しました。

これで、PostGISラスタについて、格納、表示という、基礎的なことができることが確認できました。

---
title: "インストールしてみよう FreeBSD編"
---

# はじめに

まずはPostGISをインストールしなければなりません。

ここでは、FreeBSD 12上で、Packages, Portsを用いたインストールを紹介します。


# PostGISのインストール

本書記述時点では PostGIS 3.1 が最新ですので、これをインストールします。
依存パッケージである PostgreSQL や GDAL 等がインストールされます。

```
# pkg install postgis31
```

# PostgreSQLのセットアップ

だいたい、次のようなかんじになります。

```
# mkdir ~postgres/data12
# chown -R postgres:postgres  ~postgres/data12
# sudo -u postgres /usr/local/bin/initdb \
  --no-locale --encoding=UTF8 \
  -D  ~postgres/data12 \
  -L /usr/local/share/postgresql
```

また、``/etc/rc.conf`` に次を入れます。

```
postgresql_enable="YES"
```


# gdalのインストール

## Packagesから入れる

次に念のためコマンドを示します。

```
# pkg install gdal
```

GDALが対応するフォーマットは次のコマンドで確認できます。

```
% gdal_translate --formats
  VRT -raster,multidimensional raster- (rw+v): Virtual Raster
  DERIVED -raster- (ro): Derived datasets using VRT pixel functions
  GTiff -raster- (rw+vs): GeoTIFF
  COG -raster- (wv): Cloud optimized GeoTIFF generator
  NITF -raster- (rw+vs): National Imagery Transmission Format
  RPFTOC -raster- (rovs): Raster Product Format TOC format
  ECRGTOC -raster- (rovs): ECRG TOC format
  HFA -raster- (rw+v): Erdas Imagine Images (.img)
  SAR_CEOS -raster- (rov): CEOS SAR Image
  CEOS -raster- (rov): CEOS Image
  JAXAPALSAR -raster- (rov): JAXA PALSAR Product Reader (Level 1.1/1.5)
  GFF -raster- (rov): Ground-based SAR Applications Testbed File Format (.gff)
  ELAS -raster- (rw+v): ELAS
  AIG -raster- (rov): Arc/Info Binary Grid
  AAIGrid -raster- (rwv): Arc/Info ASCII Grid
  GRASSASCIIGrid -raster- (rov): GRASS ASCII Grid
  ISG -raster- (rov): International Service for the Geoid
  SDTS -raster- (rov): SDTS Raster
  DTED -raster- (rwv): DTED Elevation Raster
  PNG -raster- (rwv): Portable Network Graphics
  JPEG -raster- (rwv): JPEG JFIF
  MEM -raster,multidimensional raster- (rw+): In Memory Raster
  JDEM -raster- (rov): Japanese DEM (.mem)
  GIF -raster- (rwv): Graphics Interchange Format (.gif)
  BIGGIF -raster- (rov): Graphics Interchange Format (.gif)
  ESAT -raster- (rov): Envisat Image Format
  BSB -raster- (rov): Maptech BSB Nautical Charts
  XPM -raster- (rwv): X11 PixMap Format
  BMP -raster- (rw+v): MS Windows Device Independent Bitmap
  DIMAP -raster- (rov): SPOT DIMAP
  AirSAR -raster- (rov): AirSAR Polarimetric Image
  RS2 -raster- (rovs): RadarSat 2 XML Product
  SAFE -raster- (rov): Sentinel-1 SAR SAFE Product
  PCIDSK -raster,vector- (rw+v): PCIDSK Database File
  PCRaster -raster- (rw+): PCRaster Raster File
  ILWIS -raster- (rw+v): ILWIS Raster Map
  SGI -raster- (rw+v): SGI Image File Format 1.0
  SRTMHGT -raster- (rwv): SRTMHGT File Format
  Leveller -raster- (rw+v): Leveller heightfield
  Terragen -raster- (rw+v): Terragen heightfield
  ISIS3 -raster- (rw+v): USGS Astrogeology ISIS cube (Version 3)
  ISIS2 -raster- (rw+v): USGS Astrogeology ISIS cube (Version 2)
  PDS -raster- (rov): NASA Planetary Data System
  PDS4 -raster,vector- (rw+vs): NASA Planetary Data System 4
  VICAR -raster,vector- (rw+v): MIPL VICAR file
  TIL -raster- (rov): EarthWatch .TIL
  ERS -raster- (rw+v): ERMapper .ers Labelled
  L1B -raster- (rovs): NOAA Polar Orbiter Level 1b Data Set
  FIT -raster- (rwv): FIT Image
  GRIB -raster,multidimensional raster- (rwv): GRIdded Binary (.grb, .grb2)
  JPEG2000 -raster,vector- (rwv): JPEG-2000 part 1 (ISO/IEC 15444-1), based on Jasper library
  RMF -raster- (rw+v): Raster Matrix Format
  MSGN -raster- (rov): EUMETSAT Archive native (.nat)
  RST -raster- (rw+v): Idrisi Raster A.1
  INGR -raster- (rw+v): Intergraph Raster
  GSAG -raster- (rwv): Golden Software ASCII Grid (.grd)
  GSBG -raster- (rw+v): Golden Software Binary Grid (.grd)
  GS7BG -raster- (rw+v): Golden Software 7 Binary Grid (.grd)
  COSAR -raster- (rov): COSAR Annotated Binary Matrix (TerraSAR-X)
  TSX -raster- (rov): TerraSAR-X Product
  COASP -raster- (ro): DRDC COASP SAR Processor Raster
  R -raster- (rwv): R Object Data Store
  MAP -raster- (rov): OziExplorer .MAP
  KMLSUPEROVERLAY -raster- (rwv): Kml Super Overlay
  PDF -raster,vector- (w+): Geospatial PDF
  CALS -raster- (rwv): CALS (Type 1)
  SENTINEL2 -raster- (rovs): Sentinel 2
  MRF -raster- (rw+v): Meta Raster Format
  PNM -raster- (rw+v): Portable Pixmap Format (netpbm)
  DOQ1 -raster- (rov): USGS DOQ (Old Style)
  DOQ2 -raster- (rov): USGS DOQ (New Style)
  PAux -raster- (rw+v): PCI .aux Labelled
  MFF -raster- (rw+v): Vexcel MFF Raster
  MFF2 -raster- (rw+): Vexcel MFF2 (HKV) Raster
  FujiBAS -raster- (rov): Fuji BAS Scanner Image
  GSC -raster- (rov): GSC Geogrid
  FAST -raster- (rov): EOSAT FAST Format
  BT -raster- (rw+v): VTP .bt (Binary Terrain) 1.3 Format
  LAN -raster- (rw+v): Erdas .LAN/.GIS
  CPG -raster- (rov): Convair PolGASP
  IDA -raster- (rw+v): Image Data and Analysis
  NDF -raster- (rov): NLAPS Data Format
  EIR -raster- (rov): Erdas Imagine Raw
  DIPEx -raster- (rov): DIPEx
  LCP -raster- (rwv): FARSITE v.4 Landscape File (.lcp)
  GTX -raster- (rw+v): NOAA Vertical Datum .GTX
  LOSLAS -raster- (rov): NADCON .los/.las Datum Grid Shift
  NTv1 -raster- (rov): NTv1 Datum Grid Shift
  NTv2 -raster- (rw+vs): NTv2 Datum Grid Shift
  CTable2 -raster- (rw+v): CTable2 Datum Grid Shift
  ACE2 -raster- (rov): ACE2
  SNODAS -raster- (rov): Snow Data Assimilation System
  KRO -raster- (rw+v): KOLOR Raw
  ROI_PAC -raster- (rw+v): ROI_PAC raster
  RRASTER -raster- (rw+v): R Raster
  BYN -raster- (rw+v): Natural Resources Canada's Geoid
  ARG -raster- (rwv): Azavea Raster Grid format
  RIK -raster- (rov): Swedish Grid RIK (.rik)
  USGSDEM -raster- (rwv): USGS Optional ASCII DEM (and CDED)
  GXF -raster- (rov): GeoSoft Grid Exchange Format
  NWT_GRD -raster- (rw+v): Northwood Numeric Grid Format .grd/.tab
  NWT_GRC -raster- (rov): Northwood Classified Grid Format .grc/.tab
  ADRG -raster- (rw+vs): ARC Digitized Raster Graphics
  SRP -raster- (rovs): Standard Raster Product (ASRP/USRP)
  BLX -raster- (rwv): Magellan topo (.blx)
  SAGA -raster- (rw+v): SAGA GIS Binary Grid (.sdat, .sg-grd-z)
  IGNFHeightASCIIGrid -raster- (rov): IGN France height correction ASCII Grid
  XYZ -raster- (rwv): ASCII Gridded XYZ
  HF2 -raster- (rwv): HF2/HFZ heightfield raster
  OZI -raster- (rov): OziExplorer Image File
  CTG -raster- (rov): USGS LULC Composite Theme Grid
  E00GRID -raster- (rov): Arc/Info Export E00 GRID
  ZMap -raster- (rwv): ZMap Plus Grid
  NGSGEOID -raster- (rov): NOAA NGS Geoid Height Grids
  IRIS -raster- (rov): IRIS data (.PPI, .CAPPi etc)
  PRF -raster- (rov): Racurs PHOTOMOD PRF
  SIGDEM -raster- (rwv): Scaled Integer Gridded DEM .sigdem
  CAD -raster,vector- (rovs): AutoCAD Driver
  GenBin -raster- (rov): Generic Binary (.hdr Labelled)
  ENVI -raster- (rw+v): ENVI .hdr Labelled
  EHdr -raster- (rw+v): ESRI .hdr Labelled
  ISCE -raster- (rw+v): ISCE raster
```

コンパイルオプションは次の通りです。

```
# pkg info gdal
gdal-3.1.4_3
...
Options        :
        ARMADILLO      : off
        CFITSIO        : off
        CHARLS         : off
        CRYPTOPP       : off
        CURL           : off
        DODS           : off
        ECW            : off
        EXPAT          : off
        EXR            : off
        FREEXL         : off
        GEOS           : off
        GTA            : off
        HDF5           : off
        JASPER         : on
        KML            : off
        LIBXML2        : off
        MYSQL          : off
        NETCDF         : off
        ODBC           : off
        OPENJPEG       : off
        PCRE           : off
        PGSQL          : off
        PODOFO         : off
        POPPLER        : off
        SFCGAL         : off
        SPATIALITE     : off
        SQLITE         : off
        TILEDB         : off
        WEBP           : off
        XERCES         : off
        ZSTD           : off
```

## 個人的にはPortsから入れています

フォーマットや対応機能で不満があるなら、Portsから入れて下さい。ただし、ビルドには結構時間がかかります。

筆者は次のオプションを付けています。

+ CURL
+ EXPAT
+ GEOS
+ JASPER
+ KML
+ LIBXML2
+ NETCDF
+ OPENJPEG
+ PCRE
+ PGSQL
+ POPPLER
+ SFCGAL
+ SPATIALITE
+ SQLITE
+ WEBP
+ XERCES
+ ZSTD

これにより、対応フォーマットで次のものが追加されました。

```
GMT -raster- (rw): GMT NetCDF Grid Format
netCDF -raster,multidimensional raster,vector- (rw+s): Network Common Data Format
JP2OpenJPEG -raster,vector- (rwv): JPEG-2000 driver based on OpenJPEG library
WCS -raster- (rovs): OGC Web Coverage Service
WMS -raster- (rwvs): OGC Web Map Service
WEBP -raster- (rwv): WEBP
Rasterlite -raster- (rwvs): Rasterlite
MBTiles -raster,vector- (rw+v): MBTiles
PLMOSAIC -raster- (ro): Planet Labs Mosaics API
WMTS -raster- (rwv): OGC Web Map Tile Service
PostGISRaster -raster- (rws): PostGIS Raster driver
RDA -raster- (ro): DigitalGlobe Raster Data Access driver
EEDAI -raster- (ros): Earth Engine Data API Image
DAAS -raster- (ro): Airbus DS Intelligence Data As A Service driver
GPKG -raster,vector- (rw+vs): GeoPackage
PLSCENES -raster,vector- (ro): Planet Labs Scenes API
NGW -raster,vector- (rw+s): NextGIS Web
HTTP -raster,vector- (ro): HTTP Fetching Wrapper
```

こういったものが必要かどうかは検討して下さい。前述の通り、コンパイル時間が結構長くなるので、検討した結果必要でないと判断するならPackagesから入れた方がいいです。

ビルド終了後には、次の通り、Packages版を消してPorts版を入れなおすようにしています。

```
# make deinstall
# make reinstall
```

# おわりに

FreeBSD上でのPostGISのインストール方法を示しました。しかしながら、現時点では、エクステンションの作成が可能になった状態です。

この次からは、実際にPostGISを使用します。


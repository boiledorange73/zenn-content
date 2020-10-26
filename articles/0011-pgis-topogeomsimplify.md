---
title: "トポジオメトリを使うと隙間や重なりの無いポリゴン簡略化が比較的簡単にできる"
emoji: "😀"
type: "tech"
topics: [PostGIS]
published: false
---
# はじめに

ジオメトリ (およびジオグラフィ)は、他のジオメトリやジオグラフィとの関係を考慮していません。

このため、簡略化を行うと、たとえば接している2つのポリゴンが隙間なく接していたのが、隙間や重なりができてしまい、元の接続関係が維持されなくなります。

これを解決するには、PostGISが持つ「トポロジ」という機能を使いますが、結構面倒なので、ジオメトリとトポロジとの中間に立つ「トポジオメトリ」という型が用意されています。

今回は、トポジオメトリを使って、隙間や重なりのないポリゴン簡略化をやってみます。

## トポロジ

トポロジは、ノード (点)、エッジ (線)、フェイス (面)という3つのテーブルを持っていて、エッジの端点はノードで、フェイスの境界はエッジといった具合に、ノード、エッジ、フェイスには相互関係があるのですが、その関係を記憶しています。たとえば、フェイスは、自らの境界となるエッジのID一覧を持っています。エッジは、自らの端点となるノードのID一覧を持っています。

また、逆方向の関係データも持っていて、エッジは、自分を境界にしているフェイスの一覧を記憶しています。

## トポジオメトリ

トポロジは、ノード、エッジ、フェイスおよびこれらの関係を記録していますが、別のスキーマで一括して保存しているので、テーブルのカラムとしてトポロジ型を置けません。

このため、テーブルのカラムにジオメトリ型を追加して、そのタプルに地物と属性を保存する、というモデルになっていません。

トポロジからジオメトリを生成するために必要な情報を抜き出すデータ型が用意されています。これをカラムに追加すれば、ジオメトリのように扱うことができるでしょう。

このデータ型の名前がトポロジとジオメトリとをあわせたトポジオメトリです。覚えやすいといえば覚えやすいですね。

トポジオメトリ型はジオメトリ型とは異なるものですが、キャストで簡単にジオメトリにできるようにしてくれてあります。

## トポロジの操作は直接は行いません

トポロジに自前でノードを追加したり、エッジを追加したり、削除したり、ノードを移動したり、といったことは、今回はしません。もしやりたいのでしたら、たとえば、次のページを参照して下さい。

* http://strk.kbt.io/projects/postgis/Paris2011_TopologyWithPostGIS_2_0.pdf
* http://d.hatena.ne.jp/yellow_73/20120311

特に1つ目は、トポロジはどういうもので、どういうデータ構造をとっているか、といったことを書いてくれています。

# 前提

まず、次のようなテーブル w_mncpl があるとします。

```sql
-- データは自分で入れてください
CREATE TABLE w_mncpl (
  gid SERIAL PRIMARY KEY,
  mcode INT, -- 市区町村コード
  geom GEOMETRY(POLYGON, 4612) -- ポリゴン
);
```

# （対照として）ジオメトリの簡略化

トポロジ/トポジオメトリを使わない簡略化を行い、何が問題かを確認します。簡略化パラメータは100mとします。

```sql
CREATE TABLE geomtable_3857 (
  gid SERIAL PRIMARY KEY,
  mcode INT,
  geom GEOMETRY(POLYGON, 3857)
);
CREATE INDEX ix_geomtable_3857_mcode ON geomtable_3857 (mcode);
CREATE INDEX ix_geomtable_3857_geom ON geomtable_3857 USING GiST(geom);
```

```sql
INSERT INTO geomtable_3857 (mcode, geom)
SELECT mcode, ST_Simplify(ST_Transform(geom,3857), 100) FROM w_mncpl;
```

これを図示すると次のようになります。

![良くない簡略化地図](https://storage.googleapis.com/zenn-user-upload/e574k7l3794ioq3e1sqimcztrlsb)

問題ないように見えますね。でもこれを拡大してみましょう。

![良くない簡略化地図の一部拡大表示](https://storage.googleapis.com/zenn-user-upload/rhg82hlvwsspaigruvzemsgd27uh)

隙間ができてしまいました。

塗りつぶしの不透過度を下げてみましょう。

![良くない簡略化地図の一部拡大して不透過度を下げて表示](https://storage.googleapis.com/zenn-user-upload/tbirbwfw9wjon59mcsnlo8ks1w60)

状況がもう少し複雑で、隙間のほか、重なりも確認できました。

# トポロジを作りレイヤを作る

## エクステンションのインストール

トポロジはPostGISとは分離されているので、導入したいデータベースで次を実行して、トポロジを導入します。

```sql
CREATE EXTENSION postgis_topology;
```

## トポロジを作る

```
SELECT topology.CreateTopology('topo_jp_3857', 3857);

 createtopology
----------------
              3
(1 行)
```

CreateTopology()の返り値は、生成されたトポロジのIDです。たぶんこのデータベースでは過去に2つほどトポロジが作られていたのでしょう。

トポロジIDは使わないので、気にしないでおきましょう。

もしトポロジID等を得たいなら、次のようなクエリを投げます。

```sql
SELECT * FROM topology.topology;
```

たとえば、次のような結果が得られます。

```
 id |     name     | srid | precision | hasz
----+--------------+------+-----------+------
  2 | topo_jp      | 4612 |         0 | f
  3 | topo_jp_3857 | 3857 |         0 | f
(2 行)
```

このデータベースでは、前に``topo_jp``というトポロジを作っていたのですね（しらじらしい）、あと、id=1のトポロジは、たぶん一度作られて、消されたのでしょう（正直覚えてません）。

## トポジオメトリを作る/レイヤを作る

```
CREATE TABLE topogeom_jp_3857 (
  gid serial primary key,
  mcode INT
);

SELECT topology.AddTopoGeometryColumn(
  'topo_jp_3857',
  'public', 'topogeom_jp_3857', 'topo',
  'POLYGON'
);

 addtopogeometrycolumn
-----------------------
                     1
(1 行)
```

```sql
SELECT * FROM topology.layer;
```

```
 topology_id | layer_id | schema_name |    table_name    | feature_column | feature_type | level | child_id
-------------+----------+-------------+------------------+----------------+--------------+-------+----------
           3 |        1 | public      | topogeom_jp_3857 | topo           |            3 |     0 |
(1 行)
```

トポロジIDが3でレイヤIDが1のレイヤに関するものがあります。

ここでいうレイヤは、ジオメトリカラムに対するgeometry_columnsのようなテーブルです。しかし、トポジオメトリカラムのジオメトリタイプとSRIDだけでなく、トポロジとの接続がなされている点が違います。

## トポジオメトリとしてジオメトリを叩き込む

```sql
INSERT INTO topogeom_jp_3857(mcode, topo)
SELECT mcode, topology.toTopoGeom(ST_Transform(geom, 3857), 'topo_jp_3857', 1)
FROM w_mncpl;
```

### 非常に時間がかかります

ご注意ください。上記のクエリは、非常に時間がかかります。手持ちの機械では、国土数値情報（行政区域）全体で、2日以上かかりました。

# トポジオメトリの簡略化

トポジオメトリを使った簡略化を行います。簡略化パラメータは、トポロジ/トポジオメトリを使わない簡略化と同じ100mとします。

まず、結果を格納するsimplified_3857というテーブルを作ります。ジオメトリタイプはマルチポリゴンとします。

```sql
CREATE TABLE simplified_3857 (
  gid SERIAL PRIMARY KEY,
  mcode INT,
  geom GEOMETRY(MULTIPOLYGON, 3857)
);
CREATE INDEX ix_simplified_3857_mcode ON simplified_3857 (mcode);
CREATE INDEX ix_simplified_3857_geom ON simplified_3857 USING GiST(geom);
```

簡略化をしながら挿入していきます。

```sql
INSERT INTO simplified_3857 (mcode, geom)
SELECT mcode, ST_Simplify(topo,100)
FROM topogeom_jp_3857 ORDER BY gid;
```

ここでのポイントは、トポジオメトリを引数に取る``ST_Simplify`` (簡略化関数)がある、ということです。たとえば http://www.finds.jp/docs/pgisman/2.5.0/TP_ST_Simplify.html を参照して下さい。

これで、何も考えずにトポロジを基にした簡略化が簡単にできるようになっています。

# 見よトポジオメトリのちからを

さきほどと同じ地域を見てみます。

![トポロジによる簡略化の結果](https://storage.googleapis.com/zenn-user-upload/91v0vpkvmfsy8jq3nxltnhxngxwv)

隙間はありません。

塗りつぶしの不透過度を下げて、重複を見てみましょう。

![トポロジによる簡略化の結果を不透過度を下げて表示](https://storage.googleapis.com/zenn-user-upload/flduumybswkvx2ek8hx69z14moo2)

何も変わりないように見えますが、これを求めていたのです、重複が無いのが確認できました。

# おわりに

ジオメトリの簡略化では隙間や重複ができること、トポロジによってジオメトリ間の関係性を保存できること、トポジオメトリを使うとトポロジをより簡単に使えることを示し、PostGISトポロジによって、隙間や重複のない簡略化が行えることを示しました。

また、ジオメトリからトポロジを生成する（今回はトポジオメトリを介して生成しています）際には非常に長い時間がかかることも触れています。特にトポロジ構築には時間がかかるので、2日かかるとかとなっても、じっと我慢するか、全国で無く特定県のみに限定する等してデータ数を減らす、といった対応を検討して下さい。

# 出典

本稿の画像は、国土数値情報（行政区域）の平成27年データを使用ました。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

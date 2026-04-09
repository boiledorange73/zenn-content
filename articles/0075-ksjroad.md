---
title: "国土数値情報（道路データ）をPostGISに取り込んでpgRoutingでゴニョゴニョする"
emoji: "😀"
type: "tech"
topics: [国土数値情報,PostGIS,pgRouting,トポロジー]
published: false
---

# はじめに

# 生データを叩きこむ

## フォルダを掘る

ファイル数が多くなるのでフォルダを掘っておきます (必須ではありません)。

```
mkdir zip
mkdir shp
mkdir sql
```

## ダウンロード

https://nlftp.mlit.go.jp/ksj/ からダウンロードします。シェープファイル版を落としました。

ダウンロードしたファイルは ZIP になっているので、``zip`` フォルダに入れておきます。

## ZIPファイルの展開

**この節では Cygwin bash を使っています**

これらをunzipするのですが、``shp``に移動してから``unzip``を実行した方がやりやすいです。次のようにします

```
cd shp

for f in ../zip/*.zip; do
  unzip $f
done
```

ここでzipごとにフォルダが掘られて、その下にファイルが展開されちゃった場合は、配下のファイルを上に上げて、空になったフォルダを消します。

```
mv */*.shp .
mv */*.shx .
mv */*.dbf .
mv */*.prj .
mv */*.cpg .
mv */*.xml .
rmdir *_SHP
```

最後に、``shp``フォルダから抜けます。

```
cd ..
```

## シェープからPGDUMPの作成

``ogr2ogr`` を使ってPGDUMPデータを作成してきます。

```
cd sql

for f in ../shp/*.shp
do
  body=`echo $f|sed -E 's#^\.\./shp/(.*)\.shp$#\1#'`
  out="${body}.sql"
  echo "$out" 2>&1
  ogr2ogr -f PGDUMP \
    --config PG_USE_COPY YES \
    -lco CREATE_TABLE=NO \
    -lco GEOMETRY_NAME=geom \
    -oo ENCODING=UTF-8 \
    -nln road \
    "$out" "$f"
done
```

これで ``sql`` フォルダ内にメッシュごとの PGDUMP ファイルができます。ただし ``CREATE TABLE``などはありません。

次に、ファイルを ``../data.sql`` に結合します。

追加リダイレクションを使うので ``../data.sql`` を消してから、追加していきますが、ファイルをソートしています。その方が地物が近くなるかなと思ったためです。位置的に近い地物が近い行にいる方がインデックスが効率的になるかなと思ったからです。

```
rm -f ../data.sql

/bin/ls *.sql|sort|tr -d '\r'|while read -r f
do
  echo "$f" 2>&1
  cat "$f" >> ../data.sql
done
```

カレントフォルダを一つ上に戻します。

```
cd ..
```

# テーブル作成とデータ格納

テーブル作成のために ``tables.sql`` を作成します。次の SQL たちをコピペします。

```
BEGIN;

DROP TABLE IF EXISTS road CASCADE;
DROP TABLE IF EXISTS road_type CASCADE;
DROP TABLE IF EXISTS road_category CASCADE;
DROP TABLE IF EXISTS road_state CASCADE;
DROP TABLE IF EXISTS road_width CASCADE;
DROP TABLE IF EXISTS road_toll CASCADE;

CREATE TABLE road_type (
  id INT PRIMARY KEY,
  name TEXT
);

CREATE TABLE road_category (
  id INT PRIMARY KEY,
  name TEXT
);

CREATE TABLE road_state (
  id INT PRIMARY KEY,
  name TEXT
);

CREATE TABLE road_width (
  id INT PRIMARY KEY,
  name TEXT
);

CREATE TABLE road_toll (
  id INT PRIMARY KEY,
  name TEXT
);

COPY road_type (id, name) FROM stdin;
1	通常部
2	庭園路
3	徒歩道
4	石段
5	不明
\.

COPY road_category (id, name) FROM stdin;
1	国道
2	都道府県道
3	市区町村道等
4	高速自動車国道等
5	その他
6	不明
\.

COPY road_state (id, name) FROM stdin;
1	通常部
2	橋・高架
3	トンネル
4	雪囲い
5	建設中
6	その他
7	不明
\.

COPY road_width (id, name) FROM stdin;
1	3m未満
2	3m-5.5m未満
3	5.5m-13m未満
4	13m-19.5m未満
5	19.5m以上
6	不明
\.

COPY road_toll (id, name) FROM stdin;
1	無料
2	有料
\.

CREATE TABLE road (
  gid SERIAL PRIMARY KEY,
  n13_001 TIMESTAMP,
  n13_002 INT REFERENCES road_type(id), -- 道路の種別
  n13_003 INT REFERENCES road_category(id), -- 道路の分類
  n13_004 INT REFERENCES road_state(id), -- 道路の状態
  n13_005 INT, -- 階層順
  n13_006 INT REFERENCES road_width(id), -- 幅員区分
  n13_007 INT REFERENCES road_toll(id), -- 有料区分
  n13_008 VARCHAR(6), -- 二次メッシュ番号
  geom GEOMETRY (LINESTRING, 6668)
);
CREATE INDEX ON road USING GiST(geom);
CREATE INDEX ON road (n13_002);
CREATE INDEX ON road (n13_003);
CREATE INDEX ON road (n13_004);
CREATE INDEX ON road (n13_005);
CREATE INDEX ON road (n13_006);
CREATE INDEX ON road (n13_007);

CREATE OR REPLACE VIEW v_road AS
SELECT gid,
  n13_001,
  n13_002, road_type.name road_type_name,
  n13_003, road_category.name road_category_name,
  n13_004, road_state.name road_state_name,
  n13_005,
  n13_006, road_width.name road_width_name,
  n13_007, road_toll.name road_toll_name,
  n13_008,
  geom
FROM road
  LEFT JOIN road_type ON n13_002=road_type.id
  LEFT JOIN road_category ON n13_003=road_category.id
  LEFT JOIN road_state ON n13_004=road_state.id
  LEFT JOIN road_width ON n13_006=road_width.id
  LEFT JOIN road_toll ON n13_007=road_toll.id;
  
END;
```

データベース作成、エクステンションの作成をしていきます。

データベース名は ``ksj`` にしています。

```
createdb -U postgres ksj
psql -U postgres -d ksj -c "CREATE EXTENSION postgis"
```

次に ``tables.sql`` と ``data.sql`` を順次実行します。

```
psql -U postgres -d ksj

ksj=> \i table.sql
ksj=> \i data.sql
```

ここまでで道路データの格納が完了しています。また、属性に代表値が使われていますが、``v_road``ビューでは、属性を説明する日本語文字列も見ることができます。

QGISで一部を見たら次のようになります。

![道路データをQGISで表示した図](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0075/01-roads.png)

いい眺めですね。

# トポロジーを構築する

国土数値情報（道路データ）はかなりきれいなデータです。立体交差している道路には交点が入っていませんし、平面交差している道路には交点があります。なので、立体交差を考慮したトポロジーを構築することは可能です。

ただし、シェープファイルではトポロジー情報を持っていません。

じゃあ PostGISトポロジーを使おうかというと、あれはノード、エッジ、フェイスの空間次元2次元向けのもので、ネットワークトポロジーは扱え切れません。

ということで自分でどうにかしたいと思います。

むずかしくありません、各ラインストリングの両端点をノードとして登録して、そのノードのIDとラインストリングとを結び付けてあげるだけです。

ただし、ラインストリング同士が端点をノードとして共有するために、ノードが重なる場合には一方のノードだけを存続させ、もう一方は存続させてはいけません。

なお、確認していませんが、ラインストリングの端点同士が全く同じ数値になってるんじゃないかと思えるレベルで作られていますが、念のため、10cm (0.1m) を許容範囲としてました。この際の距離計測はジオグラフィを使ってします。

## テーブルだけ作成

まずテーブルだけ作成します。インデックスは ``roadarc.geog`` だけ後で付けます。

```
CREATE TABLE roadarc (
  gid BIGINT PRIMARY KEY,
  nid1 BIGINT, -- source
  nid2 BIGINT, -- target
  cost DOUBLE PRECISION,
  reverse_cost DOUBLE PRECISION,
  geog GEOGRAPHY (LINESTRING, 6668)
);

CREATE TABLE roadnode (
  gid SERIAL PRIMARY KEY,
  geog GEOGRAPHY (POINT, 6668)
);
CREATE INDEX ON roadnode USING GiST(geog);
```

## アークの準備

アークとして、ラインストリングの挿入、コスト (ここでは距離)を設定します。

なお ``reverse_cost`` は逆方向に進む場合のコストで、今回は順方向のコストと同じコストとしています。

```
-- 2 min 25 secs.
INSERT INTO roadarc (gid, geog)
SELECT gid, geom::GEOGRAPHY FROM road
ORDER BY gid;
-- 5 min
UPDATE roadarc SET cost = ST_Length(geog);
-- 3 min 15 sec
UPDATE roadarc SET reverse_cost = cost;
```

ここで ``roadarc.geog`` にインデックスを付けます。大量挿入後にインデックスを付けた方が効率が良くなりそうだからです。

```
-- 5 min 15 secs
CREATE INDEX ON roadarc USING GiST (geog);
-- 54 secs 455 msec
VACUUM (ANALYZE) roadarc;
```

## ノード作成、アークに登録する関数 10cmトレランス版

指定したラインストリングの両端点を抽出して、roadnode に既に登録されている (近いものも含む)場合には、その ID (''gid'') を端点ノードIDとします。登録されていない場合には、新規に追加して、新たに生成された ``gid`` を端点ノードIDとします。

アークの各行について 始端のノードIDは ``nid1``、終端のノードIDは ``nid2`` にそれぞれ格納されます。

```
CREATE OR REPLACE FUNCTION road_mknode_geog10(in a_gid BIGINT, a_geog GEOGRAPHY(LINESTRING, 6668))
RETURNS BOOLEAN AS $$
DECLARE
  v_n1 GEOGRAPHY(POINT, 6668);
  v_n2 GEOGRAPHY(POINT, 6668);
  v_nid1 INTEGER;
  v_nid2 INTEGER;
  v_linestring GEOGRAPHY(LINESTRING, 6668);
  v_nrow roadnode%ROWTYPE;
  v_gid INTEGER;
BEGIN
  v_n1 := ST_StartPoint(a_geog::GEOMETRY)::GEOGRAPHY;
  SELECT gid INTO v_nid1 FROM roadnode WHERE ST_DWithIn(geog, v_n1, 0.1) LIMIT 1;
  IF v_nid1 IS NULL THEN
    INSERT INTO roadnode(geog) VALUES(v_n1) RETURNING gid INTO v_nid1;
  END IF;
  v_n2 := ST_EndPoint(a_geog::GEOMETRY)::GEOGRAPHY;
  SELECT gid INTO v_nid2 FROM roadnode WHERE ST_DWithIn(geog, v_n2, 0.1) LIMIT 1;
  IF v_nid2 IS NULL THEN
    INSERT INTO roadnode(geog) VALUES(v_n2) RETURNING gid INTO v_nid2;
  END IF;
  UPDATE roadarc SET nid1 = v_nid1, nid2 = v_nid2 WHERE roadarc.gid=a_gid;
  RETURN TRUE;
END;
$$ LANGUAGE plpgsql;
```

アークの行ごとに処理を実行します。

```
-- 33 min 08 sec
SELECT road_mknode_geog10(gid, geog) FROM roadarc ORDER BY gid;

-- 2 sec
ANALYZE roadnode;
```

### 余談: インデックスを使わないと

``roadnode.geog`` のインデックス付け忘れで、ジョブを走らせて放置してたら、15時間超えても終わらなかったです。

最初にインデックスを付けるより、大量挿入後にインデックスを付ける方が効率が良くなるので、忘れるのはまあありえます。とはいえ、インデックス作成をしないのはよろしくないですね。

最後に、``nid1``、``nid2`` のそれぞれにインデックスと ``REFERENCES``制約 とを付けます。

```
-- 56 sec
CREATE INDEX ON roadarc (nid1);
-- 20 sec
CREATE INDEX ON roadarc (nid2);
-- 1 min 30 sec
ALTER TABLE roadarc ADD FOREIGN KEY (nid1) REFERENCES roadnode(gid);
ALTER TABLE roadarc ADD FOREIGN KEY (nid2) REFERENCES roadnode(gid);
-- 2 sec
ANALYZE roadarc;
```

## チェック

まずは、道路アークと、道路ノードの数を見てみましょう。

```
SELECT COUNT(*) FROM roadarc;

  count
----------
 24065831
(1 row)

SELECT COUNT(*) FROM roadnode;

  count
----------
 18609265
(1 row)
```

続いて、いろいろチェックしていきます。

```
-- roadnode,geog が全部ちゃんとあるか? (0ならいい)
SELECT COUNT(*) FROM roadnode WHERE geog IS NULL;

-- 全ラインストリングに始端・終端ノードが乗っているか? (0ならOK)
SELECT COUNT(*) FROM roadarc WHERE nid1 IS NULL;
SELECT COUNT(*) FROM roadarc WHERE nid2 IS NULL;

-- nid1/nid2 が roadnode.gid に無いことはないか (リファレンス) → 出力行なしならいい。
SELECT roadnode.gid, nid1 FROM roadarc LEFT JOIN roadnode ON nid1=roadnode.gid WHERE roadnode.gid IS NULL;
SELECT roadnode.gid, nid2 FROM roadarc LEFT JOIN roadnode ON nid2=roadnode.gid WHERE roadnode.gid IS NULL;

-- ループしてない? → ループするラインストリングは存在する。
-- 13sec
SELECT COUNT(*) FROM roadarc WHERE nid1 = nid2;
 count
-------
 23547
(1 行)

-- ループを QGIS などで見たいなら次のビューを使用
CREATE VIEW v_roadarc_loop AS SELECT gid, geog FROM roadarc WHERE nid1=nid2;
```

# pgRouting

## 最短経路探索

最短経路探索を行います。まずは結果を格納するテーブルを作っておきます。

```
CREATE TABLE pgr_dijkstra_result (
  seq INT,
  path_seq INT,
  start_vid BIGINT,
  end_vid BIGINT,
  node BIGINT,
  edge BIGINT,
  cost DOUBLE PRECISION,
  agg_cost DOUBLE PRECISION,
  geog GEOGRAPHY(LINESTRING, 6668)
);
CREATE INDEX ON pgr_dijkstra_result USING GiST(geog);
```

ノード番号は次のものを使います (必ずお手持ちの環境でノード番号を確認してくだ足)

- 4077827 -- 福山駅北口のスクランブル交差点
- 4139969 -- 182と408の交差点フジグラン神辺に近い

```
DELETE FROM pgr_dijkstra_result;

-- 2 secs 845 msec./144
WITH params AS (
  SELECT 4077827::BIGINT AS nid1, 4139969::BIGINT AS nid2
)
INSERT INTO pgr_dijkstra_result(seq,path_seq,start_vid,end_vid,node,edge,cost,agg_cost,geog)
SELECT
  r.seq,
  r.path_seq,
  r.start_vid,
  r.end_vid,
  r.node,
  r.edge,
  r.cost,
  r.agg_cost,
  (SELECT geog FROM roadarc WHERE gid=r.edge)
FROM params
CROSS JOIN LATERAL pgr_dijkstra(
  format($$
    SELECT
      gid AS id,
      nid1 AS source,
      nid2 AS target,
      cost,
      reverse_cost
    FROM roadarc
    WHERE ST_DWithin(
      geog,
      (SELECT geog FROM roadnode WHERE gid = %s),
      2.0 * ST_Distance(
        (SELECT geog FROM roadnode WHERE gid = %s),
        (SELECT geog FROM roadnode WHERE gid = %s)
      )
    )
  $$, params.nid1, params.nid1, params.nid2),
  params.nid1,
  params.nid2,
  directed := true
) AS r;
```

結果は次の通りです。

![福山駅北口から国道182号と国道486号との交差点の間の最短経路の計算結果をQGISで表示した図](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0075/02-shortest.png)

## 到達圏の探索

今回は、福山駅前通りと2号との交差点から 5000m 圏内の行程を探索していきます。

結果テーブルを作成します

```
CREATE TABLE pgr_drivingdistance_result (
  seq INT,
  depth INT,
  start_vid BIGINT,
  pred BIGINT, -- redecessor of node (ひとつ前のノード)
  node BIGINT,
  edge BIGINT,
  cost DOUBLE PRECISION,
  agg_cost DOUBLE PRECISION,
  geog GEOGRAPHY(LINESTRING, 6668)
);
CREATE INDEX ON pgr_drivingdistance_result USING GiST(geog);
```

次のノードIDを使います。

- 4079954 -- 福山駅前通りと2号との交差点

```
DELETE FROM pgr_drivingdistance_result;

-- 0.742 s / 14531
WITH params AS (SELECT 4079954::BIGINT nid)
INSERT INTO pgr_drivingdistance_result(seq,depth,start_vid,pred,node,edge,cost,agg_cost,geog)
SELECT r.seq,
  r.depth,
  r.start_vid,
  r.pred,
  r.node,
  r.edge,
  r.cost,
  r.agg_cost,
  (SELECT geog FROM roadarc WHERE gid=edge)
FROM params
CROSS JOIN LATERAL pgr_drivingDistance(
  format('
    SELECT gid id, nid1 AS source, nid2 AS target, cost, reverse_cost FROM roadarc WHERE ST_DWithIn(
      geog,
      (SELECT geog FROM roadnode WHERE gid= %s),
      5000
    )
    ', params.nid
  ),
  params.nid,
  5000,
  directed := true
) AS r;
```

![福山駅前通りと2号との交差点5000m圏内の計算結果をQGISで表示した図](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0075/03-drive.png)

いい眺めですね。

# おわりに


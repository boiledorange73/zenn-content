---
title: "簡略化した都道府県シングルポリゴンの生成"
emoji: "😀"
type: "tech"
topics: [PostGIS]
published: false
---
# はじめに

[トポジオメトリを使うと隙間や重なりの無いポリゴン簡略化が比較的簡単にできる](https://zenn.dev/boiledorange73/articles/3534ceab8100eb6b165c)から、もう少し足して都道府県ポリゴンを生成してみましょう。

## 前提

``topogeom_jp_3857``テーブルへのデータ格納までできているものとします。

# クエリの実行

ここでは、市区町村簡略化と、統合および面積によるフィルタリングを順次実行して、別個のテーブルに格納しています。まとめてもいいかも知れません。

## 市区町村簡略化

まず、1000メートルで簡略化を行います。この際、マルチポリゴンで出てくるので、``ST_Dump()``でシングルポリゴンにします。

```sql
CREATE TABLE smncpl1000_3857 (
  gid SERIAL PRIMARY KEY,
  mcode INT,
  geom GEOMETRY(POLYGON,3857)
);

CREATE INDEX ix_smncpl1000_3857_mcode ON smncpl1000_3857 (mcode);
CREATE INDEX ix_smncpl1000_3857_geom ON smncpl1000_3857 USING GiST(geom);

INSERT INTO smncpl1000_3857 (mcode, geom)
SELECT mcode, (ST_Dump(ST_Simplify(topo,1000))).geom FROM topogeom_jp_3857 ORDER BY gid;
```

上記のクエリは1分6秒かかりました。

## 都道府県ポリゴンの生成および面積によるフィルタリング

都道府県ポリゴンの生成と面積によるフィルタリングの、2つのフェイズに分かれます。
都道府県ポリゴンの生成は、``ST_Union``でインタセクトがある場合にできるだけつなげたものとしたうえで、``ST_Dump``で単一ポリゴンにしています。

1000メートルで簡略化すると、小さい離島は、三角形に変わり果ててしまうようなものが出てきているし、小縮尺で見るとぽつぽつとドットとして表示されていて、見づらいぐらいですので、面積でフィルタリングします。ここでは、4,000,000平方メートル以上の地物のみとします。

なお、面積によるフィルタリングでは、しきい値を大きくしすぎると、和歌山県北山村が消えてしまい、かつ奈良県ポリゴンでは対応していないので、そこに大きな湖ができることになります。注意が必要です。

```sql
CREATE TABLE spref1000_3857 (
  gid SERIAL PRIMARY KEY,
  pcode INT,
  geom GEOMETRY(POLYGON,3857)
);

CREATE INDEX ix_spref1000_3857_geom ON spref1000_3857 USING GiST (geom);
CREATE INDEX ix_spref1000_3857_pcode ON spref1000_3857 (pcode);

INSERT INTO spref1000_3857 (pcode, geom)
SELECT pcode, geom FROM (
    SELECT (mcode/1000)::INT AS pcode, (ST_Dump(ST_Multi(ST_Union(geom)))).geom
    FROM smncpl1000_3857 GROUP BY (mcode/1000)::INT
) AS Q
WHERE ST_Area(geom) >= 4000000
ORDER BY pcode;
```

上記クエリは、2.9秒かかりました。

# おわりに

簡略化した都道府県のシングルポリゴンを生成してみました。

トポロジの構築に2日ほどかかったのと比較したら、たいへん素早く処理できます。

逆に言うと、トポロジの構築には、非常に多くの計算が必要です。

一度時間をかけてトポロジを構築し、トポロジ/トポジオメトリを使いまわすようにした方がいいかなと思います。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

---
title: "Node.jsでPostGISから出たGeoJSONを受ける"
emoji: "😀"
type: "tech"
topics: [PostGIS, NodeJS, FreeBSD]
published: false
---
# はじめに

ふとNode.jsでPostGISを扱えるのかなと思いました。

# 方針

* PostGISはあくまでPostgreSQLのエクステンション。まずはPostgreSQLとNode.jsがつながらなければいけない。
* ジオメトリは放っておけばWKBHEXで出てくるけどJavaScriptで扱うならJSONにした方が良いだろう。
* ていうかPostGISからGeoJSONを出してみよう。
* js側ではGeoJSONをJSONとして受けよう。

# Node.jsとpgのインストール

Node.jsとNPMを入れ、``npm install pg``でpgを入れました。

# GeoJSONを吐くSQL

https://qiita.com/kshigeru/items/8940ecf7f261a6b01ed0 を参照。というか引き写しです。

```sql
SELECT row_to_json(featurecollection) FROM (
  SELECT
    'FeatureCollection' AS type,
    array_to_json(array_agg(feature)) AS features
  FROM (
    SELECT
      'Feature' AS type,
      ST_AsGeoJSON(geom)::json AS geometry,
      row_to_json((
        SELECT p FROM (
          SELECT [プロパティ1],[プロパティ2],...
        ) AS p
      )) AS properties
    FROM [テーブル]
    [(必要なら) LIMIT 5とか]
  ) AS feature
) AS featurecollection;
```

* [プロパティ1],... は、プロパティで出したいカラム
* [テーブル] は、テーブル名（geomとプロパティ1,...を持っている)
* [(必要なら) LIMIT 5とか] の部分は、タプルに応じて非常に長い配列になるので、テストで地物数を限定する場合に追加します。

```
    SELECT
      'Feature' AS type,
      ST_AsGeoJSON(geom)::json AS geometry,
      row_to_json((
        SELECT p FROM (
          SELECT [プロパティ1],[プロパティ2],...
        ) AS p
      )) AS properties
```

では

```
type      | geometry | properties        |
---------+------------|------------------|
Feature | ジオメトリ | {プロパティ,...}|
```

というタプルができます。ジオメトリはJSON型で、GeoJSON形式の文字列型からキャストしたものです。

そして、

```
  SELECT
    'FeatureCollection' AS type,
    array_to_json(array_agg(feature)) AS features
  FROM (
  ...
  ) AS feature
```

では、服問い合わせで出てきたタプル(複数行になりうる)を``array_agg(feature)`` で配列にまとめて（1行になる）、``array_to_json()``でJSON型にしています。

# JSONを受けるjs

https://symfoware.blog.fc2.com/blog-entry-2114.html を参照。これも引き写しです。

```javascript
const {Client} = require('pg');

const client = new Client({
  "user": (ユーザ名),
  "host": (ホスト名),
  "port": 5432,
  "database": (データベース名),
});

client.connect();

const query = `
SELECT row_to_json(featurecollection) AS json FROM (
  SELECT
    'FeatureCollection' AS type,
    array_to_json(array_agg(feature)) AS features
  FROM (
    SELECT
      'Feature' AS type,
      ST_AsGeoJSON(geom)::json AS geometry,
      row_to_json((
        SELECT p FROM (
          SELECT [プロパティ1],[プロパティ2],...
        ) AS p
      )) AS properties
    FROM [テーブル]
    LIMIT 5
  ) AS feature
) AS featurecollection
`;

client.query(query, (err, res) => {
  if( err == null ) {
    // succeeded
    let json = res.rows[0].json;
    console.log(typeof json);
    console.log(JSON.stringify(json));
  }
  else {
    // error
    console.log(err, res);
  }
  client.end();
});
```

* const query=`...`;となっているのは「テンプレート文字列」というそうです。https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/template_strings 参照。
* カラム名``json``の値は、JavaScriptのオブジェクトとして出ていて、``JSON.parse()``不要です。

なお、``console.log(res)``で``res``を見ると、次のようになりました。

```
Result {
  command: 'SELECT',
  rowCount: 1,
  oid: null,
  rows: [ { json: [Object] } ],
  fields: 
   [ Field {
       name: 'json',
       tableID: 0,
       columnID: 0,
       dataTypeID: 114,
       dataTypeSize: -1,
       dataTypeModifier: -1,
       format: 'text' } ],
  _parsers: [ [Function: bound parse] ],
  RowCtor: null,
  rowAsArray: false,
  _getTypeParser: [Function: bound ] }
```

データとしてはrowsプロパティを取り出せば、GeoJSONのデータをそのままJavaScriptで使えると思います。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

---
title: "トポロジできれいな簡略化をやってみよう"
---

# はじめに

ジオメトリではできず、トポロジならできることの例として、ポリゴンの簡略化をやってみます。

簡略化自体はジオメトリを引数に取る``ST_Simplyfy``がありますが、ポリゴンの相互関係を全く考慮していないため、簡略化を施すとそれぞれが独自に簡略化し、いたるところで隙間や交わりが発生し、所望の結果を得ることができません。

対して、トポロジで簡略化させる場合には、実際に行っているのはエッジの簡略化であり、そのエッジを共有するフェイス間の関係が保存されるので、求めた結果が得られます。

これまでの例を使って、市区町村ポリゴンの簡略化を行ってみましょう。

# 下準備

[トポジオメトリで属性を持つ](topogeom)まで実施している前提で、話を進めますので、そこまで実施して下さい。

# ジオメトリを簡略化させるとガタガタな結果になる

ジオメトリを簡略化するビューを作ってみます。

```
CREATE VIEW simplified_geom AS
  SELECT gid, n03_007::INT AS city_code, ST_Simplify(ST_Transform(geom,3857), 100) AS geom
  FROM t1;
```

``ST_Simplify``の引数は次の通りです。

1. ジオメトリ
2. 許容範囲、この値が大きいほど粗くなる


QGISで見ると次のようになります。

![ジオメトリ簡略化した結果の外観](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/simplify/01-simplifiedgeom_whole.png)

特に問題なさそうですが、拡大表示して見ます。

![ジオメトリ簡略化した結果の一部を拡大した場合](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/simplify/02-simplifiedgeom_urasaki.png)

福山市と尾道市（浦崎地区）に隙間ができています。これは望まない結果です。

今度は透明度を上げてみましょう。

![ジオメトリ簡略化した結果の一部を拡大して透明度を上げた場合](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/simplify/03-simplifiedgeom_urasaki_halftrans.png)

隙間どころか、福山市ポリゴンと尾道市ポリゴンとが重なってしまっている箇所があります。これはいけません。

# トポロジカラムを簡略化させてみよう

トポジオメトリカラムを簡略化するビューを作ります。

```
CREATE VIEW simplified AS
  SELECT gid, city_code, ST_Simplify(topo,100)
  FROM topogeom_t1_3857 ORDER BY gid;
```

``ST_Simplify``の引数は次の通りです。

1. トポジオメトリ
2. 許容範囲、この値が大きいほど粗くなる

ジオメトリを引数に与える場合と全くおなじです。

ではさきほどと同じく、QGISで見てみましょう。次のようになります。

![簡略化した結果の外観](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/simplify/04-simplified_whole.png)

特に問題なさそうですが、ジオメトリの場合は、拡大したらダメでした。本当に問題ないかを確認するため拡大して見ます。

![一部を拡大した場合](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/simplify/05-simplified_urasaki.png)

福山市と尾道市（浦崎地区）の隙間が、今度はできていません。

続いて透明度を上げてみましょう。

![一部を拡大して透明度を上げた場合](https://raw.githubusercontent.com/boiledorange73/zenn-content/main/books-images/pgis-topology-beginner/simplify/06-simplified_urasaki_halftrans.png)

隙間も交わりもできていません。きれいな市境界線になっています。

# おわりに

簡略化をジオメトリで行う場合とトポロジ（トポジオメトリ）で行う場合とで比較し、トポロジでなければ実現できないことの一例を示すことができました。

# 出典

本記事では、国土交通省国土政策局が発行する国土数値情報（行政区域）を利用しました。


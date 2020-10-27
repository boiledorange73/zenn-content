---
title: " MySQLと地理院サイトで2点間の距離が違う理由"
emoji: "😀"
type: "tech"
topics: [MySQL, GIS]
published: true
---
# はじめに

[MySQL8.0のGIS距離計算を検証してみた](https://dupont.hatenablog.jp/entry/rdbms_gis_advent_calendar20181204) で、MySQLの``ST_Distance()``で計算した距離と https://vldb.gsi.go.jp/sokuchi/surveycalc/surveycalc/bl2stf.html で計算した距離とが合わない、という報告がありました。

そして最後に、

> 現時点での結論としては
> わずかな違いはあるが、とても近い数値が出ているのでそれっぽい
> とさせてください。

とまとめられています。

これについては、私からは「『それっぽい』のが出てるんだから、これでいいと思います」と申し上げてで終了したいと思います。

# 「測地線」の「逆解法」とかいうらしい

でもちょっとの差でもやはり気になるのが一部のマニアのサガですね。もういいじゃんと思いつつも、サイバー空間をうろうろしている（ぐぐったりgithubに行ったりしている）と、いくつか分かったことがあります。

曲面上の2点間の最短線を「測地線」といいます。

測地線の計算には順解法と逆解法があります。順解法は、一つの端点と方位、距離からもう一つの端点を求めます。逆解法は、一つの端点からもう一つの端点までの距離と方位を求めます。

なので、距離（と方位）を求める計算法を探すキーワードは、"geodesic" (測地線) と "inverse" (逆) です。

# いくつかの距離計算法

回転楕円体面上の2点間の距離を計算する計算法は、いくつかあるようだと分かってきました。

Vincenty法は[WikiPediaにもあります](https://ja.wikipedia.org/wiki/Vincenty%E6%B3%95)。

Boost.GeometryのGithubを漁っていると、https://github.com/apple/turicreate/tree/master/deps/src/boost_1_65_1/boost/geometry/formulas に行きつきました。

ここから、Boost.Geometryの測地線計算では、"Vincenty"の他に"Thomas"もあるのが分かります (後述しますが他にもあります)。

## 当初Vincentyと決め打ちした

Boost.Geometryでは、複数の計算法に対応しているのですが、たぶんVincenty法を使っているだろう、と決め打ちしました。

となると、地理院さんの[距離と方位角の計算 計算式](https://vldb.gsi.go.jp/sokuchi/surveycalc/surveycalc/algorithm/bl2st/bl2st.htm)では、反復計算して、打ち切り誤差が``1.0e-15``としている、となっていたので、打ち切り誤差の差が出てるのではないか、と仮定しました。実際、Boost.Geometryの打ち切り誤差は``1.0e-12``でした。これでちょっとだけ違う結果が出る説明ができる、と思いました。

が、これは間違いでした。

# 計算法を比較する文書があった

「打ち切り誤差の違い」という（間違った）結論にたどり着いたものの、文書として出す前に、もうちょっと調べて、Boost.Geometryの開発チームが精度が落ちているのを把握しているかを確認しようと、ぐぐってみました。
その結果、ちょっと違う話を書いているPDFが引っかかりました。

行きついた先は https://archive.fosdem.org/2018/schedule/event/geo_boost/attachments/slides/2472/export/events/attachments/geo_boost/slides/2472/FOSDEM18_vissarion.pdf でした。

"Distance Computation on Boost.Geometry"というタイトルの文書です。

MySQLというより、Boost.Geometryの話です。``23.725750, 37.971536`` の点と ``4.3826169, 50.8119483`` の点との距離を、いくつかの計算方法ごとに計算した結果を提示しています。

計算結果を2つだけ引用します。

> vincenty 2088384.36606831
> andoyer  2088389.07865908

## 余談: 緯度経度か経度緯度か

``23.725750, 37.971536``と「アテネ」という情報だけ出されて、緯度経度か経度緯度か分かります？

…たぶん、分かる人、いると思う。

なお、私は分かりませんでした。

そこで、Google Mapsを開いて、検索テキストボックスに緯度経度を入れると移動してくれる、という機能を使いました。

緯度経度の順と仮定すると「23.725750N, 37.971536E」、経度緯度の順と仮定すると「23.725750E, 37.971536N」で検索します。

前者だと紅海の海上に行ってしまい、後者だと無事パルテノン神殿に到達しました。ということで、後者、経度緯度が当たりでした。

経度緯度なのでMySQLでは緯度経度に改めてクエリを投げてみます。

# 距離を計算して計算法を確認してみた

```sql
> SELECT ST_Distance(  
  ST_GeomFromText('POINT(37.971536 23.725750)', 4326),
  ST_GeomFromText('POINT(50.8119483 4.3826169)', 4326)
) AS result;

+--------------------+
| result             |
+--------------------+
| 2088389.0786590837 |
+--------------------+
1 row in set (0.00 sec)

2088389.0786590837
```

と出ています。

"Distance Computation on Boost.Geometry"の比較表のうち、vincentyとandoyerの計算結果をもう一度引用します。

> vincenty 2088384.36606831
> andoyer  2088389.07865908

どうでしょう？これは"andoyer"を使っていると推測されませんか？

で、地理院さんのサイトで計算すると ``2088384.366`` と出ました（ただし準拠回転楕円体はGRS80）。[距離と方位角の計算 計算式](https://vldb.gsi.go.jp/sokuchi/surveycalc/surveycalc/algorithm/bl2st/bl2st.htm)によると、こちらではVincentyを使っています。そして、"Distance Computation on Boost.Geometry"の比較表で見て、確かにVincentyっぽい結果だな、と思うような結果でした。

# Andoyerを本当に使ってるのか確認したら使ってた

本当に使ってるのかは最低限確認しないといけないので、Githubでmysql-serverを見てみました。https://github.com/mysql/mysql-server/blob/4f1d7cf5fcb11a3f84cff27e37100d7295e7d5ca/sql/gis/distance.cc にアクセスしてみると、確かに、Andoyerを使っているようです（98行目）。

# Andoyerはノーマーク

Andoyer、実はノーマークでした。Boost.Geometryでは、VincentyとThomasしか把握しておらず、かつ比較的新しいVincentyを使っているだろう、と決め打ちしていました。

Andoyerについては、見落としていて、そもそも存在自体を把握していませんでした。

# おわりに

若干の違いが出るのは、計算方法の差によるものでした。

Boost.Geometryは、いくつかの計算法を提供していて、MySQLはAndoyerを選択しています。

そこまで分かればすっきりできるので、誤差そのものは気にするものではありません。そもそも浮動小数点数を使ってる時点でキッチリ計算になじみません（キッチリ計算したいならDecimalとか使いますよね？）。多少の誤差があっても気にするな、です。さらには、計算法自体からして違っているので、この程度の誤差があっても気にしてはいけないと思います。

「計算方法が違う」という言葉を追加して

「計算方法が違うのだから、多少の差が出て当然なので、『それっぽい』のが出てるんだから、これでいいと思います」で終了したいと思います。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

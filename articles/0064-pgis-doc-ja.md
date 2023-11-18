---
title: "PostGIS 公式サイト上の日本語版の URL"
emoji: "😀"
type: "tech"
topics: [PostGIS]
published: true
---

# はじめに

PostGISのソースにはマニュアルのDocbook用XMLファイルとその翻訳ファイル(.poファイル)が同梱されていて、多言語でHTMLファイル等が生成できます(実際に翻訳が完成しているのは日本語とフランス語のみ)。
また、PostGISの公式サイトのマニュアルは、たぶんソースツリーで生成するのと同じ手法で生成しているものと思います。
3.3以前は、XSLTが多言語化できていなくて、翻訳できない部分がありました。3.4からは、XSLTの多言語化ができ、フルに翻訳できるようになりました。

これまでは、XSLTを無理やり日本語訳する必要があったので、https://boiledorange73.github.io/pgismanja/ などで独自に日本語訳を生成し、アップしていましたが、その必要もなくなり、今後は公式サイトでマニュアル日本語訳を読むことになると思います。

ただ、公式サイトの日本語マニュアルの URL がやや分かりにくいので、ここでご紹介します。

# 基本

英語分割版 `https://postgis.net/docs/manual-(バージョン)/`

ここをベースに "ja"フォルダ または "ja/index.html" ファイルが分割版のトップページになります。

また "postgis-ja.html" ファイルは一括版になります。

まとめると、次のようになります。

  * 分割版 `https://postgis.net/docs/manual-(バージョン)/ja/`
  * 一括版 `https://postgis.net/docs/manual-(バージョン)/postgis-ja.html`

# 最新版

  * 分割版 https://postgis.net/docs/manual-dev/ja/
  * 一括版 https://postgis.net/docs/manual-dev/postgis-ja.html

# バージョン 3.4

  * 分割版 https://postgis.net/docs/manual-3.4/ja/
  * 一括版 https://postgis.net/docs/manual-3.4/postgis-ja.html

# それ以前のバージョン

3.3以前では、分割版がありません。また、``15. PostGIS Special Functions Index`` の英語のままの部分が多くなっています。

  * 一括版 https://postgis.net/docs/manual-3.3/postgis-ja.html
  * 一括版 https://postgis.net/docs/manual-3.2/postgis-ja.html
  * 一括版 https://postgis.net/docs/manual-3.1/postgis-ja.html
  * 一括版 https://postgis.net/docs/manual-3.0/postgis-ja.html

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

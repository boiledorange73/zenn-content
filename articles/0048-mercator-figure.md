---
title: "メルカトルとは何かを計算式から見る"
emoji: "😀"
type: "tech"
topics: [GIS]
published: true
---
# はじめに

「メルカトル図法」という言葉は、非常によく聞いたものと思います。聞いていなくても、よくある長方形で描かれる世界地図は、大抵はメルカトル図法ですので、本当によく目にしています。

次の図のような地図です。

![メルカトル図法による世界地図](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0048/01-mercator.png)

見覚えあるでしょう？

これは、Natural Earthのデータを用いて、メルカトル図法（EPSG:3857）で投影した世界地図です。

今回は、メルカトル図法とは何かを数式で見てみたいと思います。

## 球面の話であって回転楕円体面の話ではない

本稿では、球面でのメルカトルについて書いています。回転楕円体面は、私には厳しいです。

# メルカトル図法は円筒図法の一種

メルカトル図法は投影法の一つで、円筒図法の一種です。

## ここで正距円筒図法を見てみましょう

メルカトル図法からずれますが、同じ円筒図法の一族で、一番楽な投影法である正距円筒図法について、ちょっと見てみます。

正距円筒図法の投影の式は次の通りとなります。

$$
\begin{matrix}
x &=& R\lambda \\
y &=& R\varphi \\
\end{matrix}
$$

ここで、``λ``, ``φ`` は経度、緯度です。また、x, yはそれぞれ緯線方向（東西）、経線方向（南北）の座標値です。Rは地球の半径です。

正距円筒図法による世界地図（Natural Earthを使用）は次の通りです。

![正距円筒図法による世界地図](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0048/02-plate-carree.png)

どうでしょう。上下に押されてへしゃげていませんか？

## メルカトル図法とは正距円筒図法のへしゃげているのをなおしたもの

このへしゃげているのを、経線方向（南北）にむりやり伸ばして、なおしたのがメルカトル図法です。

正距円筒図法の左端から右端までの横線は、それぞれの緯度における円周です。

正距円筒図法の左端から右端までの横線の距離は``R``です。対して、円周の長さは 緯度``φ`` においては``R cos(φ)``です。本来なら``R cos(φ)``でないといけないのに``R``になっている（拡大されている）ことになります。

ということは、緯線方向（東西）の拡大率は、``1/cos(φ)`` ということになります。

じゃあ経線方向（南北）はというと、上端から真ん中（赤道）までの縦線の距離は``R``で、北極から赤道までの子午線の長さは``R``ですので、拡大率は常に``1``です。

緯線方向（東西）の拡大率（``1/cos(φ)``）と経線方向（南北）の拡大率（``1``）が異なるので、へしゃげてしまっているのです（なお、赤道上だけは縦横どちらも拡大率が``1``で、へしゃげていません）。

これを縦方向にも同じ拡大率を適用すると、へしゃげているのはなおります。が、どの緯度でも一律な拡大率の適用ができないので、一度微分して、各緯度について拡大率を適用して、積分するといいです。

まず、``φ``について微分します。

$$
\frac{dy}{d\varphi} = R
$$

続いて、``φ``に異存した拡大率をかけます。

$$
\frac{dy}{d\varphi} = \frac{R}{cos{\varphi}}
$$

で、積分します。

$$
y = R\int_0^{\varphi}\frac{1}{cos{\theta}}d\theta
$$

これが、メルカトル図法の y つまり経線方向（南北）の位置です。

## 心射図法とは違う

心射図法とは、地球の中心に光源を置き、そこから地表を通って地図面に投影する、地図投影法の一派です。メルカトル図法は心射図法に入ると思われているそうです。が、違います。

本稿では「正距円筒図法のへしゃげているのを無理やりなおしたもの」と見ています。正距円筒図法は、そもそも光源を置くタイプではありません。経度緯度の極座標値に半径をかけて距離単位にしたうえで平面座標系にプロットしているだけです。この流れからだと、まあ心射図法ですとはちょっと言えないです。

また、心射円筒図法から補正する形で導出している場合もありますが、これもやはり補正している以上、心射図法とは言えません。

# 実際に積分してWikiPediaの式を確認してみよう。

$$
y = R\int_0^{\varphi}\frac{1}{cos{\theta}}d\theta
$$

から、WikiPedia ( https://ja.wikipedia.org/wiki/%E3%83%A1%E3%83%AB%E3%82%AB%E3%83%88%E3%83%AB%E5%9B%B3%E6%B3%95 ) の式と同じになるか確認してみましょう。


$$
y = R log(tan(\frac{\pi}{4}+\frac{\varphi}{2}))
$$

まず、

$$
t = tan{\frac{\theta}{2}}
$$

と定義すると、

$$
\begin{matrix}
cos{\theta} &=& cos^2{\frac{\theta}{2}} - sin^2{\frac{\theta}{2}} \\
&=& cos^2{\frac{\theta}{2}}(1-\frac{sin^2{\frac{\theta}{2}}}{cos^2{\frac{\theta}{2}}}) \\
&=& \frac{1}{1+t^2}(1-t^2) \\
&=& \frac{1-t^2}{1+t^2} \\
\end{matrix}
$$

よって、

$$
\frac{1}{cos{\theta}} = \frac{1+t^2}{1-t^2}
$$

が成り立ちます。

tを微分すると、

$$
\begin{matrix}
\frac{dt}{d\theta} &=& \frac{d}{dx}(tan{\frac{\theta}{2}}) \\
&=& \frac{d}{d\theta}(sin{\frac{\theta}{2}}\frac{1}{cos{\frac{\theta}{2}}}) \\
&=& (\frac{d}{d\theta}sin{\frac{\theta}{2}})\frac{1}{cos{\frac{\theta}{2}}} + (sin{\frac{\theta}{2}} \frac{d}{d\theta}\frac{1}{cos{\frac{\theta}{2}}}) \\
&=& \frac{1}{2} \frac{cos{\frac{\theta}{2}}}{cos{\frac{\theta}{2}}} + sin\frac{\theta}{2} (\frac{1}{2}(-\frac{1}{cos^2{\frac{\theta}{2}}}(-sin{\frac{\theta}{2}}))) \\
&=& \frac{1}{2}(1+\frac{sin^2{\frac{\theta}{2}}}{cos^2{\frac{\theta}{2}}}) \\
&=& \frac{1}{2}(1+tan^2{\frac{\theta}{2}}) \\
&=& \frac{1+t^2}{2} \\
\end{matrix}
$$

よって

$$
d\theta = \frac{2}{1+t^2}dt
$$

となります。

三つの式をもう一度、次に示します。

$$
\begin{matrix}
y &=& R\int_0^{\varphi}\frac{1}{cos{\theta}}d\theta \\
\frac{1}{cos{\theta}} &=& \frac{1+t^2}{1-t^2} \\
d\theta &=& \frac{2}{1+t^2}dt \\
\end{matrix}
$$

今度こそ y を計算します。

$$
\begin{matrix}
y &=& R\int_0^{\frac{\varphi}{2}}\frac{1}{cos{\theta}}d\theta \\
&=& R\int_0^{tan{\frac{\varphi}{2}}}\frac{1+t^2}{1-t^2}\frac{2}{1+t^2}dt \\
&=& R\int_0^{tan{\frac{\varphi}{2}}}\frac{2}{1-t^2}dt \\
&=& R\int_0^{tan{\frac{\varphi}{2}}}(\frac{1}{1-t}+\frac{1}{1+t})dt \\
&=& R[log|1+t|-log|1-t|]_0^{tan{\frac{\varphi}{2}}} \\
&=& R((log|1+tan{\frac{\varphi}{2}}|-log(1))-(log|1-tan{\frac{\varphi}{2}}|-log(1)) \\
&=& R(log|1+tan{\frac{\varphi}{2}}|-log|1-tan{\frac{\varphi}{2}}|) \\
&=& R log|\frac{1+tan{\frac{\varphi}{2}}}{1-tan{\frac{\varphi}{2}}}| \\
\end{matrix}
$$

なんとか出ました。でもこれ、WikiPediaの式とは違いますね。

ここで、

$$
\begin{matrix}
tan(\frac{\pi}{4}+x) &=& \frac{sin(\frac{\pi}{4}+x)}{cos(\frac{\pi}{4}+x)} \\
&=& \frac{sin(\frac{\pi}{4})cos(x) + cos(\frac{\pi}{4})sin(x)}{cos(\frac{\pi}{4})cos(x) - sin(\frac{\pi}{4})sin(x)} \\
&=& \frac{\frac{\sqrt{2}}{2}cos(x) + \frac{\sqrt{2}}{2}sin(x)}{\frac{\sqrt{2}}{2}cos(x) - \frac{\sqrt{2}}{2}sin(x)} \\
&=& \frac{cos(x) + sin(x)}{cos(x) - sin(x)} \\
&=& \frac{1 + tan(x)}{1 - tan(x)} \\
\end{matrix}
$$

となるので、これを先ほどのyの式に適用します。

$$
\begin{matrix}
y &=& R log|\frac{1+tan{\frac{\varphi}{2}}}{1-tan{\frac{\varphi}{2}}}| \\
&=& R log|tan(\frac{\pi}{4}+\frac{\varphi}{2})|
\end{matrix}
$$

絶対値記号があるのが気に入らないので、北半球に限定するとして、絶対値記号を取っちゃいましょう。

$$
y = R log(tan(\frac{\pi}{4}+\frac{\varphi}{2}))
$$

となりました。

ほら、ほら、WikiPedia の式と同じになってます。

# おわりに

「メルカトル図法とは正距円筒図法のへしゃげているのをなおしたもの」で、その通りに式を作り、ごにょごにょしたら、[WikiPediaのメルカトル図法](https://ja.wikipedia.org/wiki/%E3%83%A1%E3%83%AB%E3%82%AB%E3%83%88%E3%83%AB%E5%9B%B3%E6%B3%95)に書かれてる式に到達して、この考えで間違いではなかったことを確認しました。

ここからメルカトル図法の特徴が見えてくるのですが、WikiPediaにあるのでそっちを見て下さいていうか本稿全体が「WikiPediaを見て下さい」に置き換えることができそうな気がするのでこれに費やした時間は無駄だったのかもしれない。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

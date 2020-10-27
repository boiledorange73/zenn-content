---
title: "点と線分の距離を求める"
emoji: "😀"
type: "tech"
topics: [算数, JavaScript]
published: true
---
# はじめに

任意の点と任意の線分との距離を求めてみましょう。
無限遠に延長されている直線の場合には、直線と直行してかつ指定された点を通る直線との交点を求める方法（指定された点を中心として直線と接する円を求めるのも結局は同じ）で十分ですが、線分の場合には交点が線分内にあるとは限らないので、もうちょっとややこしくなります。

そこで、媒介変数を使った方法を検討してみました。

# 線分と点との距離
点$P_1=(x_1,y_1)$と点$P_2(x_2,y_2$)を端点とする線分と点$P_0=(x_0,y_0)$との距離の二乗を求めます。

線分上の点を$P_t$とすると、$P_t=(at+x_1, bt+y_1)$ となります。ただし、媒介変数 $t$は $0 \le t \le 1$ をとります。また、$a=x_2-x_1$, $b=y_2-y_1$ です。

$P_t$と$P_0$との距離の2乗は、$t$の関数となります。

$$
d2(t) = (at+x_1-x_0)^2 + (bt+y_1-y_0)^2
$$

ここで

$$
\begin{matrix}
x_1' &=& x_1 - x_0 \\
y_1' &=& y_1 - y_0
\end{matrix}
$$

とすると、$d2(t)$は次のようになります。

$$
\begin{matrix}
d2(t) &=& (at+x_1')^2 + (bt+y_1')^2 \\
&=& (a^2+b^2)t^2 + 2(ax_1'+by_1')t + (x'^2_1+y'^2_1) \\
&=& (a^2+b^2)(t^2 + 2\frac{ax_1'+by_1'}{a^2+b^2}t) + (x'^2_1+y'^2_1) \\
&=& (a^2+b^2)(t + \frac{ax_1'+by_1'}{a^2+b^2})^2 - \frac{(ax_1'+by_1')^2}{a^2+b^2} + (x'^2_1+y'^2_1) \\
&=& (a^2+b^2)(t + \frac{ax_1'+by_1'}{a^2+b^2})^2 + \frac{-(ax_1'+by_1')^2 + (x'^2_1+y'^2_1)(a^2+b^2)}{a^2+b^2} \\
&=& (a^2+b^2)(t + \frac{ax_1'+by_1'}{a^2+b^2})^2 + \frac{(ay_1'-bx_1')^2}{a^2+b^2} \\
&=& (a^2+b^2)(t + \frac{ax_1'+by_1'}{a^2+b^2})^2 + \frac{(ay_1'-bx_1')^2}{a^2+b^2}
\end{matrix}
$$

これを$x1, y1$で表すと、

$$
d2(t) = (a^2+b^2)(t + \frac{a(x_1-x_0)+b(y_1-y_0)}{a^2+b^2})^2 + \frac{(a(y_1-y_0)-b(x_1-x_0))^2}{a^2+b^2}
$$

最短点をとるときを$\overline{t}$とすると、

$$
\begin{matrix}
\overline{t} &=& -\frac{a(x1-x0)+b(y1-y0)}{a^2+b^2} \\
d2(\overline{t}) &=& \frac{(a(y1-y0)-b(x1-x0))^2}{a^2+b^2}
\end{matrix}
$$

となります。

$0 \le t \le 1$であるので、$\overline{t}$がこの範囲内に無い場合は、線分を無限遠まで延長した直線上のどこかに最短点が存在することになります。線分を考慮する際、$d2(t)$は下に凸の二次関数であるので、$\overline{t} < 0$の時は$d2(0)$が、$\overline{t} > 1$の時は$d2(1)$が、それぞれの距離となります。

$$
\begin{matrix}
d2(0) &=& (x_1-x_0)^2 + (y_1-y_0)^2 \\
d2(1) &=& (x_2-x_0)^2 + (y_2-y_0)^2
\end{matrix}
$$

# 除算を減らす

@kkdd さんからコメントを頂きました。除算を減らした方がベターですので、そのようにします。

$$
\begin{matrix}
\overline{t} = -\frac{a(x1-x0)+b(y1-y0)}{a^2+b^2} \\
0 \le \overline{t} \le 1
\end{matrix}
$$
は、$a^2+b^2 \gt 0$から、

$$
\begin{matrix}
\overline{t'} = -(a(x1-x0)+b(y1-y0)) \\
0 \le \overline{t'} \le a^2+b^2
\end{matrix}
$$
と置き換えられます。


# JavaScriptで書いてみる


```js
function min_d2(x0, y0, x1, y1, x2, y2) {
  var a = x2 - x1;
  var b = y2 - y1;
  var a2 = a * a;
  var b2 = b * b;
  var r2 = a2 + b2;
  var tt = -(a*(x1-x0)+b*(y1-y0));
  if( tt < 0 ) {
    return (x1-x0)*(x1-x0) + (y1-y0)*(y1-y0);
  }
  if( tt > r2 ) {
    return (x2-x0)*(x2-x0) + (y2-y0)*(y2-y0);
  }
  var f1 = a*(y1-y0)-b*(x1-x0);
  return (f1*f1)/r2;
}
```

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

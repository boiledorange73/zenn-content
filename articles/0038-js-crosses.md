---
title: "二つの線分がクロスするかを判定する"
emoji: "😀"
type: "tech"
topics: [算数, JavaScript]
published: true
---
# はじめに

[点と線分の距離を求める](0037-js-distance-pt-seg)にあじをしめて、今度は、二つの線分がクロスする（交わる）かどうかを求めます。

直線（無限遠）の場合には、平行でない限り、どこかで必ず交わりますが、線分の場合には、必ずしも交わりません。

## クロスする

クロスの条件は、[PostGISの空間関係をテストする関数たち](https://zenn.dev/boiledorange73/books/caea8d4c77dbba2e23a0/viewer/testing)で示しています。線分に限ると、共有部分があって、共有部分が点である場合にクロスします。

# 数式

線分$s_1 = \overline{(x_{11},y_{11}),(x_{12},y_{12})}$と$s_2 = \overline{(x_{21},y_{21}),(x_{22},y_{22})}$との交点の有無を考えます。

$s_1$, $s_2$上の任意の点を$t$で表すと

$$
\begin{matrix}
p_1(t)&=&( (t(x_{12}-x_{11})+x_{11}, t(y_{12}-y_{11})+y_{11} ) \\
p_2(t)&=&( (t(x_{22}-x_{21})+x_{21}, t(y_{22}-y_{21})+y_{21} ) \\
\end{matrix}
$$

ただし $0 \le t \le 1$ です。

$p_1(t_1)=p_2(t_2)$となる$t_1$, $t_2$を求めます。

xについて
$t_1(x_{12}-x_{11})+x_{11}=t_2(x_{22}-x_{21})+x_{21}$
$t_2 = \frac{t_1(x_{12}-x_{11})-(x_{21}-x_{11})}{x_{22}-x_{21}}$

ここで、

$$
\begin{matrix}
dx_{1} &=& x_{12}-x_{11} \\
dx_{2} &=& x_{22}-x_{21} \\
dx_{a} &=& x_{21}-x_{11}
\end{matrix}
$$

とそれぞれおくと

$$
t_2 = \frac{t_1 \cdot dx_{1}-dx_{a}}{dx_{2}}
$$

yについても同じように

$$
t_2 = \frac{t_1 \cdot dy_{1}-dy_{a}}{dy_{2}}
$$

上の二つの式から次に示す等式ができます。

$$
\frac{t_1 \cdot dx_{1}-dx_{a}}{dx_{2}} = \frac{t_1 \cdot dy_{1}-dy_{a}}{dy_{2}}
$$

これを解くと

$$
t_1 = \frac{dx_{a} \cdot dy_{2} - dy_{a} \cdot dx_{2}}{dx_{1} \cdot dy_{2} - dx_{2} \cdot dy_{1}}
$$

となります。

同じようにしてx, yのそれぞれについて$t_1$について解き、等式にすると

$$
t_2 = \frac{dx_{1} \cdot dy_{a} - dx_{a} \cdot dy_{1}}{dx_{2} \cdot dy_{1} - dx_{1} \cdot dy_{2}}
$$

$0 \le t_1 \le 1$ かつ $0 \le t_2 \le 1$ なら、線分上で交わっています。

# 分母がゼロの場合

$t_1$, $t_2$の両方で分母がゼロになる可能性があります。

$$
dx_{2} \cdot dy_{1} = dx_{1} \cdot dy_{2}
$$

となるときです。

$$
\frac{dy_{1}}{dx_{1}}=\frac{dy_{2}}{dx_{2}}
$$

と変形でき($dx_{1} \ne 0$, $dx_{2} \ne 0$のとき)、傾きが同じである場合であることが分かります。

この場合、解不能（交点がない）または解不定（いたるところに共有点がある）となります。

クロスに限定した場合は、解不能、解不定のどちらにしてもクロスしていません（解不定の場合はインタセクトすることにはなります）。

# 分母をそろえる

$$
\begin{matrix}
t_1 &=& \frac{dx_{a} \cdot dy_{2} - dy_{a} \cdot dx_{2}}{dx_{1} \cdot dy_{2} - dx_{2} \cdot dy_{1}} \\
t_2 &=& \frac{dx_{1} \cdot dy_{a} - dx_{a} \cdot dy_{1}}{dx_{2} \cdot dy_{1} - dx_{1} \cdot dy_{2}}
\end{matrix}
$$
これの分母をそろえて、割り算を減らします。$t_{2}$の右辺の分母と分子の両方について$-1$をかける、つまり正負反転します。ついでに、分母第2項の$dy_{a}$と$dx_{1}$の順序も整理します。

$$
\begin{matrix}
t_2 &=& \frac{dx_{a} \cdot dy_{1} - dy_{a} \cdot dx_{1}}{dx_{1} \cdot dy_{2} - dx_{2} \cdot dy_{1}}
\end{matrix}
$$

# コード

```javascript
function crosses_ss(s1, s2) {
  var p11 = s1[0];
  var p12 = s1[1];
  var p21 = s2[0];
  var p22 = s2[1];
  var x11 = p11[0];
  var y11 = p11[1];
  var x12 = p12[0];
  var y12 = p12[1];
  var x21 = p21[0];
  var y21 = p21[1];
  var x22 = p22[0];
  var y22 = p22[1];
  var dx1 = x12 - x11;
  var dy1 = y12 - y11;
  var dx2 = x22 - x21;
  var dy2 = y22 - y21;
  var dxa = x21 - x11;
  var dya = y21 - y11;
  var denom = dx1*dy2-dx2*dy1;

  var t1 = (dxa*dy2-dya*dx2)/denom;
  var t2 = (dxa*dy1-dya*dx1)/denom;

  return t1 >= 0 && t1 <= 1 && t2 >= 0 && t2 <= 1;
}

alert(crosses_ss([[10,10],[20,20]], [[20,10],[10,20]]));
alert(crosses_ss([[10,10],[20,20]], [[0,0],[5,0]]));
alert(crosses_ss([[10,10],[20,20]], [[110,110],[120,120]]));
```

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

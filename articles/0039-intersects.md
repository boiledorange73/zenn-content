---
title: "二つの線分がインタセクトするかを判定する"
emoji: "😀"
type: "tech"
topics: [算数, JavaScript]
published: true
---
# はじめに

[二つの線分がクロスするかを判定する](0038-js-crosses) では、二つの線分が「クロス」するかを判定しました。ここで言うクロスは、二つの線分の共有部分があり、かつ共有部分が点である場合、としています。この際、解不定となる場合は、常にクロスしないと判定しました。

今回は、もう少し条件を緩くして、インタセクトするかどうかを判定してみます。インタセクトは[PostGISの空間関係をテストする関数たち](https://zenn.dev/boiledorange73/books/caea8d4c77dbba2e23a0/viewer/testing)で示しています。ここでは、共有部分がある場合、とします。

この際、クロスしているなら必ずインタセクトしているので、そこはいいのですが、解不能と解不定との判別が必要ですし、解不定の場合には線分の区間内で共有部分があるかを判定する必要があります。

# 前回のおさらい

線分$s_1 = \overline{(x_{11},y_{11}),(x_{12},y_{12})}$と$s_2 = \overline{(x_{21},y_{21}),(x_{22},y_{22})}$とがクロスするかどうかを考えました。

$s_1$, $s_2$上の任意の点を$t$で表すと

$$
\begin{matrix}
p_1(t)&=&( (t(x_{12}-x_{11})+x_{11}, t(y_{12}-y_{11})+y_{11} ) \\
p_2(t)&=&( (t(x_{22}-x_{21})+x_{21}, t(y_{22}-y_{21})+y_{21} ) \\
\end{matrix}
$$

ただし $0 \le t \le 1$ です。

$$
\begin{matrix}
dx_{1} &=& x_{12}-x_{11} \\
dx_{2} &=& x_{22}-x_{21} \\
dx_{a} &=& x_{21}-x_{11}
\end{matrix}
$$

とおくと

$$
\begin{matrix}
p_1(t)&=&(t \cdot dx_{1} + x_{11}, t \cdot dy_{1}+y_{11} ) \\
p_2(t)&=&(t \cdot dx_{2} + x_{21}, t \cdot dy_{2}+y_{21} ) \\
\end{matrix}
$$

となります。

解不能または解不定になる条件は、

$$
dx_{2} \cdot dy_{1} = dx_{1} \cdot dy_{2}
$$

です。

これ以外はクロスする可能性があり、$t$の値の検討につながっていきます（前回の記事に書いています）。

## 解不能か解不定かの判定

解不能の場合は、共有部分が全くないので、インタセクトしません。解不定の場合は、インタセクトする可能性があります（区間については後述）。

$p_1(t_1)=p_2(t_2)$となる$t_1$, $t_2$を求める途中で、$t_2$について解いた式が出ました。

$$
\begin{matrix}
t_2 &=& \frac{t_1 \cdot dx_{1}-dx_{a}}{dx_{2}} \\
t_2 &=& \frac{t_1 \cdot dy_{1}-dy_{a}}{dy_{2}}
\end{matrix}
$$

ここから

$$
\begin{matrix}
\frac{t_1 \cdot dx_{1}-dx_{a}}{dx_{2}} &=& \frac{t_1 \cdot dy_{1}-dy_{a}}{dy_{2}} \\
t_1 \cdot dx_{1} \cdot dy_{2} - dx_{a} \cdot dy_{2} &=& t_1 \cdot dx_{2} \cdot dy_{1} - dx_{2} \cdot dy_{a}
\end{matrix}
$$

解不能または解不定になる条件は $dx_{2} \cdot dy_{1} = dx_{1} \cdot dy_{2}$ であると示しているので、これを入れ込みます。

$$
dx_{2} \cdot dy_{a} = dx_{a} \cdot dy_{2}
$$

ここで$t_1$が消えました。上の等式が満たされると解不定（インタセクトする可能性がある）、そうでないなら解不能（インタセクトしない）となります。

## 区間の判定

線分でなく無限遠に伸びる直線の場合には、さきほど導いた$dx_{2} \cdot dy_{a} = dx_{a} \cdot dy_{2}$を満たすと、いたるところで共有部分が存在します。

しかし、ここで扱っているのは線分ですので、お互いの区間内に入らなければ、共有部分は存在しません。ここからは、区間について検討します。

$s_1 = \overline{(x_{11},y_{11}),(x_{12},y_{12})}$の区間内に$(x_{21}, y_{21})$または$(x_{22}, y_{22})$が存在すれば、''ST_Overlaps(s1,s2)``（どちらか一方が存在）か``ST_Covers(s1,s2)``（両方存在）が成り立ちます。

ただし、``ST_Within(s1,s2)``については検討していません。これをカバーするには``ST_Covers(s2,s1)``のチェックをすればいいので、$s_2$の区間内に$p_{11}$または$p_{12}$があればよいことになります。

$s_1$の区間内に$(x_{21}, y_{21})$があるかどうかは、

$$
\begin{matrix}
x_{21} &=& t \cdot dx_{1} + x_{11} \\
t &=& \frac{x_{21}-x_{11}}{dx_{1}} \\
t &=& \frac{dx_{a}}{dx_{1}}
\end{matrix}
$$

で、

$$
0 \le t \le 1
$$

となればよいです。

または、yについて計算して、

$$
t=\frac{dy_{a}}{dy_{1}}
$$

を使ってもよいです。

$s_1$の区間内に$(x_{22}, y_{22})$があるかどうかは、

$$
\begin{matrix}
t &=& \frac{x_{22}-x_{11}}{dx_{1}} \\
t &=& \frac{y_{22}-y_{11}}{dy_{1}}
\end{matrix}
$$

のいずれかで、

$$
0 \le t \le 1
$$

となればよいです。


$s_2$の区間内に$(x_{11}, y_{11})$または$(x_{12}, y_{12})$があるかどうかは、

$$
\begin{matrix}
t &=& \frac{-x_{a}}{dx_{2}} \\
t &=&\frac{-y_{a}}{dx_{2}} \\
t &=&\frac{x_{12}-x_{21}}{dx_{2}} \\
t &=&\frac{y_{12}-y_{21}}{dy_{2}}
\end{matrix}
$$


のいずれかで、

$$
0 \le t \le 1
$$

となればよいです。 

ここでは、$t$に関して合計8つの式が出てきました。これらのうち一つでも条件を満たせば、インタセクトすることになります。

# 分母がゼロとなる場合の検討は不要

本来なら分母が0のときの場合分けが必要となりますが、分母が0のときは$t$は``NaN``になります。$0 \le t \le 1$は必ず満たしませんので、放置しています。

# コード

```javascript
function between01(v) {
  return v >= 0 && v <= 1;
}

function intersects_ss(s1, s2) {
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
  if( denom != 0 ) {
    // crosses ?
    var t1 = ( dxa*dy2-dya*dx2)/( dx1*dy2-dx2*dy1);
    var t2 = (-dxa*dy1+dya*dx1)/(-dx1*dy2+dx2*dy1);
    return t1 >= 0 && t1 <= 1 && t2 >= 0 && t2 <= 1;
  }
  else {
    if( dx2*dya != dxa*dy2 ) {
      // impossible
      return false;
    }
    else {
      // indefinite
      return between01(dxa/dx1) ||
        between01(dya/dy1) ||
        between01((x22-x11)/dx1) ||
        between01((y22-y11)/dy1) ||
        between01(-dxa/dx2) ||
        between01(-dya/dy2) ||
        between01((x12-x21)/dx2) ||
        between01((y12-y21)/dy2);
    }
  }
}

alert(intersects_ss([[10,10],[20,20]], [[20,10],[10,20]]));
alert(intersects_ss([[10,10],[20,20]], [[0,0],[5,0]]));
alert(intersects_ss([[10,10],[20,20]], [[110,110],[120,120]]));
alert(intersects_ss([[11,11],[12,12]], [[10,10],[20,20]]));
alert(intersects_ss([[10,10],[20,20]], [[20,20],[22,22]]));
alert(intersects_ss([[10,10],[20,20]], [[21,21],[22,22]]));
```

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

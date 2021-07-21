---
title: "Pythonで移流拡散計算を行った結果を表示する"
emoji: "🍙"
type: "tech"
topics: [Python]
published: false
---
# はじめに

[Pythonで移流拡散計算を行う](0051-py-convdiff)では、Pythonを用いた移流拡散計算を行いました。

今回は、この結果をアニメーション表示してみます。計算と表示を全て持った最終的なスクリプトも付けました。

# 描画準備

matplotlinのうちpyplotとanimationをインポートします。

``` python
from matplotlib import pyplot as plt
from matplotlib import animation as animation
```

``pyplot.figure``インスタンスを生成します。

```python
def main():
    fig = plt.figure()
```

``figure``インスタンスから``axes``インスタンスを生成します。引数``111``は、1行1列テーブルの1番目に置かれる図という意味だそうです。

```python
class SimBase:
    ...
    def __init__(self, fig, t_end, dt_out, dt, nx, dx):
        ...
        self.ax = fig.add_subplot(111)
```

``axes``インスタンスを使って、出力します。

## ``grid()``

``grid()`` は、グリッド線を引区メソッドです。

## ``set_xlim()``, ``set_ylim()``

``set_xlim()``, ``set_ylim()``は、それぞれ描画範囲を指定します。配列でない場合は、一方は0と仮定されます。

## ``plot()``

``plot()`` データをプロットします。ここで使う引数は次の通りです。

* X軸データ列
* Y軸データ列
* color: 線の描画色 ('forestgreen'など)

## ``text()``

``text()`` 文字を出力します。

* 文字位置X
* 文字位置Y
* 文字列
* ha (または horizontalalignment): 'center', 'right', 'left' のいずれか
* va (または verticalalignment): 'center', 'top', 'bottom', 'baseline', 'center_baseline' のいずれか
* transform
* fontsize: 'xx-small', 'x-small', 'small', 'medium', 'large', 'x-large', 'xx-large' のいずれか

transform = ax.transAxes を指定した場合には、axの左下隅を``(0,0)``、右上隅を``(1,1)``とする座標系となります。

## ``plot()``と``text()``の返り値をまとめる

``plot()``の返り値は、``matplotlib.lines.Line2D``型の配列です。
``text()``の返り値は、``Text``型です。
これら二つの返り値をまとめるには、``matplotlib.lines.Line2D``型や``Text``型がまざるフラットな配列にします。この配列は、描画インスタンスの配列、と見るといいのかなと思います。

なお、複数回実行した``plot()``の返り値をまとめる場合も、やはり描画インスタンスの配列の要素にします。

``ArtistAnimation``コンストラクタに与えるために、コマからなる配列で、各コマは上述の描画インスタンスの配列となっている、二次配列を返します。ここでは``ims``配列にフラットな配列を追加し、``ims``を返しようにしています。

```python
class SimBase:
    ...
    def doit(self):
        ims = []
        ...
        while t < self.t_end:
            if( t >= t_next ):
                self.ax.grid()
                self.ax.set_xlim(self.xlim)
                self.ax.set_ylim(self.ylim)
                im = self.ax.plot([float(n) * self.dx for n in range(self.nx)], self.data, color = 'forestgreen')
                title = ax.text(0.5, 1.01, 't=%.2f' % (t),
                     ha='center', va='bottom',
                     transform=ax.transAxes,
                     fontsize='large'
                    )
                ims.append(im+[title])
                t_next = t + self.dt_out # 次の出力時刻
            ...
        ...
        return ims
```

二次配列は``ArtistAnimation``のコンストラクタに与えます。

```python
def main():
    ...
    # Artistリストからアニメーションを生成
    ani = animation.ArtistAnimation(fig, ims, interval=50, blit=False, repeat_delay=0)
```

最後に、画面に表示させるなら``show()``メソッドを使用します。ファイルへの書き込みは``PillowWriter``インスタンスを用いて、``save()``メソッドで保存します。

```python
def main():
    ...
    show_screen = False
    if show_screen:
        # 画面に表示する場合
        plt.show()
    else:
        # アニメーションGIFで書き出す場合
        w = animation.PillowWriter(fps=10)
        ani.save('sim_result-exp.gif', writer=w)
```

# 計算結果

計算結果は次のようになります。

![移流拡散計算結果](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0052/01-sim_result-exp.gif)

# スクリプト

``` python
# -*- coding: utf-8 -*-
import numpy as np
from matplotlib import pyplot as plt
from matplotlib import animation as animation
from pylab import *
import time

#
# FuncAnimationで1次元計算を行うクラス
#
class SimBase:
    # コンストラクタ
    # fig: Figureオブジェクト (matplotlib)
    # t_end: 終了時刻
    # dt_out: 出力の時間刻み
    # dt: Δt
    # nx: ボックス数
    # dx: ボックスのΔx
    def __init__(self, fig, t_end, dt_out, dt, nx, dx):
        self.fig = fig
        self.t_end = t_end
        self.dt_out = dt_out
        self.dt = float(dt)
        self.nx = int(nx)
        self.dx = float(dx)
        self.dx2 = self.dx * self.dx
        self.data = np.zeros([self.nx])
        self.ax = fig.add_subplot(111)
    # 実行
    def doit(self):
        ims = []
        self.prepareData()
        self.xlim = (0, float(self.nx)*self.dx)
        self.ylim = (0, np.amax(self.data)*1.2)
        t_next = 0
        nt = 0
        t = 0.
        while t < self.t_end:
            if( t >= t_next ):
                self.ax.grid()
                self.ax.set_xlim(self.xlim)
                self.ax.set_ylim(self.ylim)
                im = self.ax.plot([float(n) * self.dx for n in range(self.nx)], self.data, color = 'forestgreen', label="t=%.1f" % (t))
                title = self.ax.text(0.5, 1.01, 't={:.2f}'.format(t),
                     ha='center', va='bottom',
                     transform=self.ax.transAxes,
                     fontsize='large'
                )
                ims.append(im+[title])
                t_next = t + self.dt_out # 次の出力時刻
            # 計算
            data_old = np.copy(self.data)
            self.calculate(data_old)
            nt = nt + 1
            t = float(nt) * self.dt
        return ims

class Sim (SimBase):
    # コンストラクタ
    # u: 流速
    # a: 拡散係数
    def __init__(self, fig, t_end, dt_out, dt, dx, u, a):
        super().__init__(fig, t_end, dt_out, dt, 100, dx)
        self.u = float(u)
        self.a = float(a)
    # データの初期化
    def prepareData(self):
        self.data = np.full(self.nx, 0.0)
        self.data[4] = 1.0
        self.data[5] = 1.0
    # 計算本体。陽解法、1次差分は中心差分。
    # data_old: 1ステップ前のデータ
    def calculate(self, data_old):
        # 境界でない部分の計算
        # rangeは 1 から self.nx-2 になる
        # (境界は0とself.nx-1)
        for ix in range(1, self.nx - 1):
            if ix >= 1 and ix < self.nx - 1:
                # ここに記述
                # ix=1..nx-2
                # self.data[ix] を data_old とパラメータ u, a を使用して更新
                dydx = 0.5 * (data_old[ix+1]-data_old[ix-1]) / self.dx
                dy2dx2 = (data_old[ix+1]-2.0*data_old[ix]+data_old[ix-1]) / self.dx2
                self.data[ix] = data_old[ix] + self.dt * (- self.u * dydx + self.a * dy2dx2)
                pass # ブロックに有効なコマンドが無いとエラーになるため記述
        # 境界条件: ここではディリクレ条件
        # ix=0
        self.data[0] = self.data[1]
        # ix = self.nx - 1
        self.data[self.nx-1] = self.data[self.nx-2]


def main():
    fig = plt.figure()
    # 計算オブジェクト生成
    sim = Sim(fig, dt_out = 1., t_end=100., dt= .1, dx = 1.0, u = 1., a = 1.)
    # 計算実行→Artistのリスト
    ims = sim.doit()
    # Artistリストからアニメーションを生成
    ani = animation.ArtistAnimation(fig, ims, interval=50, blit=False, repeat_delay=0)
    # 出力: show_screenで制御
    show_screen = False
    if show_screen:
        # 画面に表示する場合
        plt.show()
    else:
        # アニメーションGIFで書き出す場合
        w = animation.PillowWriter(fps=10)
        ani.save('sim_result-exp.gif', writer=w)

if __name__ == "__main__":
    start = time.time()
    main()
    print ("elapsed_time:{0}".format(time.time()-start) + "[sec]")
```

# おわりに

ここでは、移流拡散計算の結果をアニメーション表示するために、matplotを使用してみました。

途中で若干ふれましたが、複数のデータ系列に対して複数の``plot()``を実行した際にも、描画インスタンスの配列にするとまとめられるので、複数の計算方法を同時に表示させるといったことができます。


# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

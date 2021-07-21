---
title: "Pythonで移流拡散計算を行う"
emoji: "🍙"
type: "tech"
topics: [Python]
published: false
---
# はじめに

Pythonで移流拡散の計算を行ったのでここに記します。

とりあえず、陽解法を使い、matplotを使う予定ですが、matplotの使用は次回に回します。よって、計算結果がいっさい出てこない、計算資源のごく潰しです。

# 移流拡散方程式とその差分式とそのコード

X軸1次元で、濃度を``Φ``、拡散係数を``a``、流速を``u``とすると、移流拡散方程式は次のようになります。

$$
\frac{\partial \phi}{\partial t} = -u\frac{\partial \phi}{\partial x} + a\frac{\partial^2\phi}{\partial x^2}
$$

陽解法の差分方程式は次のようになります。

$$
\frac{\phi'_x-\phi_x}{\Delta t} = -u\frac{\partial \phi}{\partial x} + a\frac{\partial^2\phi}{\partial x^2}
$$

``data_old``から``data_new``に更新する場合、次のようになります。

``` python
                dydx = 0.5 * (data_old[ix+1]-data_old[ix-1]) / dx
                dy2dx2 = (data_old[ix+1]-2.0*data_old[ix]+data_old[ix-1]) / dx2
                data_new[ix] = data_old[ix] + dt * (- u * dydx + a * dy2dx2)
```

ここで、``u``, ``a`` は微分方程式の流速および拡散係数で、``dx``, ``dt``は、差分化した際の幅刻みと時間刻みで、``dx2``は、事前に計算した``dx*dx``です。

# 差分計算本体を除いた計算クラス

差分計算本体を派生クラスにした方がいいと思い、基底クラスを作ります。差分計算は陽解法とは限りません。

``doit()``が呼ばれると、``prepareData()`` (派生クラスに作る)を呼び出し、ループで``calculate()``(派生クラスに作る)を呼び出していきます。ちょっと特殊なのが、描画タイミングを管理している点でしょうか。

``` python
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
                #
                # TODO: 描画
                #
                t_next = t + self.dt_out # 次の出力時刻
            # 計算
            # 現在のデータを data_old に複写
            data_old = np.copy(self.data)
            # data_oldを元に現在のデータを更新
            self.calculate(data_old)
            nt = nt + 1
            t = float(nt) * self.dt
        return ims
```

# 差分計算クラス

``SimBase``から派生したクラスです。陽解法のスニペットは既に記載していますが、あれば若干変えたものでしたので、改めて記載しています。

なお、必ずセルは100個としています。

```python
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
```

# 計算開始まで

とりあえず ``main()``に全部集め、トップレベル(=名前が'__main__')スクリプトなら``main()``を呼び出すようにしています。

```python
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

とりあえずは動くと思いますが、結果を表示も出力もしていません。

いったんここで切って、次に描画について書いていきたいと思います。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

---
title: "深い深い穴を探せ"
emoji: "😀"
type: "tech"
topics: [GIS, PostGIS]
published: true
---

# はじめに

[基盤地図情報DEM配信サービス](https://boiledorange73.sakura.ne.jp/dem.html)を公開した記念の記事 [DEMのCOGを始めました](0059-dem-service) で「-134mってどこかはともかく」と申しました。

これ、しれっと言っていますが、データ変換とかやってるときにおかしくしたんじゃないかと思い、これはやばいぞと思いました。

とにかく原因究明と、自分のせいでおかしくしてたのならデータの作り直しになります。一刻も早く究明せねばなりません。

さっそく見ていきましょう。

# 最小値を出す関数を作ってみる

まずはファイルを特定しないといけません。方針としては、各行（基盤地図情報DEMのGMLファイルの1ファイルに相当）ごとに最小値を出して、-100mより下になるような行を抽出すると、それで概ね位置が特定できます。

そのうえで、ファイル内の最小値を取るセル位置も特定できるようにしておくと、さらにいいだろうと考えました。

で、作ってみたのが次の関数です。

```
CREATE TYPE rst_minval_result AS (
  x INTEGER,
  y INTEGER,
  v DOUBLE PRECISION
);


CREATE OR REPLACE FUNCTION rst_minvalue(
  rast RASTER,
  band INTEGER DEFAULT 1) RETURNS rst_minval_result AS $$
DECLARE
  x INTEGER;
  y INTEGER;
  nw INTEGER;
  nh INTEGER;
  arr DOUBLE PRECISION[][];
  ret rst_minval_result;
  v DOUBLE PRECISION;
BEGIN
  nw := ST_Width(rast);
  nh := ST_Width(rast);
  arr := ST_DumpValues(rast, band);
  FOR y IN 1..nh LOOP
    FOR x IN 1..nw LOOP
      v := arr[y][x];
      IF NOT v is NULL THEN
        IF ret IS NULL THEN
          ret := (x, y, v);
        ELSE
          IF v < ret.v THEN
            ret := (x, y, v);
          END IF;
        END IF;
      END IF;
    END LOOP;
  END LOOP;
  RETURN ret;
END;
$$ LANGUAGE plpgsql;
```

``ST_DumpValues()``でダンプしたものを二重ループで最小値を取るだけのものです。

なお、``ST_Value()``で値を取ろうとすると、とんでもなく遅くなりました。2次配列にダンプさせた方がいいです。

# 探索開始

上記の関数を dem10b の入ったテーブルで実行してみました。

```
db=# SELECT filename FROM dem10b WHERE (rst_minvalue(rast)).v < -100;
              filename              
------------------------------------
 FG-GML-6041-54-dem10b-20161001.xml
(1 □s)
```

どうもファイルはひとつだけのよう。

```
db=# SELECT rst_minvalue(rast) FROM dem10b WHERE filename = 'FG-GML-6041-54-dem10b-20161001.xml';
         rst_minvalue          
-------------------------------
 (334,432,-134.60000610351562)
(1 □s)
```

最小値を取るセルは X=334, Y=432 にあるものらしいです。

経度緯度はというと、

```
db=# SELECT ST_AsEWKT(ST_PixelAsPoint(rast, 334, 432))
  FROM dem10b
  WHERE filename = 'FG-GML-6041-54-dem10b-20161001.xml';
            st_asewkt
----------------------------------
 POINT(141.537 40.45211111130267)
(1 row)
```

東経141.537度、北緯40.45217度のようです。

# 問題の位置は分かったので見てみよう

QGISでどのへんか見てみましょう。

``Quick WKT``で``POINT(141.537 40.45211111130267)``のレイヤを一つ作ります。

![Quick WKTで入力しているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0060/01-wkt.png)

そのレイヤについて、「レイヤの領域にズーム」を実行します。

![レイヤの領域にズームを実行しようとしているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0060/02-howtojump.png)

そうすると、そのポイントに飛びます。ジャンプもしてくれるし、目印もあるしで、いいこと尽くしですね（やや言い過ぎ）。

![目標に移動したところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0060/03-jumped.png)

うん、なんか穴が開いてますね。cog-dem10bレイヤを選択したうえで、情報を見てみましょう。

![標高を見ているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0060/04-depth.png)

確かに穴でした。

ちなみに、ここはどこかというと、八戸です。

![大縮尺で見ているところ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0060/05-large.png)

## 地理院地図へのリンク

地理院地図で見てみて下さい。

https://maps.gsi.go.jp/#15/40.4521/141.537/&base=std&ls=st で見てみると、石灰石鉱山のようです。

# おわりに

いかがだったでしょうか？

本当に -134m の穴が開いてるんですね。

-134m の穴の存在は半信半疑で、おかしなことをしてたら大変だと思いましたが、正しい値と考えて良かった、と胸をなでおろしました。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

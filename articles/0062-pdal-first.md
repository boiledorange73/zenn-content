---
title: "PDALでゴニョゴニョして和歌山県某町を見てみよう"
emoji: "😀"
type: "tech"
topics: [PDAL, GIS]
published: true
---

# はじめに

https://twitter.com/shi__works/status/1696498453063627239 のツイートを見たことから始まります。

和歌山県が点群データを公開してくれています。https://wakayamaken.geocloud.jp/mp/22 参照。しかし、恥ずかしながらこれまで知らなかったです。

で、ツイートの動画を見ると、和歌山駅近辺を見せてくれていました。ステーションデパートもはっきり分かるし、貴志川線の架線柱さえもしっかり見える。和歌山駅ユーザだった者にとっては楽しいです…が、そこは本来はどうでもよい話。

和歌山県さんがアップなさっている点群データはテキストファイルですが、[PDAL](https://pdal.io/)を使って[COPC](https://copc.io/)に変換してどこかにアップすると、ウェブアプリベースでとりあえずビジュアライズできるということです。

これはやってみるしかないですね。

# 参考にしたもの

まず https://github.com/shi-works/copc-format-wakayama-point-cloud-data です。ここが全てと言っていいぐらい。

あと、pdalの公式サイト https://pdal.io/ もおさえておくべきです。

それと、ビジュアライズしてくれるツールは https://copc.io/ が出して下さっています。

# 順を追って何をしたかを記録しておく

## とりあえずコマンドを実行していくもCOPC化できず

まず、https://github.com/shi-works/copc-format-wakayama-point-cloud-data でなんとなく変換していく。

しかし「ちょっと」問題があって、最後の COPC への変換に失敗しました。

ということで、LASを某サーバにアップしてビジュアライズしてみると、表示されない。

あー「めちゃくちゃ」問題やないか。

なお、入れたコマンドと出たメッセージは

```
% pdal translate -i 06QC293_org.las -o 06QC293.laz --writers.copc.forward=all
PDAL: Argument references invalid/unused stage: 'writers.copc'.
```

でした。

## とりあえずパイプラインを楽しむ

https://github.com/shi-works/copc-format-wakayama-point-cloud-data の、パイプラインのサンプル見ながら、ファイル名をとりあえずいじりました。

```
{
  "pipeline":[
    {
      "type":"readers.text",
      "filename":"06QC293_org.txt",
      "separator":",",
      "header":"ID,X,Y,Z,B",
      "spatialreference":"EPSG:6674"
    },
    {
      "type":"filters.reprojection",
      "out_srs":"EPSG:6674"
    },
    {
      "type":"writers.las",
      "filename":"06QC293_org.las"
    }
  ]
}
```

ここでふと思ったのが、「パイプライン」を自称している以上、処理を渡していってるんだろうと。ここにシレっとCOPCを入れたら、COPCが出るんでないかと。

```
{
  "pipeline":[
    {
      "type":"readers.text",
      "filename":"06QC293_org.txt",
      "separator":",",
      "header":"ID,X,Y,Z,B",
      "spatialreference":"EPSG:6674"
    },
    {
      "type":"filters.reprojection",
      "out_srs":"EPSG:6674"
    },
    {
      "type":"writers.las",
      "filename":"06QC293_org.las"
    },
    {
      "type":"writers.copc",
      "filename":"06QC293_org.laz"
    }
  ]
}
```

きた！きた！ LAS と LAZ が同時に出た！

## 振り返ってパイプラインを使わない方法を考える

``pdal translate``で、どうにかCOPCに変換できないかと考えました。

既に言及してますが、もう一度書きます。次のエラーが出ました。

```
% pdal translate -i 06QC293_org.las -o 06QC293.laz --writers.copc.forward=all
PDAL: Argument references invalid/unused stage: 'writers.copc'.
```

### オプションがいらないのでは?→そんなわけなかった

オプションを外すと変換できるので、場合によってはファイル名のサフィックスを見てタイプを判断して変換してくれる可能性があるので、やってみました。

```
% pdal translate -i 06QC293_org.las -o 06QC293.laz
% pdal info --metadata 06QC293.laz
{
...
  "copc": false,
...
}
```

``pdal translate``自体は成功しましたが、COPCにはなりませんでした。

### 書き出しフォーマットを指定するオプション発見

それでも、なんとなく https://pdal.io/en/latest/apps/translate.html をみてて、``--writer``オプションがあるのを発見。おそらく ``--writer copc`` でCOPC出力が可能になるのだろうと。

### 読み込み/書き出しオプションもなんとなく分かる

あと、失敗していたコマンドの中にあったオプション ``--writers.copc.forward=all`` に手を付けていませんでした。でも、これはおそらく COPC に出力する際のオプションであろうと思われます。

というのがパイプラインファイルを眺めると、``"type":"readers.text"`` や ``"type":"writers.copc"`` とかが出てきます。つまり ``reader.(タイプ)``、``writer.(タイプ)`` という書式があるということです。

また、reader / writer にかかるオプションは、``pdal --options readers.text`` や ``pdal --options writers.copc`` などで確認できます。

ここで、``--writers.copc.forward=all`` を見返して考えると、``--reader.(タイプ).(プロパティ名)=(プロパティ値)``、``--writer.(タイプ).(プロパティ名)=(プロパティ値)`` の書式が有効であろうと思われます。

読み取り書き出しフォーマットの指定、オプション指定等を、パイプラインファイルを参照しつつ検討したところ、やや冗長ですが、次のような引数になるだろうと考えました。

```
pdal translate \
  --reader text \
  --readers.text.separator="," \
  --readers.text.header="ID,X,Y,Z,B" \
  --readers.text.spatialreference="EPSG:6674" \
  --writer copc \
  --writers.copc.forward=all \
  06QC293_org.txt \
  06QC293.laz
```

これは成功しました。次に、``pdal info``で COPC かどうかの確認をしました。

```
pdal info --metadata 06QC293.laz | less
```

```
  "metadata":
  {
    ...
    "copc": true,
  ...
```

これこれ、COPCになってます。こういうデータが出るのを待ってました。

# ということでアップ

## 利用規約について

https://wakayamaken.geocloud.jp/mp/22 に最初にアクセスすると、次の文面が現れます（抜粋、丸囲み数字改変）。

```
【利用規約】
「和歌山県地理情報システム」利用規約の他、
「3次元点群データ」の使用にあたっては、
以下の利用条件のすべてに同意していただきます。

　本データは、和歌山県が航空レーザー測量により取得したデータに
　よるものです。
　　測量時期：平成25年度(北山村付近)
　　　　　　　令和元年度

　着色メッシュについては、県によるデータ取得範囲外となりますので、
　公開の対象外となります。
...
　本データの内訳は、次のとおりです。
　　(1) オリジナルデータ
　・建物や樹木などの地物の高さを含む地球表面データ
　　　...
　　(2) グラウンドデータ
　　　・オリジナルデータのうち建物や樹木などの地物の高さを
　　　取り除いたもの
...
　・上記(1) (2)については、データをパソコンにダウンロードして、
　　申請なしにご利用いただけますが、出典の記載をお願いします。
```

ということで、``*._org.txt``、``*._grd.txt`` は、出典明記で大丈夫そうです。

## アップロードと設定

普通にアップロードはいいのですが、今回はCORS対策が必要なようですので、``.htaccess``に次を書き込みました。

```
Header set Access-Control-Allow-Origin: "*"
```

そのうえで、lazファイルのURLを指定して、見てみます。

https://viewer.copc.io/?copc=https://boiledorange73.sakura.ne.jp/06QC293.laz

## 空から某所を見てみよう

![06QC293.laz全体](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0062/01-overview.png)

06QC293 の全体です。

紀ノ川の水面は、レーザーの性質上、欠測になってしまいますが、中州っぽいのは見えてますね。

画面の左下のあたりには、川に削られた様子が見えます。小学校が河岸段丘のヘリに立っていたので、南側の見晴らしは意外とよく、国鉄和歌山線の列車をよく見てました。特に11時40分頃に通過する急行が記憶にあります。

妙寺小学校も松源妙寺店も見えてます。京奈和も見えてて、農業大学校もちょっとだけ見えてます。

紀ノ川の南側に目を向けると、三角形の平野部は九度山なのでよくわかりませんが、さらに南にある山は「テレビ塔」と呼んでいました（山なのに）。テレビ塔のふもとと、平野部とに、やや高い構造物がありますが、送電鉄塔です。


![妙寺小学校](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0062/02-elementary.png)

妙寺小学校が見えます。私の通っていたころの建物はほぼ撤去されてしまいましたが、南端の「新館」だけ残ってます。まと、校舎の東南東にある四角い「抜け」はプールです。

![中飯降駅](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0062/03-stop.png)

これは本当にどうでもいいけど(上の2枚もたいがいですかそうですか)、中飯降駅。赤線で囲ったあたりにホームがあります。切通しにホームが置かれた無人駅で、周りから見えません。近くの者としたらまったく気にしていませんでしたが、「初見殺し」です。

# おわりに

いかがだったでしょうか？

自分の知ってるところを見ると、本当に気分が上がりますね。これまで点群データをいじらなかったに急にやりだしたのは、データが身近だったからです。

あとビジュアライゼーションができにくかったので、何度も出しますけど https://viewer.copc.io/ のアプリと、 https://github.com/shi-works/copc-format-wakayama-point-cloud-data の記事がありがたかったです。

# 出展

本稿の画像作成にあたっては「3次元点群データ」(和歌山県) (https://wakayamaken.geocloud.jp/mp/22) を使用しました。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

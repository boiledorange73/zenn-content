---
title: "多面体サーフェス"
emoji: "😀"
type: "tech"
topics: [PostGIS]
published: true
---
# はじめに

多面体サーフェス(POLYHEDRALSURFACE)については、全く調べてなかったので、マルチポリゴンと何が違うのかも分かっていませんでした。
"OpenGIS Implementation Specification for Geographic information - Simple feature access - Part 1: Common architecture" 06-103r4 に定義が出ている(https://www.ogc.org/standards/sfa からダウンロードできます)ので、若干確認しました。

あくまで若干の確認程度ですよ。自分でもいまいち理解できていません。

## 確認方法

``ST_IsValid()``でできるかなと思ったのですが、判定してくれませんでした。代わりに、SFCGALによる関数に渡すと妥当性の確認をしてくれました。ここでは``ST_3DArea()``を使っています。

# 妥当となる条件

多面体サーフェスは、ポリゴンを要素に持ちます。この点においては、マルチポリゴンと同じですが、さらに二つの条件が付きます。

## 他の要素と接続されなければならない

ある要素は、他の要素と接触していて、共通部分が線でなければなりません。そうでなければ、接続されていない(not connected)エラーになります。

```sql
SELECT ST_3DArea('POLYHEDRALSURFACE(((0 0, 10 0, 10 10, 0 0)),((0 -10, 10 -10, 10 -20, 0 -10)))'::geometry);
```

```
ERROR:  When converting to 3D - PolyhedralSurface is invalid : not connected : POLYHEDRALSURFACE(((0/1 0/1 0/1,10/1 0/1 0/1,10/1 10/1 0/1,0/1 0/1 0/1)),((0/1 -10/1 0/1,10/1 -20/1 0/1,10/1 -10/1 0/1,0/1 -10/1 0/1)))
```

共通部分が点になっている場合もダメです。

```sql
SELECT ST_3DArea('POLYHEDRALSURFACE(((0 0, 10 0, 10 10, 0 0)),((0 0, -10 0, -10 -10, 0 0)))'::geometry);
```

```
ERROR:  When converting to 3D - PolyhedralSurface is invalid : not connected : POLYHEDRALSURFACE(((0/1 0/1 0/1,10/1 0/1 0/1,10/1 10/1 0/1,0/1 0/1 0/1)),((0/1 0/1 0/1,-10/1 0/1 0/1,-10/1 -10/1 0/1,0/1 0/1 0/1)))
```

## 回る方向を揃えなければならない

要素が接続されていても、要素が右回りか左回りかに統一されていないといけません。接続する線においては、二つのポリゴンは互いに逆方向に向きます。

![隣接ポリゴンと巡回方向](https://storage.googleapis.com/zenn-user-upload/qhwpj6ch2jky4ztgfd2rhe5m9k8e)

```sql
SELECT ST_3DArea('POLYHEDRALSURFACE(((0 0, 10 0, 10 10, 0 0)),((0 0, 10 0, 10 -10, 0 0)))'::geometry);
```

```
ERROR:  When converting to 3D - PolyhedralSurface is invalid : inconsistent orientation of PolyhedralSurface detected at edge 0 (0-1) of polygon 1 : POLYHEDRALSURFACE(((0/1 0/1 0/1,10/1 0/1 0/1,10/1 10/1 0/1,0/1 0/1 0/1)),((0/1 0/1 0/1,10/1 0/1 0/1,-10/1 10/1 0/1,0/1 0/1 0/1)))
```

上記の例では、接続される線は``0 0, 10 0``の部分です。一つ目の要素も二つ目の要素も``0 0, 10 0``で同じ方向に向く線になっているので、エラーとされました。

## エラーメッセージは接続の方が優先

接続していないことと、回る方向が揃っていないことが同時に発生した場合に、エラーメッセージで優先されるのは接続の方です。

```sql
SELECT ST_3DArea('POLYHEDRALSURFACE(((0 0, 10 0, 10 10, 0 0)),((0 -10, 10 -20, 10 -10, 0 -10)))'::geometry);
```

```
ERROR:  When converting to 3D - PolyhedralSurface is invalid : not connected : POLYHEDRALSURFACE(((0/1 0/1 0/1,10/1 0/1 0/1,10/1 10/1 0/1,0/1 0/1 0/1)),((0/1 -10/1 0/1,10/1 -20/1 0/1,10/1 -10/1 0/1,0/1 -10/1 0/1)))
```

## 不正扱いされなかったのはこれだ

```sql
SELECT ST_3DArea('POLYHEDRALSURFACE(((0 0, 10 0, 10 10, 0 0)),((0 0, 10 -10, 10 0, 0 0)))'::geometry);
```

## XYでOKなの!?

"OGC 06-103r4" によると、XYZでなければならないなんて書いていませんでした。書いてあるのは、接続されていることと、回る方向を揃えることだけです。

"OGC 06-103r4"では「多面体サーフェスは閉じているなら立体の境界である」、という記述もあったりする（6.10.1）ので、通常はXYZ, XYZMで使用するものだろうと思います。

# おわりに

多面体サーフェスについて、若干踏み込んでみました。

それでもいまひとつ分かっていません。``ST_IsValid()``が多面体サーフェスの条件を確認していないことさえ知らなかったぐらいですから。

その程度のものとして取り扱って頂ければありがたく思います。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

---
title: "メソッドを作る"
emoji: "😀"
type: "tech"
topics: [JavaScript]
published: false
---
# はじめに

[newで独自オブジェクトを作る](0024-js-creatobj)で、オブジェクトを作る方法を示しましたが、ここにメソッドを追加します。

# メソッドは prototype に入れる

## ソース

``isYakusodhi()``というメソッドを作ってみました。

最初に、全体を示します。

```js
/*
 * Person 人のデータ
 * @param name 氏名
 * @param age 年齢
 */
function Person(name, age) {
  this.name = name;
  this.age = age;
}

Person.prototype.isYakudoshi = function() {
  if( this.age == 25 || this.age == 42 || this.age == 61 ) {
    return "厄年です";
  }
  else {
    return "違います";
  }
}

Person.prototype.CLASSNAME = 'Person';

var p1 = new Person("山田太郎", 30);
var p2 = new Person("中村太郎", 42);

console.log(p1.name+" は厄年?: "+p1.isYakudoshi()); // 山田さんは厄年でない
console.log(p2.name+" は厄年?: "+p2.isYakudoshi()); // 中村さんは厄年

console.log(p1.CLASSNAME);
```

次にメソッドだけを切り出してみます。

```js
Person.prototype.isYakudoshi = function() {
  return this.age == 25 || this.age == 42 || this.age == 61;
}
```

``new Person()``で作られたオブジェクトは２つ（別に３つ以上であっても変わりません）であるのに対して、メソッドは１つ作っているだけです。

``Person.prototype``の下に置いておいたものは、``new Person()``で作られたオブジェクトの全てからアクセス可能です。

これによっていくつものオブジェクト同士で共有できています。同じ関数を呼び出し使うので、``new Person``で生成するたびに関数を作る必要が無くなります。

ひとたびメソッドを作れば使いまわせるところが、メソッドの良いところです。

## メソッドにアクセスする手順

もう少しアクセスする手順を掘ってみます。処理系がメソッド``p1.isYakudoshi``にアクセスする手順は、次の通りです。

* ``p1``に``isYakudoshi``が存在しているならそれを実行
* なければ``p1.__proto__``に``isYakudoshi``が存在しているならそれを実行
* それでもなければ``p1.__proto__.__proto__``を見て存在しているならそれを実行（以下繰り返し）

ここで``__proto__``というキーが出てきています。[newで独自オブジェクトを作る](0024-js-creatobj)で少しだけ言及していますが、「「生成したオブジェクト」の``__proto__``に関数の``prototype``を代入」しています。オブジェクトには必ず存在します。

具体例を見てみましょう。``p1``は次のようになっています。

```js
{
  "name": "山田太郎",
  "age": 30,
  "__proto__": { // Person.prototypeと同じ
    "isYakudoshi": function() {...}, // Person.prototype.isYakudoshiと同じ
    "__proto__": {
      ...
    }
  }
}
```

``p1.isYakudoshi``は存在しません(p1にあるのは``name``, ``age``, ``__proto__``)。
次に``p1.__proto__``を見ます。ここに``isYakudoshi``があるので、これを実行します。

## thisが参照するもの

次にメソッド内部を見てみると、``this``というワードが出ています。

[newで独自オブジェクトを作る](0024-js-creatobj)でコンストラクタを呼び出す際に``this``が「生成されたオブジェクト」を参照するように設定されている、と言いました。

今回の場合も``this``は、オブジェクトを参照します。

``p1.isYakudoshi``では、``this``と``p1``は同じオブジェクトを参照しています。ですから、メソッド内で``this``のメンバの値を変えると、``p1``のメンバの値が変わります。

## prototypeに入れられるのはメソッド（関数）だけではない

```js
Person.prototype.CLASSNAME = 'Person';
```

という式文があります。これによって、``p1``でも``p2``でも同じ値"Person"になります。

# おわりに

``(関数).prototype``によって ``new``で生成したオブジェクト間でメンバが共有されることと、メンバを関数にすると、共有した関数を呼び出せるので、メソッドのようになることを紹介しました。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

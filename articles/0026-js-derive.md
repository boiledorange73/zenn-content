---
title: "派生クラスを作る"
emoji: "😀"
type: "tech"
topics: [JavaScript]
published: true
---
# はじめに

オブジェクト指向では、新しいクラスを定義する際に、既存のクラスの定義に追加するようなかたちで定義することがあります。

そうすると、既に書いてあることを、そのまま別のところに書く、といったことをする必要が無くなります。

コーディングの労力だけで見ると、コピペで済むので別にありがたみも湧きませんが、バグが発見され、一方で訂正して、もう一方で訂正せずにいると、デバッグが終わったのに同じような症状が出て、いったいどうしたものか分からなくなる、といったややこしい状況を作ってしまいかねません。

## 派生クラスと基底クラス

「新しいクラスを定義する際に、既存のクラスの定義に追加するようなかたちで定義する」場合の、「新しいクラス」を「派生クラス」、「既存のクラス」を「基底クラス」と呼びます。

# ソースコード

今回のPersonでは氏名に敬称を付けるメソッドを追加してみました。また、CustomerというクラスをPersonから派生させました。

```js

// -------- Person

/*
 * Person 人のデータ
 * @param name 氏名
 * @param age 年齢
 */
function Person(name, age) {
  this.name = name;
  this.age = age;
}

/**
 * Person#getTitle() - その人の敬称を得る
 * @return 常に"さん"を返す
 */
Person.prototype.getTitle = function() {
  return "さん";
};

/**
 * Person#getNameWithTitle() - その人の名前を敬称付きで得る
 * @return その人の名前に敬称を付けた文字列
 */
Person.prototype.getNameWithTitle = function() {
  return this.name + this.getTitle();
};

// -------- Customer

/**
 * Customer extends Person - 顧客
 * @param name 名前
 * @param age 年齢
 */
function Customer(name, age, id) {
  this.name = name;
  this.age = age;
}

/* prototypeチェーンを作る */
Customer.prototype = new Person();

/**
 * Person#getTitle() - その人の敬称を得る
 * @return 常に"様"を返す
 */
Customer.prototype.getTitle = function() {
  return "様";
};

// Person 山田太郎(30歳)を生成
var p1 = new Person("山田太郎", 30);
// 名前を敬称を付けてコンソールに出力
console.log(p1.getNameWithTitle());

// Customer 鈴木次郎(40歳)を生成
var c1 = new Customer("鈴木次郎", 40,"A12345678");
// 名前を敬称を付けてコンソールに出力
console.log(c1.getNameWithTitle());
```

これはPersonオブジェクトの山田さんについて「山田太郎さん」が出力されます。また、Custermオブジェクトの鈴木さんについて「鈴木次郎様」が出力されます。

# メソッドの呼び出しのおさらい

``p1.getNameWithTitle()``の挙動を見ていきましょう。``p1``は次のようになっています。

```js
{
  "name": "山田太郎",
  "age": 30,
  "__proto__": {  // Person.Prototypeと同じ
    "getTitle": function() { return "さん"; },
    "getNameWithTitle": function() {...},
    "__proto__": {...}
  }
}
```

``p1._proto__.getNameWithTitle``が発見され、これが呼び出されます。このメソッドの中身は次のようになっています。

```js
return this.name + this.getTitle();
```

ここで前回述べたとおり``this``と``p1``は同じです。ので、置き換えてみます。

```js
return p1.name + p1.getTitle()
```

同じように、``p1.getTitle``も探してみると、

```js
return "さん";
```

となっているので、「さん」という文字列が返されます。

これで、``getNameWithTitle``は次のように置き換えることができます。

```js
return p1.name + "さん";
```

これで「山田太郎さん」と出力されることが分かりました。

# 派生クラスの場合

次はCustomerクラスについて見てみましょう。

Customerでは、不思議に思われるかもしれない式が出てきます。

```js
Customer.prototype = new Person();
```

これによって、``Customer.prototype``は次のようになります。

```js
{
  "name": null,
  "age": null,
  "__proto__": { // Person.prototypeと同じ
    "getTitle": function() { return "さん"; },
    "getNameWithTitle": function() {...},
    "__proto__": {...}
  }
}
```

## メソッドの継承

これを念頭に、``c1``についてみてみましょう。

```js
{
  "name": "鈴木次郎",
  "age": 40,
  "__proto__": { // Customer.prototypeと同じ
    "name": null, // 無視
    "age": null, // 無視
    "getTitle": function() { return "様"; },
    "__proto__": {  // Person.prototypeと同じ
      "getTitle": function() { return "さん"; },
      "getNameWithTitle": function() {...},
      "__proto__": {...}
    }
  }
}
```

``c1.getNameWithTitle``を実行しようとすると、どうでしょうか。``c1.getNameWithTitle``はもちろん、``c1.__proto__.getNameWithTitle``もありません。そこでさらに``c1.__proto__.__proto__.getNameWithTitle``が発見され、これが実行されます。

これは「Person.prototypeと同じ」と書いているブロック内にあります。ちょっと戻って``p1``を見ると「Person.prototypeと同じ」と書いているブロック内にあります。

つまり、``c1.getNameWithTitle``と``p1.getNameWithTitle"は同じです。

基底クラスのメソッドに派生クラスからアクセスでき、言い方を変えれば、派生してもメソッドが受け継がれていることが分かります。

## 派生クラスでのメソッドの上書き

``c1.getTitle``については``c1.__proto__.getTitle```で発見されて、「様」を返します。

よって、「Person.prototypeと同じ」と書いているブロック内にある``getTitle``（「さん」を返す）は呼ばれません。

Customer（派生クラス）ではメソッドが上書きされ、Person（基底クラス）では以前のまま使えます。

派生クラスと基底クラスで違いを出したい場合には、このようにしていきます。

## 基底クラスのメソッドでのthis

``c1.getNameWithTitle``で呼ばれるのはPersonクラスのメソッドです。もしかして
``this``が``p1``と同じという状況になるのではないか、と不安を感じるかも知れません。

安心して下さい、``this``と``c1``が同じになります。

# おわりに

派生クラスを作る際に、派生クラスの``prorotype``に``new 基底クラス()``としたものを放り込むという「おまじない」をすることで、メソッドが継承でき、また、上書きもできることを示しました。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

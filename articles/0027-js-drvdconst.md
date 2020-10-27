---
title: "派生クラスのコンストラクタ"
emoji: "😀"
type: "tech"
topics: [JavaScript]
published: true
---
# はじめに
 
派生クラスを作ったけど、派生クラスのコンストラクタと基底クラスのコンストラクタは相互に独立したものになっています。派生クラスのコンストラクタも基底クラスのコンストラクタの一部を使いまわしたいところです。

派生クラスから基底クラスのコンストラクタを呼び出してみます。

## コンストラクタとは

これです。

```js
function Person(name, age) {
  this.name = name;
  this.age = age;
}
```

# コード

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
 * @param id 顧客ID
 */
function Customer(name, age, id) {
  Person.call(this, name, age); // Personのコンストラクタを呼ぶ
  this.id = id;
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

# そういうものだと思ってくれ

派生クラスのコンストラクタは、次のようになっています。

```js
function Customer(name, age, id) {
  Person.call(this, name, age); // Personのコンストラクタを呼ぶ
  this.id = id;
}
```

これで、Personのコンストラクタを呼び出し、かつ``function Person()``内の``this``は、``function Customer()``内の``this``と同じになります。

# 派生クラスを作るときのまとめ

[派生クラスを作る](0026-js-derive)の記述とあわせて、派生クラスを作るときにどうするかを次に示します。

* 派生クラスのコンストラクタで基底クラスのコンストラクタを``<基底クラス>.call()``で呼ぶ
* 派生クラスのコンストラクタの直後で''<派生クラス>.prototype = new <基底クラス>()''を実行する

# おわりに

ざーっと駆け足で、クラスと派生クラスについての話を書きました。

``function``の話とか、もうちょっと書きたいところですが、また今度、気が向いたらやります。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

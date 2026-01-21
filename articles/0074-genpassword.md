---
title: "パスワード自動生成プログラム"
emoji: "😀"
type: "tech"
topics: [JavaScript]
published: true
---

# はじめに

パスフレーズの手動生成が必要そうになったので急遽作成したものです。

# スクリプト

'''html
<html>
<body>
<script>
/**
 * パスワードを生成します。
 * 整数配列 types で指定した整数値に対応する英小文字、英大文字、数字、記号が必ず1つ以上入ります。
 * types の要素で有効な整数値でないものは無視されます。
 * types が指定されていないか、有効な整数値が全く無い場合には、すべての文字タイプが使われます。
 * パスワード長が 1 以上でない場合には空文字列が返されます。
 * パスワード長が types の要素数未満の場合、英子文字、英大文字、数字、記号の順で優先して使われます。
 * @param {number} len - パスワード長
 * @param {number[]} [types] - 使用する文字タイプ
 * @returns {string} 生成されたパスワード
 */
function genPassword(len, types) {
  if( !(len > 0) ) {
    return "";
  }
  var chars_arr = [
    "abcdefghijklmnopqrstuvwxyz",
    "ABCDEFGHIJKLMNOPQRSTUVWXYZ",
    "0123456789",
    "!@#$%^&*()-_=+[]{};:,.<>?"
  ];
  var chars_arr_len = chars_arr.length;
  // typesをきれいにする | null にする
  if( Array.isArray(types) ) {
    types = types.filter(
      (v, ix, arr) => (arr.indexOf(v) === ix) && (v >= 0 && v < chars_arr_len)
    );
  }
  else {
    types = null;
  }
  // 正しいtypes値が一つも入っていない場合にはデフォルトにする
  if( types == null ) {
    types = [0,1,2,3];
  }
  // 前半: 各タイプに少なくとも1文字を入れる
  var aret = []; // パスフレーズ
  var cands = ""; // 後半で使用する候補
  for(var t = 0; len > 0 && t < types.length; t++, len-- ) {
    var chars = chars_arr[types[t]];
    cands = cands + chars;
    var ix = Math.floor(Math.random() * chars.length);
    aret.push(chars.charAt(ix));
  }
  // 後半: 全タイプを混ぜた候補から指定
  while( len > 0 ) {
    var ix = Math.floor(Math.random() * cands.length);
    aret.push(cands.charAt(ix));
    len--;
  }
  // ランダムに並べ替え
  for(var n = 0; n < aret.length-1; n++ ) {
    var ix = n + Math.floor(Math.random() * (aret.length-n));
    // swaps
    var c = aret[n]
    aret[n] = aret[ix];
    aret[ix] = c;
  }
  return aret.join("");
}

document.write(genPassword(10,[99,99,99]));
</script>
</body>
</html>
```

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

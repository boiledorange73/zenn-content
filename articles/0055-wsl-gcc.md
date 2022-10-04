---
title: "WSL上でCコンパイラを動かすまで"
emoji: "🖥"
type: "tech"
topics: [C, WSL, Ubuntu]
published: true
---

# はじめに

WindowsではWSLを使ってUbuntuが動きますので、Linux系プログラム作成に使ってみます。

# WSLを有効化しUbuntuを入れる

## WSlの有効化

WSLの有効化は「Windowsの機能の有効化または無効化」から行います。

「スタート」→「Windows システムツール」→「コントロールパネル」を順に選択して、コントロールパネルを開きます。

「コントロールパネル」ではアイコンが並んでいますが「プログラムと機能」をクリックします。「プログラム」とだけ表示されている場合は、そこをクリックすると「プログラムと機能」のアイコンが現れます。

「プログラムと機能」では、インストールされているプログラムの一覧が現れますが、それは無視して、左端の「Windowsの機能の有効化または無効化」をクリックします。

「Windowsの機能の有効化または無効化」では、小さめウィンドウで、チェックボックスがついた機能の一覧が表示されています。この中の「Linux用Windowsサブシステム」にチェックを入れ、「OK」をクリックします。

## Ubuntuを入れる

「Microsoft Store」から「Ubuntu」をダウンロード、インストール、実行します。
そうすると、ストアアプリは Ubuntu のインストーラだったことが分かります。
ご自身のユーザ名、パスワード等を入力します。
「Mount Location」「Mount option」は、デフォルトのままでいいです。
「The installer running on is currently installing the system.」とか表示されるので、そのまま置いておきます。

そのうちインストールが終了して、Ubuntuをリブートします。Windowsのリブートではありません。

## apt updateに失敗する

```
sudo apt update
```

などを実行するときに

``
Temporary failure resolving 'archive.ubuntu.com'
``

が出ました。これは、DNS名前解決ができていないためなのだそうです。

https://qiita.com/ryosukeYamazaki/items/c04ec3ff78aac6eb8d26

(2022年10月4日追加) まず、/etc/resolv.conf が /run/resolvconf/resolv.conf へのシンボリックリンクなので、これをカットします。

```
sudo rm /etc/resolv.conf
```

そのうえで /etc/resolv.conf に置き換えます。、**(DNSアドレス)は適切なアドレスに置き換えて下さい**。

```
sudo sh -c "echo 'nameserver (DNSアドレス)' > /etc/resolv.conf"
```

今後 resolv.conf を自動で書き換えないようにするために、次のようにします。

```
sudo sh -c "echo '[network]\ngenerateResolvConf = false' > /etc/wsl.conf"
```

なお、'\n'が付いているのに "echo -e" としていないのですが改行できてしまっています。これは、どうも引数を "" で括っているために改行コードに置換されているみたいです。


## gcc, gdb を入れる

```
sudo apt install gcc
sudo apt install gdb
```

# コードを書いてみる

## Windowsからファイルシステムにアクセス

「エクスプローラ」のアドレスバーに

```
\\wsl$
```

と入れると、Ubuntuのファイルシステムにアクセスできます。

ここから "home\(自分のアカウント名)" のフォルダにおります。ここがホームディレクトリです。

エクスプローラのアドレスバーでは、次のようになっていると思います。

```
\\wsl$\Ubuntu\home\(自分のユーザ名)
```

ここでフォルダを作ったり、Cファイルを作ったりします。Cのファイルはメモ帳で作成してOKです。

ためしに、ファイル名を a.c として次のようなコードを書いてみましょう。

```C
#include <stdio.h>

int main() {
    printf("Hello World.\n");
    return 0;
}
```

## コンパイル

Ubuntuを立ち上げると、ホームディレクトリにいます。ここで、gcc でコンパイルします。

```
gcc a.c
```

これで、実行可能ファイル a.out ができます。

これを実行してみます。

```
./a.out
```

これで "Hello World."が画面に表示されたらOKです。

# おわりに

いかがだったでしょうか。

WSLのUbuntuを使って、とりあえずC言語のプログラムを書いて実行するところまでやってみました。

VSCodeと合わせると、しっかりスクリーン上でのデバッグも可能になります。


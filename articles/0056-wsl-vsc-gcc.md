---
title: "VSCode+WSL+Ubuntu+gcc+gdb+gmakeで開発してみる"
emoji: "🖥"
type: "tech"
topics: [C, WSL, Ubuntu]
published: true
---

# はじめに

Windows上でVisual Studio Code (VSCode)をインストールして、gccを使った開発を行おうとしたお話です。

## Visual Studio Code を使う

VSCodeは単純なテキストエディタでなく、プラグインを入れるとデバッガとかも使えるようになります。

## Cygwinはうまくいかなかった

VSCode+Cygwinで使おうとしたのですが、ちょっとうまくいかなかったので、WSL Ubuntuを使うことにしました。

## WSL+Ubuntuを導入して gcc, gdb, make を入れておこう

まず、[WSL上でCコンパイラを動かすまで](0055-wsl-gcc) を参考に WSL+Ubuntuを導入して下さい。
また、gcc, gdb, make も入れて下さい。

```bash
sudo apt install make gdb gcc
```

# VScodeにWSLプラグインを入れる

左端の垂直バーにアイコンが並んでいますが、その中の「拡張機能」を選びます。
垂直バーの右側にインストール済のプラグインが表示されますが、その最上部の「Marketplace で拡張機能を検索する」で検索して、インストールします。

## WSLへのアクセス

WSLへは**リモートで接続するかんじ**になります。

左下が「><」となっているとWindows上で動作しています。

![Windows上で動作している場合の左下隅](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0056/01-lb-win.png)

その左下部分をクリックすると、コマンドパレットに「新しいWSLウィンドウ」や「WSLでフォルダを開く」等が表示されます。

だいたい「新しいWSLウィンドウ」でいいのですが、「WSLでフォルダを開く」は、「ドキュメント」等のWindows側のフォルダを開く際に、Windows側から見たパスからWSL側から見たパスへの変換を、VSCode側でやってくれるので、結構ありがたい存在となります。

WSLに繋がっている場合には「>< WSL:Ubuntu」のようになっています。

![Ubuntu上で動作している場合の左下隅](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0056/02-lb-ubt.png)

# 「C/C++拡張機能」プラグインを入れる

左端の垂直バーにアイコンが並んでいますが、その中の「拡張機能」を選びます（さっきと同じ）。

「C/C++」を選択します。

これでプラグインが入りますが、Windows用のみがインストールされた状態です。

## Ubuntuにも入れる

Ubuntuにも入れてあげる必要があります。「C/C++」のすぐ下に表示される「WSL:Ubutu にインストールする」をクリックして下さい。

## 構成設定と .vscode/c_cpp_properties.json ファイル

コマンドパレット (Ctrl+Shift+"p")→「C/C++: Edit Configurations」で設定します。

|名称|値|
|------|-----|
|構成名|Linux|
|コンパイラパス|/usr/bin/gcc|
|コンパイラ引数|(空白)|
|IntelliSenseモード|linux-gcc-x64|
|パスを含める|${workspaceFolder}/**|
|定義|(空白)|
|C標準|gnu17|
|C++標準|gnu++14|


これで ``.vscode/c_cpp_properties.json`` ファイルが出来上がります。

```json
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/gcc",
            "cStandard": "gnu17",
            "cppStandard": "gnu++14",
            "intelliSenseMode": "linux-gcc-x64",
            "configurationProvider": "ms-vscode.makefile-tools"
        }
    ],
    "version": 4
}
```

# とありあえずデバッグ

左端のツールバーからデバッグっぽいアイコンをクリックします。

![実行とデバッグ](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0056/03-debug.png)

さらに「実行とデバッグ」をクリックしてみます。

デバッグの選択肢が現れます。

![デバッグ構成の選択](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0056/04-debug-pre.png)

「コンソール」が現れ、「デバッグ コンソール」が表示されます。

![デバッグ コンソール](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0056/05-console-debug.png)

特に何もなくて困りましたが、ここで「ターミナル」をクリックしてみて下さい。

![ターミナル](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0056/06-console-terminal.png)

結果が出ています。

## ブレークポイントの設定

Cファイルの行番号の左側をクリックすると、ブレークポイントが切り替えられます。

![12行目にブレークポイントを設定した例](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0056/07-breakpoint.png)

「実行とデバッグ」をクリックすると、今度はブレークポイントで止まります。

![12行目で止まっている例](https://github.com/boiledorange73/zenn-content/raw/main/articles-images/0056/08-break.png)

## .vscode/tasks.json ファイル

ここまでで、``.vscode/tasks.json`` ファイルが生成されます。

```json
{
    "tasks": [
        {
            "type": "cppbuild",
            "label": "C/C++: gcc-11 アクティブなファイルのビルド",
            "command": "/usr/bin/gcc-11",
            "args": [
                "-fdiagnostics-color=always",
                "-g",
                "${file}",
                "-o",
                "${fileDirname}/${fileBasenameNoExtension}"
            ],
            "options": {
                "cwd": "${fileDirname}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "detail": "デバッガーによって生成されたタスク。"
        }
    ],
    "version": "2.0.0"
}
```

# makeを使う場合

いくつかのファイルを作成することで、makeも通るようになります。

## Makefile

Makefileは、たとえば次のようにしてみます。

ここで、コンパイルオプションに ``-g`` を指定することを忘れないで下さい。

```
CCFLAGS=-g
CFLAGS=$(CCFLAGS)

a.out:
	$(CC) $(CFLAGS) a.c -o a.out

clean:
	+rm a.out *~
```

## .vscode/launch.json ファイル

``.vscode/launch.json``ファイルを作成します。

ここで``"program": "${workspaceFolder}/a.out",``に気を付けて下さい。実行ファイルを指定しています。ここは Makefile と合うように設定して下さい。

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(GDB) Debug for WSL",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/a.out",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "/usr/bin/gdb",
            "setupCommands": [
                {
                    "description": "gdb の再フォーマットを有効にする",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "My Make for WSL"
        }
    ]
}
```

## .vscode/tasks.json ファイル (上書き)

上記のままで実行しようとすると"My Make for WSL"が存在しない、と怒られます。

``.vscode/tasks.json``に"My Make for WSL"のタスクを作成します。

```json
{
    "tasks": [
        {
            "label": "My Make for WSL",
            "type": "cppbuild",
            "command": "/usr/bin/make",
            "args": [],
            "group": "build",
            "options": {
                "cwd": "${workspaceFolder}",
                "env": {
                   "PATH": "/usr/bin"
                }
            }
        }
    ],
    "version": "2.0.0"
}
```

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

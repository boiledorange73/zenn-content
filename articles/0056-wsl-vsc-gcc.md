---
title: "VSCodeでWSLにアクセスする"
emoji: "🖥"
type: "tech"
topics: [C, WSL, Ubuntu]
published: false
---

# はじめに

Windows上でVisual Studio Code (VSCode)をインストールして、gccを使った開発を行おうとしたお話です。


## Visual Studio Code を使う

VSCodeは単純なテキストエディタでなく、プラグインを入れるとデバッガとかも使えるようになります。

## Cygwinはうまくいかなかった

VSCode+Cygwinで使おうとしたのですが、ちょっとうまくいかなかったので、WSL Ubuntuを使うことにしました。

# VScodeにWSLプラグインを入れる

左端の垂直バーにアイコンが並んでいますが、その中の「拡張機能」を選びます。
垂直バーの右側にインストール済のプラグインが表示されますが、その最上部の「Marketplace で拡張機能を検索する」で検索して、インストールします。

## WSLへのアクセス

WSLへは**リモートで接続するかんじ**になります。

左下が「><」となっているとWindows上で動作しています。

その左下部分をクリックすると、コマンドパレットに「新しいWSLウィンドウ」や「WSLでフォルダを開く」等が表示されます。

だいたい「新しいWSLウィンドウ」でいいのですが、「WSLでフォルダを開く」は、「ドキュメント」等のWindows側のフォルダを開く際に、Windows側から見たパスからWSL側から見たパスへの変換を、VSCode側でやってくれるので、結構ありがたい存在となります。

WSLに繋がっている場合には「>< WSL:Ubuntu」のようになっています。

# 「C/C++拡張機能」プラグインを入れる

左端の垂直バーにアイコンが並んでいますが、その中の「拡張機能」を選びます（さっきと同じ）。

「C/C++」を選択します。

これでプラグインが入りますが、Windows用のみがインストールされた状態です。

## Ubuntuにも入れる

Ubuntuにも入れてあげる必要があります。「C/C++」のすぐ下に表示される「WSL:Ubutu にインストールする」をクリックして下さい。

## 構成

コマンドパレット (Ctrl+Shift+"p")→「C/C++: Configurations Edit」で設定します。

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

</dl>

## .vscode/launth.json ファイルの作成

プロジェクトフォルダを開いたら、``.vscode/launch.json``ファイルを作成します。

```json
{
    // IntelliSense を使用して利用可能な属性を学べます。
    // 既存の属性の説明をホバーして表示します。
    // 詳細情報は次を確認してください: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(GDB) Debug for WSL",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/a.exe",
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

## .vscode/c_cpp_properties.json ファイルの作成

``.vscode/c_cpp_properties.json`` ファイルも作成します。

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


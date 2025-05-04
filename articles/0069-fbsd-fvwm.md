---
title: "VirtualBox で FreeBSD+FVWM を立ち上げる"
emoji: "😀"
type: "tech"
topics: [FreeBSD,VirtualBox,Windows]
published: true
---

# はじめに

これまで FreeBSD はサーバ機として動かしていたのですが、ふと思い立って、VirtualBox + FreeBSD + FVWM をなんとしても動かしたくなりました。

まだ出来上がっていませんが、とりあえず上げておきます。

# 仮想マシン設定

* 名前とオペレーティングシステム
  * 名前: 192.168.56.70
  * ISOイメージ: (ダウンロードした .iso イメージ)
  * タイプ: BSD
  * バージョン: FreeBSD (64-bit)
* ハードウェア
  * メインメモリー: 2048MB
  * プロセッサ数; 1
* ハードディスク
  * サイズ: 256GB
  * ハードディスクのファイルタイプ: VDI

## 作成後の設定

仮想マシンの設定を開きます。
* システム
  * マザーボード
    * 「ハードウェアクロックをUTCに」をチェック
* ネットワーク
  * アダプター2
    * 「有効化」にチェック
    * 割り当て: ホストオンリーアダプター
    * 名前: (適切なホストオンリーアダプター名)
* ディスプレイ
  * スクリーン (virtualbox-ose-additions のメッセージから)
    * グラフィックスコントローラー: VBoxVGA
    * 「3Dアクセラレーションを有効化」は**無効に**

# インストール

## インストール設定と初期インストール

* キーマップ: Japanese 106 (テストで " が出るかどうか確認)
* ホスト名: pc70
* ディストリビューション: 全部なし
* ネットワーク: em0 を IPv4, DHCP に設定、IPv6は設定しない
* パーティション: "Entire Disk"(ディスク全域を使用)
  * ファイルシステム: Auto (UFS)
  * パーティションスキーム: GPT
  * パーティション
    * ada0: (256G) MBR none
      * ada0s1: (256GB) BSD none
        * ada0s1a: 64GB freebsd-ufs /
        * ada0s1b: 4GB freebsd-swap none
        * ada0s1d: (残り) freebsd-ufs /home
* ダウンロード先: 最初のものでいい
* システムコンフィギュレーション
  * rootパスワード
  * タイムゾーン: Asia → Japan
  * サービス: sshd, ntpd だけチェック、他は入れない。
  * セキュリティハードニング: なし
  * Add User Account: ひとつは作っておく
    * Username: (好きなように)
    * Uid: (空白)
    * Login group [(ユーザ名)]: wheel
    * other groups: (空白)
    * Login class [default]: (空白)
    * Shell (sh csh tcsh nologin) [sh]: tcsh
    * (以後パスワード入力まで全て空白)
    * Enter password: (パスワード)
    * Enter password again: (パスワード)
    * (以後全て空白)
* Manual Configuration (手動変更): "No"
* 終了後の動作: リブート
* リブート前に仮想マシンウィンドウのメニューバーの「デバイス」→「光学ドライブ」→「仮想ドライブからディスクを除去」を選択

## 再起動後

### ホストオンリーアダプターを有効にする
``/etc/rc.conf``を編集します。

```
ifconfig_em1="inet 192.168.56.70 netmask 255.255.255.0"
defaultrouter="10.0.2.2"
```

### freebsd-update は kernel だけにする
``vi /etc/freebsd-update.conf`` を編集します。

```
Components kernel
```

## アップデート

```
# freebsd-update fetch
# freebsd-update install
# pkg
```
(``pkg``でpkgngをインストールします)

# portsの設定

https://docs.freebsd.org/en/books/handbook/ports/#ports-using にある通り、git を使います。

```
pkg install git
git clone --depth 1 https://git.freebsd.org/ports.git /usr/ports
```

## ports のアップデート

```
git -C /usr/ports pull
```

# sudoを利用可能にする

```
pkg install sudo
visudo
```

wheelは全員sudo可能にする。

```
%wheel ALL=(ALL:ALL) ALL
```

## wheelグループに入れる方法

```
pw groupmod wheel -m (ユーザ名)
```

# GUI環境の準備

```
pkg install xorg
```

``/etc/rc.conf``

```
# x11
dbus_enable="YES"
```
(haldはなくなりました)


## xorgのキーボードを日本語対応にする

``xinit`` で X が立ち上がるが、キーボードが英語配列で認識されているので、日本語にします。

``/usr/local/etc/X11/xorg.conf.d/keyboard.conf``

```
Section "InputClass"
  Identifier       "KeyboardDefaults"
  MatchIsKeyboard  "on"
  Option           "XkbModel" "jp106"
  Option           "XkbLayout" "jp"
EndSection
```

## 日本語フォント

```
pkg install ja-font-vlgothic ja-font-bizud-gothic ja-font-bizin-gothic ja-font-bizud-mincho
```

## xtermのeightBitSelectTypes

"use "eightBitSelectTypes" XTerm resource setting." と言われたので、

``~/.Xresources``

```
xterm*eightBitSelectTypes: true
```

## virtualbox-ose-additions 6 を入れる

7 の Guest Additions がちょっと入れられなかったので、6 としています。6 は難なく入ります。

```
pkg install virtualbox-ose-additions
```

``/etc/rc.conf``

```
# vbox additions
vboxguest_enable="YES"
vboxservice_enable="YES"
```

## xinitrcで画面サイズを設定する

まず Xサーバの端末エミュレータから ``xrandr`` でディスプレイサイズを確認します。

```
% xrandr
Screen 0: minimum 64 x 64, current 720 x 400, maximum 32766 x 32766
VGA-0 connected primary 720x400+0+0 0mm x 0mm
   720x400       60.00*+
   2560x1600     60.00
   2560x1440     60.00
   2048x1536     60.00
   1920x1600     60.00
   1920x1080     60.00
   1600x1200     60.00
   1680x1050     60.00
   1400x1050     60.00
   1280x1024     60.00
   1024x768      60.00
   800x600       60.00
   640x480       60.00
```

ここから ``1024x768`` にすることとしました。

## クリップボード共有がホスト→ゲストはうまくいくがゲスト→ホストはうまくいかない 

コピペ
https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=254266

``~/.xinitrc`` に次のコマンドを入れ込んでみると、各種サービスが開始されます。ホスト→ゲストは OK になります。でも、ゲスト→ホストで失敗します。
```
VBoxClient --clipboard &
```

## fvwmを入れる

```
pkg install fvwm
```

## xinitrc

```
# xinitrc

# enables clipboard
VBoxClient --clipboard &
VBoxClient --seamless &
VBoxClient --draganddrop &

# changes windows size
xrandr --output VGA-0 --mode 1024x768
# shows xterm
xterm -C -g 80x24+10+10 &
# starts window manager
#twm
fvwm
```

## fvwmの設定

いったん落ち着いた後でももうちょっと前でもいいですが、fvwm の設定をしていきます。

```
cd
mkdir .fvwm
cd .fvwm
cp /usr/local/share/fvwm/default-config/config .
vi config
```

マウスカーソルがデスクトップの端で静止したら仮想デスクトップが変更されるのを防ぎ、また、カーソルキー上で ``xterm`` が起動するのを防ぐようにします。

```
...
#EdgeResistance 450
EdgeResistance -1
...
#Silent Key Super_R A A Exec exec $[infostore.terminal]
...
```

# 共有フォルダ

* マネージャーからインスタンスの設定を開く
* 「共有フォルダー」タブを指定→右端のフォルダ追加ボタンをクリック
* 「共有フォルダーの追加」ダイアログで次のことを指定
  * フォルダーのパス: 適切なパス
  * フォルダー名: shared
  * マウントポイント: (空)
  * 読み込み専用: (チェックを外す)
  * 自動マウント: (チェックを外す)

## ためしにマウントする

https://legacyos.ichmy.0t0.jp/virtualbox/

```
mkdir /mnt/shared
mount_vboxvfs -w shared /mnt/shared
```

## 起動時にマウントする

``/etc/fstab``ではどうもうまくいかないので``/etc/rc.local``で``mount_vboxvfs``を動かすようにしました。

``/etc/rc.local``

```
/usr/local/sbin/mount_vboxvfs -w shared /mnt/shared
```

``/usr/local/sbin/`` にパスが通ってないので、フルパス指定が必要です。

# その他メモ

## FontPath を見たい時

```
pkg info -aD | grep FontPath
```

## X11 modulesを見たい時

```
grep LoadModule /var/log/Xorg.0.log
```

## xset

```
xset -q |& less
```

# おわりに

いかがだったでしょうか。

もう少し確認しないといけないところがあります。特にクリップボード共有がホスト→ゲストはうまくいくがゲスト→ホストはうまくいかないのは大問題です。

それはともかく、とりあえずメモ的に公開しておきます。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。

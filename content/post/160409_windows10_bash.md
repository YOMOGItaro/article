+++
date = "2016-04-09T19:14:33+09:00"
draft = false
title = "Widows10 の bash を試してみる"

+++

[Windows10 の build 14316 から bash が利用できるようになる](https://blogs.windows.com/windowsexperience/2016/04/06/announcing-windows-10-insider-preview-build-14316/
)との事だったので試してみる。
いままで Windows のマシンを使うときは、 Windows 上に VM 立てたり、 Windows 用の Emacs と mingw 組み合わせて使ってたりしたけど、 Ubuntu on Windows で代替できるか試したい。
Xming 使って、 ウインドウモードで起動できたら嬉しい。


# セットアップ
## Windows 10 にする
とりあえず、  Windows 8 を Windows 10 にする。
1時間くらいかかった。

## 開発者モードにする
* 設定から開発者モードにする。
* 再起動する。

## Insider Preview ビルドを取ってくる
* Insider Preview の登録をする
* Insider のレベルをファーストにする
ここが全然進まない。

# そのた
windows 10 に仮想デスクトップが追加されていてよかった。

- Win + Ctrl + D: 新規デスクトップ
- Win + Ctrl + カーソル：デスクトップ移動

上下に配置したり、移動時のアニメーションを無効化したり、一番右端のデスクトップから、一番左端のデスクトップに移動できると嬉しいけど、設定が見つからなかった。

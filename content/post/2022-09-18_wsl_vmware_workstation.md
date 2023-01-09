---
title: "WSL2とVMware Workstation の nested ESXi を同居させたい"
date: 2022-09-18T18:35:29+09:00
draft: true
---

## 困ったこと

WSL2 インストールしたら、 VMware Workstation 上の VM でエラーメッセージが出るようになってしまった。

```
仮想 Intel VT-x/EPT はこのプラットフォームではサポートされていません。
仮想化された Intel VT-x/EPT を使用せずに続行しますか。
```

WSL2 をインストールすると `Intel VT-x/EPT` を占有するとかで、
VMware Workstation で使えなくなってしまうらしい。

`Intel VT-x/EPT` を利用しなくても ESXi は起動はできるが、ESXi 上の VM が起動できない。  
困った。

Windowsの機能で「Windows ハイパーバイザープラットフォーム」を追加しても、
 `Intel VT-x/EPT` を有効化して起動できるわけではなかった。

Windowsの機能で「仮想マシン プラットフォーム」、「Linux 用 Windows サブシステム」を無効化しても、
VMware Workstation で `Intel VT-x/EPT` が有効化できる状態に戻らなかった。

とりあえず、 VMware Workstation 上で nested vSpehre を動かすのは一回あきらめる。

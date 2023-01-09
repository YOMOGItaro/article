---
title: "おうちのノートPCでvSphereを試したい"
date: 2022-09-17T13:19:54+09:00
draft: true
categories: [vmware]
tags: [vmware,tech,study]
---

おうちのノートPCでお手軽にvSphereを試したい。ノートPC上で個人向けの無償ライセンスや評価版だけで環境を作る。VMware Workstationをインストールして、その上にnestedのvSphere環境を立てた。1時間くらいで試せる。

## 環境

* Windows 11
* CPU Corei5-1135G7 (4コア)
* MEM 16GB
* DISK 500GB
* ESXi 7.0
* vCenter 7.0

## VMware Workstation Player の準備

ダウンロードする。  
https://www.vmware.com/jp/products/workstation-player/workstation-player-evaluation.html

インストールする、ウィザードはデフォルトの設定のままで進めた。
拡張仮想キーボードドライバはいらなそう。
PATHへのコンソールツール追加はデフォルトでオンになっている。

すぐに終わる。


## nested ESXi の VM を作る

ダウンロード方法はここに書かれている通り。    
https://docs.vmware.com/jp/VMware-vSphere/7.0/com.vmware.esxi.install.doc/GUID-016E39C1-E8DB-486A-A235-55CAB242C351.html

VMware vSphere の評価版をダウンロードするサイトに移動する。
VMwareのアカウントを作ってログインする必要がある。
`VMware vSphere Hypervisor (ESXi ISO) image` を手動ダウンロードする。

VMware Workstation Playerを開く。新規仮想マシンの作成を選択。
ゲストOSのインストール方法を聞かれるので、さっきダウンロードしたESXiのISOからのインストールを指定する。
デフォルトで142GBが入力されているが今回は最小構成にしたいので32GBにする。
メモリとCPUはデフォルトで最小構成ので2vCPU、4GB MEMが入力されていた。

ESXiのハードウェア要件  
https://docs.vmware.com/jp/VMware-vSphere/7.0/com.vmware.esxi.install.doc/GUID-DEB8086A-306B-4239-BF76-E354679202FC.html

VMが起動してインストールの読み込み画面になるので入力できるようになるまで2分くらい待った。
インストール先ディスク(ローカルの32GBへ)、キーボードレイアウト(Japaneseに)、パスワードなど聞かれるので入力する。
すべてが終わると再起動する。2分くらいで起動した。

こちらを使うとOVAデプロイのみで済みそうなので、楽かもしれない  
https://williamlam.com/nested-virtualization/nested-esxi-virtual-appliance

## vCenter を作る

https://docs.vmware.com/jp/VMware-vSphere/7.0/com.vmware.vcenter.install.doc/GUID-11468F6F-0D8C-41B1-82C9-29284630A4FF.html
この辺りを見ながら、試用版のisoをダウンロードする。

ワークステーションのドキュメントにvCenterのインストール方法が書いてある。ovaかovfでということ。
https://docs.vmware.com/jp/VMware-Workstation-Player-for-Windows/16.0/com.vmware.player.win.using.doc/GUID-8AADBDF8-7369-45CB-95E5-CF210B0A03C9.html

先ほどダウンロードしたisoの中のvcsaフォルダを見るとovaファイルが存在する。
仮想マシンを開くを選択。ovaを選択。
するとウィザードが出てくる。

スペックが指定できるので最小のスペックを指定。ドキュメント上のハードウェア要件の選択肢があって親切。
https://docs.vmware.com/jp/VMware-vSphere/7.0/com.vmware.vcenter.install.doc/GUID-88571D8A-46E1-464D-A349-4DC43DCAF320.html
ディスクのサイズが500GBくらい割り当てられているが、シンプロビジョニングでデプロイ後20GB程度だったので実際はそんなに必要ない。
CPUは2つでそこまで気にならないけど、MEMが12GBと多い、大体実際に利用するのは5GBらしい。
https://williamlam.com/2020/03/homelab-considerations-for-vsphere-7.html

Networking ConfigurationはDHCPで配布されるのでスキップ。
SSO Configuration, System Configuration のパスワード設定は入れないといけないので入れておく。

それから1分くらいで「Welcome to Photon」と書かれたログイン画面が出たり青い画面がでたりするが何もしないで2分ほど待つと黒い画面に「VMware vCenter Server」と書かれたものが出るのでそこで起動完了。
アドレスがDHCPで振られているので https://192.168.154.130:5480/ にアクセスすると、まだ何かしている1分くらい待った。

いろいろ聞かれるので入力する。(もしかしたら、最初のウィザードで入力していたらスキップできるのかも)
またしばらく待つ30分くらいかかる。

## vCenter にnested ESXi を登録する

## 動かす

## NSX-T のインストール

https://customerconnect.vmware.com/group/vmware/evalcenter?p=nsx-t-eval

## アップデート 3min+ダウンロード3分

デフォルトの設定にしているとVMware Player起動時にポップアップが出てくる。

アップデートする。
ダウンロードが開始する。

ダウンロードが終わると新規うウィンドウでウィザードが表示される。
VMware Workstation Player が起動していると次に進まないので閉じる。
選択肢は、インストール時と同様の内容だった、デフォルトの設定のまま進めた。

VMware Playerのアップデートが完了後、再度起動してみた。
VMはしっかり残っていた。


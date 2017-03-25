+++
date = "2015-07-20T22:16:50+09:00"
draft = false
title = "ブログ始めるときに困ったこと"

+++

# どういうふうにブログを作ったか #

この blog は下記のものでできている。

* hugo
* github pages

github pages は下記の２通りの方法があるようだった。

* username.github.io という名前のリポジトリを作る
* 任意のリポジトリに gh-pages というブランチを作る

後者を利用することにした。
master ブランチにブログのソースを置いて,
ビルドした結果を公開ブランチにするほうが良さそうだと思った。

hugo の最上部のブランチを master,
public ディレクトリを gh-pages にした。

主に下記のサイトを参考にした。

* http://gohugo.io/overview/introduction/
* https://help.github.coms/user-organization-and-project-pages/#project-pages

# 構築中に詰まったところ #

## go インストール後パスを通したのに go がないと言われる ##

go を実行したら下記のようなメッセージが出てきた。
```zsh
$ go
zsh: no such file or directory: {{go のパス}}/go
```
下記を見なおしたが問題が無かった。

* PATH 設定
* GOROOT 設定
* GOPATH 設定

入れていた go のバイナリが間違っていたことがわかり、
入れなおしたらうまく動いた。
64bit が必要だが Linux 32-bit を入れてしまっていた。
恥ずかしいミスだった。

## hugo テーマがうまく当たらない ##

~~dddddhugo のテーマがうまく当たらなかった。
サイトのパスのトップに css や js がないとうまく動かなかった。
パスを設定できるようにして当たるようにした。~~

(2017/03/25 修正)

サイトのパスにサブディレクトリを含み、コンテンツをサブディレクトリ以下に置くときには、`config.toml` に `CanonifyUrls` を設定しないといけなかった。

``` diff
  baseurl = "http://yomogitaro.github.io/article"
+ canonifyurls=true
  languageCode = "ja"
  title = "よもぎさんのへや"
  pygmentscodefences = true
```

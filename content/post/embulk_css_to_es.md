+++
date = "2015-07-20T22:53:57+09:00"
draft = false
title = "embulk で csv を elasticsearch に移したい"

+++

# やりたいこと #

* csv のファイルの内容を elasticsearch に移行したい

# 試した方法 #

embulk を利用すればできそうなので試してみた。

## テスト用 elasticsearch の準備　 ##

* elasticsearch を入れる
```
% docker run -d --name es1 elasticsearch
```
* IP を確認して動いているか確認
```
% docker inspect --format '{{ .NetworkSettings.IPAddress }}' es1
% curl 'http://172.17.0.4:9200/_stats'
{"_shards":{"total":0,"successful":0,"failed":0},"_all":{"primaries":{},"total":{}},"indices":{}}% 
```
* ドキュメントを UI から確認するために kibana も用意
```
% docker run --link es1:elasticsearch -d --name kibana01 kibana
% docker inspect --format '{{ .NetworkSettings.IPAddress }}' kibana01
```
* http://172.17.0.6:5601 動いているか見てみる

## embulk のインストール ##

* java が必要なので入れる
```
% sudo apt-get install openjdk-8-jdk 
```
* ここに書いてあるとおりに入れてみる
https://github.com/embulk/embulk#linux--mac--bsd

* サンプルを動かしてみる
```
% embulk example ./try1
% embulk guess ./try1/example.yml -o config.yml
% embulk preview config.yml
% embulk run config.yml
```

## embulk elasticsearch プラグインのインストール ##

* elasticsearch の output-plugin をインストール
```
% embulk gem install embulk-output-elasticsearch
```
* example を下記のように編集
```
in:
  type: file
  path_prefix: "/tmp/try1/csv/sample_"
out:
  type: elasticsearch
  nodes:
  - {host: 172.17.0.4, port: 9300}
  index: embulk
  index_type: embulk
```
* config 再生成して実行
```
% embulk guess ./try1/example.yml -o config.yml
% embulk run config.yml
```
* うまくいっているか確認
```
curl 'http://172.17.0.4:9200/embulk/_search' | jq .
```

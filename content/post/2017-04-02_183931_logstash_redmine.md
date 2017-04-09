+++
date = "2017-04-02T18:39:31+09:00"
draft = true
title = "logstash を使って redmine の可視化をできるようにする"
+++

redmine は可視化する機能が少ない印象がある。
また、 redmine の情報と他のシステムの情報を関連付けて解析をしたかったりする。
logstash に redmine の output プラグインがあるが、 input のプラグインはない。
作ってためしてみたい。
elastic の 5.3 系が出たばかりだしいろいろ試したかったが、
プラグイン作って動かすところまでしかできなかった。

# 方針

- [この方のブログ](https://future-architect.github.io/articles/20160920/)と同じ情報をとりたい。
- API でとりたい。
- アップデートがあるところだけ更新するようにしたい。
  - 最後に検索した時刻をキャッシュとして持っていれば良さそう
  - すごく厳密にやろうと思ったら、 ES の中に入っている一番新しい updated_on を検索条件に入れればよいが、 input プラグインとしては、良い選択ではないかもしれない

# logstash input plugin を作る

とりあえずここを参考にした。
https://www.elastic.co/guide/en/logstash/current/_how_to_write_a_logstash_input_plugin.html


## 環境を作ってサンプルを動かす

このサンプルをみるといいらしい。ホスト名とメッセージを送っているみたい。とりあえずこれを動かしてみる。
https://github.com/logstash-plugins/logstash-input-example/

jruby のプラグインを作ることになるのでこの辺を入れた。
https://github.com/rbenv/rbenv
https://github.com/rbenv/ruby-build
jruby-1.7.26 を入れた。
とりあえず環境を作る。

``` zsh
 % rbenv install jruby-1.7.26
 % rbenv local jruby-1.7.26
 % jruby --version
 % rbenv exec gem install bundler
Fetching: bundler-1.14.6.gem (100%)
Successfully installed bundler-1.14.6
1 gem installed
 % rbenv exec bundle
 % rbenv exec bundle exec rspec
The signal EXIT is in use by the JVM and will not work correctly on this platform
Using Accessor#strict_set for specs
Run options: exclude {:redis=>true, :socket=>true, :performance=>true, :couchdb=>true, :elasticsearch=>true, :elasticsearch_secure=>true, :export_cypher=>true, :integration=>true, :windows=>true}

Randomized with seed 49369
.

Finished in 1.12 seconds (files took 2.18 seconds to load)
1 example, 0 failures

Randomized with seed 49369
```

logstash の Gemfile に以下の行を追加する
``` zsh
gem "logstash-input-example", :path => "/home/yomogi/sandbox/170326/logstash-input-example"
```
logstash でプラグインをインストールしようとしたらエラー。
プラグインと logstash で `logstash-core` のバージョンがあっていないと怒られる。
``` zsh
 % ./bin/logstash-plugin install --no-verify
Installing...
Plugin version conflict, aborting
ERROR: Installation Aborted, message: Bundler could not find compatible versions for gem "logstash-core":
  In snapshot (Gemfile.lock):
    logstash-core (= 5.2.2)

  In Gemfile:
    logstash-core-plugin-api (>= 0) java depends on
      logstash-core (= 5.2.2) java

    logstash-input-example (>= 0) java depends on
      logstash-core (< 3.0.0, >= 2.0.0) java

    logstash-core (>= 0) java

Running `bundle update` will rebuild your snapshot from scratch, using only
the gems in your Gemfile, which may resolve the conflict.
```
3.0.0 未満の制約を外して動くか試すことにした。
``` zsh
-  s.add_runtime_dependency "logstash-core", ">= 2.0.0", "< 3.0.0"
+  s.add_runtime_dependency "logstash-core", ">= 2.0.0"
```
インストールした。無事に成功した。
``` zsh
 % ./bin/logstash-plugin install --no-verify
Installing...
Installation successful
```
config 適当に作った。
``` zsh
input {
  example {
    message => "hello hello world!"
    interval => 5
  }
}
output {
  stdout { codec => rubydebug }
}
```
動いた
``` zsh
 % ./bin/logstash -f ../logstash.config
Sending Logstash's logs to /home/yomogi/sandbox/170326/logstash-5.2.2/logs which is now configured via log4j2.properties
[2017-03-26T21:22:22,979][INFO ][logstash.pipeline        ] Starting pipeline {"id"=>"main", "pipeline.workers"=>4, "pipeline.batch.size"=>125, "pipeline.batch.delay"=>5, "pipeline.max_inflight"=>500}
[2017-03-26T21:22:23,139][INFO ][logstash.pipeline        ] Pipeline main started
{
    "@timestamp" => 2017-03-26T12:22:23.159Z,
          "host" => "xxxxxxxx",
      "@version" => "1",
       "message" => "hello hello world!"
}
[2017-03-26T21:22:23,184][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}
{
    "@timestamp" => 2017-03-26T12:22:28.161Z,
          "host" => "xxxxxxxx",
      "@version" => "1",
       "message" => "hello hello world!"
}
C^A^C[2017-03-26T21:22:29,477][WARN ][logstash.runner          ] SIGINT received. Shutting down the agent.
[2017-03-26T21:22:29,492][WARN ][logstash.agent           ] stopping pipeline {:id=>"main"}
```

## プラグインを作る
ここの話に戻る。
https://www.elastic.co/guide/en/logstash/current/_how_to_write_a_logstash_input_plugin.html

プラグインを作る方法として、 `use_the_plugin_generator_tool` と `copy_the_input_code` が提案されている。
今回は `use_the_plugin_generator_tool` を試してみる。
``` zsh
 % ./bin/logstash-plugin generate --type input --name redmineissue --path ~/sandbox/170326/
 Creating /home/yomogi/sandbox/170326/logstash-input-redmineissue
         create logstash-input-redmineissue/LICENSE
         create logstash-input-redmineissue/Gemfile
         create logstash-input-redmineissue/Rakefile
         create logstash-input-redmineissue/README.md
         create logstash-input-redmineissue/DEVELOPER.md
         create logstash-input-redmineissue/lib/logstash/inputs/redmineissue.rb
         create logstash-input-redmineissue/CHANGELOG.md
         create logstash-input-redmineissue/spec/inputs/redmineissue_spec.rb
         create logstash-input-redmineissue/CONTRIBUTORS
         create logstash-input-redmineissue/logstash-input-redmineissue.gemspec
```
いろいろ出来た。それぞれのファイルにテンプレートが書かれていた。
``` zsh
 % tree logstash-input-redmineissue
logstash-input-redmineissue
|-- CHANGELOG.md
|-- CONTRIBUTORS
|-- DEVELOPER.md
|-- Gemfile
|-- LICENSE
|-- README.md
|-- Rakefile
|-- lib
|   `-- logstash
|       `-- inputs
|           `-- redmineissue.rb
|-- logstash-input-redmineissue.gemspec
`-- spec
    `-- inputs
        `-- redmineissue_spec.rb
```

`logstash-input-redmineissue/lib/logstash/inputs/redmineissue.rb` を書く必要がある。
[redmine の output プラグイン](https://github.com/logstash-plugins/logstash-output-redmine) を見ながら書いてみる。redmineissue.rb を編集して、適当に作ってみた。

``` ruby
# encoding: utf-8
require "logstash/inputs/base"
require "logstash/namespace"
require "stud/interval"
require "socket" # for Socket.gethostname
require 'json'

# Generate a repeating message.
#
# This plugin is intented only as an example.

class LogStash::Inputs::Redmineissue < LogStash::Inputs::Base
  config_name "redmineissue"

  # If undefined, Logstash will complain, even if codec is unused.
  default :codec, "plain"

  config :url, :validate => :string, :required => true
  config :token, :validate => :string, :required => true

  public
  def register
    require 'net/http'
    require 'uri'
    require 'json'

  end # def register

  def issues(offset=0, limit=100)
    begin
      @post_format = 'json'
      @formated_url = "#{@url}/issues.#{@post_format}?offset=#{offset}&limit=#{limit}"
      @uri = URI(@formated_url)
      @http = Net::HTTP.new(@uri.host, @uri.port)
      @req = Net::HTTP::Get.new("#{@uri.path}?#{@uri.query}", @header)
      res = @http.request(@req)
      if res.code == '200'
        resp_j = JSON.parse(res.body, {:symbolize_names => true})
      end

      data = resp_j[:issues]
      data.each do |issue|
        yield issue
      end

      total = resp_j[:total_count]
      offset = offset + limit
    end while offset < total
  end

  def run(queue)
    begin
      issues do |issue|
        event = LogStash::Event.new("issue" => issue)
        decorate(event)
        queue << event
      end
#    rescue => e
#      @logger.warn("Failed to get redmine issues")
    end

  end # def run
end # class LogStash::Inputs::Redmineissue
```
作成したプラグインをインストールした。
``` zsh
cd logstash-5.3.0
echo 'gem "logstash-input-redmineissue", :path => "/home/yomogi/sandbox/170326/logstash-input-redmineissue"' >> Gemfile
./bin/logstash-plugin install --no-verify
```

## おためし環境を作る
適当に ES と kibana と redmine を立ち上げる
``` zsh
docker pull docker.elastic.co/elasticsearch/elasticsearch:5.3.0
docker pull docker.elastic.co/kibana/kibana:5.3.0
docker pull redmine:3.3
docker run -p 9200:9200 --name yomo-es docker.elastic.co/elasticsearch/elasticsearch:5.3.0
docker run --name yomo-kibana -p 5601:5601 --link yomo-es --env "ELASTICSEARCH_URL=http://yomo-es:9200" docker.elastic.co/kibana/kibana:5.3.0
docker run --name yomo-redmine -p 3000:3000 redmine:3.3
```
redmine にログインして、 API アクセスできるようにしておく。
適当にランダムのデータを入れるようにした。
``` ruby
# coding: utf-8
require 'net/http'
require 'uri'
require 'json'

REDMINE = "http://localhost:3000"
TOKEN = "write api token here"
header = { 'Content-Type' => 'application/json', 'X-Redmine-Api-Key' => "#{TOKEN}" }

def post(path, body)
  uri = URI("#{REDMINE}/#{path}.json")
  http = Net::HTTP.new(uri.host, uri.port)
  header = { 'Content-Type' => 'application/json', 'X-Redmine-Api-Key' => "#{TOKEN}" }
  req = Net::HTTP::Post.new(uri.path, header)
  req.body = JSON.dump(body)
  res = http.request(req)
end

def get(path)
  uri = URI("#{REDMINE}/#{path}.json")
  http = Net::HTTP.new(uri.host, uri.port)
  header = { 'Content-Type' => 'application/json', 'X-Redmine-Api-Key' => "#{TOKEN}" }
  req = Net::HTTP::Get.new(uri.path, header)
  res = http.request(req)

  JSON.parse(res.body, {:symbolize_names => true})
end

def userids
  users = get("users")
  users[:users].map{|u| u[:id]}
end

def add_user(login, password, firstname, lastname, mail)
  user = {
    "user" => {
      "login" => "#{login}",
      "password" => "#{password}",
      "firstname" => "#{firstname}",
      "lastname" => "#{lastname}",
      "mail" => "#{mail}"
    }
  }
  post("users", user)
end

def add_project(name, identifier)
  project = {
    "project" => {
      "name" => "#{name}",
      "identifier" => "#{identifier}"
    }
  }
  post("projects", project)
end

def add_category(project, name)
  category = {
    "issue_category" => {
      "name" => "#{name}"
    }
  }
  post("projects/#{project}/issue_categories", category)
end

def add_member_to_project(project, member, role)
  members = {
    "membership" => {
      "user_id" => "#{member}",
      "role_ids" => [
        "#{role}"
      ]
    }
  }
  post("projects/#{project}/memberships", members)
end

def add_version_to_project(project, name)
  version = {
    "version" => {
      "name" => "#{name}"
    }
  }
  post("projects/#{project}/versions", version)
end

def add_issue(name, project, tracker, priority, status, user, category, version)
  issue = {
    "issue" => {
      "subject" => "#{name}",
      "project_id" => "#{project}",
      "tracker_id" => "#{tracker}",
      "priority_id" => "#{priority}",
      "status_id" => "#{status}",
      "assigned_to_id" => "#{user}",
      "category_id" => "#{category}",
      "version_id" => "#{version}"
    }
  }
end

def add_random_issue(count)
  projects = get("projects")[:projects].map{|i| i[:id]}
  trackers = get("trackers")[:trackers].map{|i| i[:id]}
  priorities = get("enumerations/issue_priorities")[:issue_priorities].map{|i| i[:id]}
  statuses = get("issue_statuses")[:issue_statuses].map{|i| i[:id]}
  users = get("users")[:users].map{|i| i[:id]}

  categories = {}
  versions = {}
  projects.each do |project|
    categories[project] = get("projects/#{project}/issue_categories")[:issue_categories].map{|i| i[:id]}
    versions[project]   = get("projects/#{project}/versions")[:versions].map{|i| i[:id]}
  end
  puts categories
  puts versions

  count.times do |ite|
    selected_project = projects.sample
    puts "#{selected_project}, #{categories[selected_project].sample}, #{versions[selected_project].sample}"
    add_issue(
      "ticket#{ite}",
      projects.sample,
      trackers.sample,
      priorities.sample,
      statuses.sample,
      users.sample,
      categories[selected_project].sample,
      versions[selected_project].sample
    )
  end
end

add_user("manager1",   "passpass", "管理者", "1", "manager1@example.net"  )
add_user("manager2",   "passpass", "管理者", "2", "manager2@example.net"  )
add_user("developer1", "passpass", "開発者", "1", "managerloper1@example.net")
add_user("developer2", "passpass", "開発者", "2", "managerloper2@example.net")
add_user("developer3", "passpass", "開発者", "3", "managerloper3@example.net")

add_project("projectA", "projecta")
add_project("projectB", "projectb")

add_category("projecta", "サーバー")
add_category("projecta", "ネットワーク")
add_category("projecta", "ストレージ")
add_category("projectb", "炊事")
add_category("projectb", "掃除")
add_category("projectb", "洗濯")
add_category("projectb", "買出し")

userids.each do |u|
  add_member_to_project("projecta", u, 3)
  add_member_to_project("projectb", u, 3)
end

add_version_to_project("projecta", "v0.1")
add_version_to_project("projecta", "v0.2")
add_version_to_project("projecta", "v1.0")
add_version_to_project("projectb", "2017-1Q")
add_version_to_project("projectb", "2017-2Q")

add_random_issue(10)
```
logstash.config を書く
``` text
input {
  redmineissue {
    url => "http://localhost:3000"
    token => "write api token here"
  }
}
output {
    stdout {
      codec => rubydebug
    }
    elasticsearch {
      hosts => ["localhost:9200"]
          user => "elastic"
          password => "changeme"
    }
}
```
logstash を実行した。
```
./bin/logstash -f logstash.config
```
いろいろ分析できるようになった。

![xxx](/images/kibana009.png)


# おまけ docker で logstash 動かす
docker で logstash を動かしてみた。自分でプラグインを作った時にどうするのが良いかは考え中。

`~/hogehoge/pipeline/logstash.config` は試しに date を1秒ごとに実行してみた。
``` text
input {
  exec {
    command => date
    interval => 1
  }
}
output {
  stdout { codec => rubydebug }
}
```
`~/hogehoge/settings/logstash.conf` は es で監視できるようにする設定が自動でオンになるようなのでオフにした。
``` text
xpack.monitoring.enabled: false
```

実行した、うまく行った。
``` zsh
 % docker run --rm -it -v ~/hogehoge/pipeline:/usr/share/logstash/pipeline -v ~/hogehoge/settings:/usr/share/logstash/config docker.elastic.co/logstash/logstash:5.3.0
Could not find log4j2 configuration at path /usr/share/logstash/config/log4j2.properties. Using default config which logs to console
08:27:52.378 [main] INFO  logstash.setting.writabledirectory - Creating directory {:setting=>"path.queue", :path=>"/usr/share/logstash/data/queue"}
08:27:52.393 [LogStash::Runner] INFO  logstash.agent - No persistent UUID file found. Generating new UUID {:uuid=>"204a3ed0-f1d4-440b-863e-fb58d9b0e542", :path=>"/usr/share/logstash/data/uuid"}
08:27:52.598 [[main]-pipeline-manager] INFO  logstash.pipeline - Starting pipeline {"id"=>"main", "pipeline.workers"=>4, "pipeline.batch.size"=>125, "pipeline.batch.delay"=>5, "pipeline.max_inflight"=>500}
08:27:52.610 [[main]-pipeline-manager] INFO  logstash.inputs.exec - Registering Exec Input {:type=>nil, :command=>"date", :interval=>1}
08:27:52.615 [[main]-pipeline-manager] INFO  logstash.pipeline - Pipeline main started
08:27:52.713 [Api Webserver] INFO  logstash.agent - Successfully started Logstash API endpoint {:port=>9600}
{
    "@timestamp" => 2017-04-01T08:27:52.869Z,
      "@version" => "1",
          "host" => "03c079666c80",
       "message" => "Sat Apr  1 08:27:52 UTC 2017\n",
       "command" => "date"
}
{
    "@timestamp" => 2017-04-01T08:27:53.625Z,
      "@version" => "1",
          "host" => "03c079666c80",
       "message" => "Sat Apr  1 08:27:53 UTC 2017\n",
       "command" => "date"
}
{
    "@timestamp" => 2017-04-01T08:27:54.631Z,
      "@version" => "1",
          "host" => "03c079666c80",
       "message" => "Sat Apr  1 08:27:54 UTC 2017\n",
       "command" => "date"
}
^C08:27:54.888 [SIGINT handler] WARN  logstash.runner - SIGINT received. Shutting down the agent.
08:27:54.919 [LogStash::Runner] WARN  logstash.agent - stopping pipeline {:id=>"main"}
```

# まとめ

始めた時はいろいろやろうと思ってたけど中途半端になってしまった。やりたかったけどできなかったことを書き残しておく。
- 複数回実行した時に同じ issue が行かないようにする。更新アイテムだけ取得する方式、更新日で検索するようにしておいて、検索条件の更新日を記録しておくとか。
- 分析方法を考える。
  - 多分 journal 一覧を使って issue status の推移と担当者を記録したりするとおもしろいとおもった。
  - kibana ダッシュボードの良いテンプレートを作る。
- logstash を embulk 的な使い方をしてるけどそれで良いのか調べる
- docker で自作プラグインを使う時の良い方法を考える

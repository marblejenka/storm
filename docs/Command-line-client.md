---
title: Command Line Client
layout: documentation
documentation: true
---
このページでは、 "storm"コマンドラインクライアントで可能なすべてのコマンドについて説明します。"storm"クライアントを設定してリモートクラスタと通信する方法については、[Setting up development environment](Setting-up-development-environment.html)）の指示に従ってください。

コマンドは次のとおりです:

1. jar
1. kill
1. activate
1. deactivate
1. rebalance
1. repl
1. classpath
1. localconfvalue
1. remoteconfvalue
1. nimbus
1. supervisor
1. ui
1. drpc
1. blobstore
1. dev-zookeeper
1. get-errors
1. heartbeats
1. kill_workers
1. list
1. logviewer
1. monitor
1. node-health-check
1. pacemaker
1. set_log_level
1. shell
1. sql
1. upload-credentials
1. version
1. help

### jar

Syntax: `storm jar topology-jar-path class ...`

引数に指定された`class`のmainメソッドを実行します。`〜/ .storm`にあるStormのjarと設定はクラスパスに置かれます。このプロセスは、トポロジをsubmitする際に、その設定を使用して[StormSubmitter](javadocs/org/apache/storm/StormSubmitter.html)は`topology-jar-path`にあるjarをアップロードします。

### kill

Syntax: `storm kill topology-name [-w wait-time-secs]`

名前が`topology-name`であるトポロジを終了させます。Stormはまず、トポロジのメッセージがタイムアウトになるまでトポロジのスパウトを無効にして、現在処理中のすべてのメッセージの処理が完了するようにします。Stormはワーカーをシャットダウンし、実行状態をクリーンアップします。無効化からシャットダウンまでStormが待機する時間の長さを、-wフラグを使用して上書きできます。

### activate

Syntax: `storm activate topology-name`

指定されたトポロジのSpoutを有効化します。

### deactivate

Syntax: `storm deactivate topology-name`

指定されたトポロジのSpoutを無効化します。

### rebalance

Syntax: `storm rebalance topology-name [-w wait-time-secs] [-n new-num-workers] [-e component=parallelism]*`

トポロジのワーカーが実行されている場所を広げたい場合があります。たとえば、ノードごとに4つのワーカー実行する10ノードのクラスタがあるとし、さらに10ノードをクラスタに追加するとします。各ノードが2つのワーカーを実行するように、Stormは実行中のトポロジを分配することができます。これを行うための1つの方法は、トポロジーを終了して再送信することですが、Stormはこれを行う簡単な方法であるる"rebalance"コマンドを提供します。

リバランスでは、まず、メッセージタイムアウトの期間（-wフラグで上書き可能）だけトポロジを無効にしてから、ワーカーをクラスタ全体に均等に再分配します。その後、トポロジは以前の有効な状態に戻ります（無効化されていたトポロジは無効化されたままで、有効化されていたトポロジは有効な状態に戻ります）。

また、rebalanceコマンドを使用して、実行中のトポロジーのparallelismを変更することもできます。-nおよび-eスイッチを使用して、コンポーネントのワーカー数またはエグゼキュータの数をそれぞれ変更します。

### repl

Syntax: `storm repl`

クラスパス上のStormのjarと設定を使用して、Clojure REPLを開きます。デバッグに便利です。

### classpath

Syntax: `storm classpath`

コマンドを実行するときにStormのクライアントが使用するクラスパスを出力します。

### localconfvalue

Syntax: `storm localconfvalue conf-name`

ローカルにおけるStormの設定で、`conf-name`の値を表示します。ローカルのStormの設定は、`~/.storm/storm.yaml`の設定が`defaults.yaml`の設定とマージされたものです。

### remoteconfvalue

Syntax: `storm remoteconfvalue conf-name`

クラスタにおけるStormの設定で、`conf-name`の値を表示します。クラスタのStormの設定は`$STORM-PATH/conf/storm.yaml`にあるもので、`defaults.yaml`の設定とマージされています。 このコマンドは、クラスタマシン上で実行する必要があります。

### nimbus

Syntax: `storm nimbus`

Nimbusデーモンを起動します。このコマンドは、[daemontools](http://cr.yp.to/daemontools.html)や[monit](http://mmonit.com/monit/)のようなツールの監視下で実行する必要があります。詳細は、[Storm clusterの設定](Setting-up-a-Storm-cluster.html)を参照してください。

### supervisor

Syntax: `storm supervisor`

Supervisorデーモンを起動します。このコマンドは、[daemontools](http://cr.yp.to/daemontools.html)や[monit](http://mmonit.com/monit/)のようなツールの監視下で実行する必要があります。詳細は、[Storm clusterの設定](Setting-up-a-Storm-cluster.html)を参照してください。

### ui

Syntax: `storm ui`

UIデーモンを起動します。UIはStormクラスタのWebインターフェイスを提供し、トポロジの実行に関する詳細な統計情報を表示します。このコマンドは、[daemontools](http://cr.yp.to/daemontools.html)や[monit](http://mmonit.com/monit/)のようなツールの監視下で実行する必要があります。詳細は、[Storm clusterの設定](Setting-up-a-Storm-cluster.html)を参照してください。

### drpc

Syntax: `storm drpc`

DRPCデーモンを起動します。 このコマンドは、[daemontools](http://cr.yp.to/daemontools.html)や[monit](http://mmonit.com/monit/)のようなツールの監視下で実行する必要があります。詳細は、[Storm clusterの設定](Setting-up-a-Storm-cluster.html)を参照してください。

### blobstore

Syntax: `storm  cmd`

list [KEY...] - blobに現在あるblobをリストします

cat [-f FILE] KEY - blobを読み込み、それをファイルに書き込むか、STDOUT（読み込みアクセスが必要です）のいずれかに書き込みます。

create [-f FILE] [-a ACL ...] [--replication-factor NUMBER] KEY - 新しいblobを作成します。内容はFI​​LEまたはSTDINから与えられます。ACLは[uo]:[username]:[r-][w-][a-]の形式でカンマ区切りリストにすることができます。

update [-f FILE] KEY - blobの内容を更新します。内容はFI​​LEまたはSTDINから与えられます（書き込みアクセスが必要です）。

delete KEY - blobstoreからエントリを削除します（書き込みアクセスが必要です）。

set-acl [-s ACL] KEY - ACLの形式は[uo]:[username]:[r-][w-][a-]で、コンマ区切りリスト（管理者アクセスが必要）です。

replication --read KEY - blobのreplication factorを読み取るために使用されます。

replication --update --replication-factor NUMBER KEY - ここで、NUMBER>0です。これは、blobのreplication factorを更新するために使用されます。

たとえば、次の例では、data.tgzに格納されたデータを使用してmytopo:data.tgzキーを作成します。ユーザーaliceはフルアクセス権を持ち、bobは読み取り/書き込みアクセス権を持ち、他のすべてのユーザーは読み取りアクセス権を持ちます。

storm blobstore create mytopo:data.tgz -f data.tgz -a u:alice:rwa,u:bob:rw,o::r

詳細については、[Blobstore(Distcahce)](distcache-blobstore.html)を参照してください。

### dev-zookeeper

Syntax: `storm dev-zookeeper`

"dev.zookeeper.path"をローカルディレクトリとして、"storm.zookeeper.port"をポートとして使用して、新しいZookeeperサーバーを起動します。これは開発/テストのみを目的としており、起動したZookeeperインスタンスは本番環境で使用するように構成されていません。

### get-errors

Syntax: `storm get-errors topology-name`

実行中のトポロジから最新のエラーを取得します。返される結果には、コンポーネント名のキー値のペアならびにエラーのあるコンポーネントのエラーが含まれます。結果はjson形式で返されます。

### heartbeats

Syntax: `storm heartbeats [cmd]`

list PATH - ClusterStateに現在あるPATHのheartbeats nodesを一覧表示します。
get  PATH - PATHにあるheartbeat dataを取得します

### kill_workers

Syntax: `storm kill_workers`

このsupervisorで動作するワーカーを停止させます。このコマンドは、supervisorノードで実行する必要があります。クラスタがセキュアモードで実行されている場合、すべてのワーカーを正常に終了させるためには、ノードに対する管理者権限が必要です。

### list

Syntax: `storm list`

実行中のトポロジとその状態を一覧表示します。

### logviewer

Syntax: `storm logviewer`

log viewerデーモンを起動します。これは、Stormのログファイルを表示するためのWebインターフェイスを提供します。このコマンドは、daemontoolsやmonitのようなツールを使って監視下で実行する必要があります。

詳細については、Stormクラスタの設定を参照してください（http://storm.apache.org/documentation/Setting-up-a-Storm-cluster）。

### monitor

Syntax: `storm monitor topology-name [-i interval-secs] [-m component-id] [-s stream-id] [-w [emitted | transferred]]`

トポロジのスループットをインタラクティブに監視します。
poll-interval, component-id, stream-id, watch-item[emitted | transferred]を指定できます。
  デフォルトでは、
    poll-intervalは4秒です;
    すべてのコンポーネントIDがリストとして渡されます;
    stream-id は 'default';
    watch-item は 'emitted';

### node-health-check

Syntax: `storm node-health-check`

ローカルにあるsupervisorにヘルスチェックを実行します。

### pacemaker

Syntax: `storm pacemaker`

Pacemakerデーモンを起動します。このコマンドは、daemontoolsやmonitのようなツールを使って監視下で実行する必要があります。

詳細については、Stormクラスタの設定を参照してください（http://storm.apache.org/documentation/Setting-up-a-Storm-cluster）。

### set_log_level

Syntax: `storm set_log_level -l [logger name]=[log level][:optional timeout] -r [logger name] topology-name`

トポロジのログレベルを動的に変更します
    
ログレベルはALL, TRACE, DEBUG, INFO, WARN, ERROR, FATAL, OFFのいずれかです

e.g.
  ./bin/storm set_log_level -l ROOT=DEBUG:30 topology-name

  ルートロガーのレベルを30秒間DEBUGに設定する

  ./bin/storm set_log_level -l com.myapp=WARN topology-name

  com.myappロガーのレベルをWARNに30秒間設定する

  ./bin/storm set_log_level -l com.myapp=WARN -l com.myOtherLogger=ERROR:123 topology-name

   com.myappロガーのレベルをWARNに設定し、com.myOtherLoggerをERRORに123秒間設定します

  ./bin/storm set_log_level -r com.myOtherLogger topology-name

  設定を消去し、元のレベルに戻します

### shell

Syntax: `storm shell resourcesdir command args`

非JVM言語を使用するためにjarを構築し、Nimbusにアップロードします

eg: `storm shell resources/ python topology.py arg1 arg2`

### sql

Syntax: `storm sql sql-file topology-name`

SQL文をTridentトポロジにコンパイルし、Stormに送信します。

### upload-credentials

Syntax: `storm upload_credentials topology-name [credkey credvalue]*`

実行中のトポロジに新しいcredentialの集合をアップロードします。

### version

Syntax: `storm version`

実行するStormリリースのバージョン番号を表示します。

### help
Syntax: `storm help [command]`

指定されたコマンドのヘルプメッセージまたは利用可能なコマンドのリストを表示する

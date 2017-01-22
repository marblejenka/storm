---
title: Setting up a Storm Cluster
layout: documentation
documentation: true
---
このページでは、Stormクラスタを起動して実行する手順を説明します。AWSを利用している場合は、[storm-deploy](https://github.com/nathanmarz/storm-deploy/wiki)プロジェクトをチェックしてください。[storm-deploy](https://github.com/nathanmarz/storm-deploy/wiki)は、EC2上のStormクラスタのプロビジョニング、設定、インストールを完全に自動化します。また、CPU、ディスク、ネットワークの使用状況を監視できるように、Gangliaを設定します。

Stormクラスタで問題が発生した場合、最初に解決策が[トラブルシューティング](Troubleshooting.html)ページにあるかどうかを確認してください。なければ、メーリングリストにメールしてください。

Stormクラスタを設定する手順の概要を以下に示します:

1. Zookeeperクラスタをセットアップする
2. Nimbusとワーカーマシンに依存関係をインストールする
3. NimbusとワーカーマシンにStormリリースをダウンロードして解凍する
4. 必須の設定をstorm.yamlに入力する
5. "storm"スクリプトを使ってスーパーバイザーとその監視下に置かれるデーモンを起動する

### Set up a Zookeeper cluster

StormはZookeeperを使用してクラスタを調整します。Zookeeperはメッセージの受け渡しには使用**されない**ので、StormにおけるZookeeperの負荷はかなり低くなります。ほとんどの場合、単一ノードのZookeeperクラスタで十分ですが、フェールオーバーが必要な場合や、大きなStormクラスタを展開する場合は、より大きなZookeeperクラスタが必要な場合があります。Zookeeperを配備する手順は[こちら](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html)です。

Zookeeper導入に関するいくつかの注意点:

1. Zookeeperはフェイル・ファストであり、エラーが発生した場合にプロセスを終了するので、監視のもとでZookeeperを実行することが重要です。詳細については、[こちら](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html#sc_supervision)を参照してください。
2. Zookeeperのデータとトランザクションログを圧縮にするためにcronを設定することが重要です。Zookeeperデーモンはこれを単独では実行しません。また、cronを設定しないと、Zookeeperはすぐにディスク領域を使い果たします。詳細については、[こちら](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html#sc_maintenance)を参照してください。

### Install dependencies on Nimbus and worker machines

次に、Nimbusとワーカーマシンに以下のStormの依存関係をインストールする必要があります:

1. Java 7
2. Python 2.6.6

これらはStormでテストされた依存関係のバージョンです。StormはJavaやPythonの異なるバージョンで動作する場合と動作しない場合があります。


### Download and extract a Storm release to Nimbus and worker machines

次に、Stormリリースをダウンロードし、Nimbusと各ワーカーマシンのどこかでzipファイルを解凍します。Stormのリリースは[こちらから](http://github.com/apache/storm/releases)からダウンロードできます。

### Fill in mandatory configurations into storm.yaml

Stormリリースの`conf/storm.yaml`がStormデーモンを設定するファイルです。デフォルトの設定値は[ここ]({{page.git-blob-base}}/conf/defaults.yaml)で見ることができます。storm.yamlはdefaults.yaml内のものをオーバーライドします。クラスタを稼働させるために必須の設定がいくつかあります:

1) **storm.zookeeper.servers**: あなたのStormクラスタ用のZookeeperクラスタ内のホストのリストです。次のようになります:

```yaml
storm.zookeeper.servers:
  - "111.222.333.444"
  - "555.666.777.888"
```

Zookeeperクラスタが使用するポートがデフォルトと異なる場合は、**storm.zookeeper.port**も設定する必要があります。

2) **storm.local.dir**: NimbusとSupervisorのデーモンは、小さな状態（jar、confsなど）を格納するために、ローカルディスク上のディレクトリを必要とします。
各マシンにそのディレクトリを作成し、適切な権限を与えてから、この設定にディレクトリの場所を入れておく必要があります。例えば:

```yaml
storm.local.dir: "/mnt/storm"
```
あなたがWindows上でStormを走らせると、それは次のようになります:

```yaml
storm.local.dir: "C:\\storm-local"
```
相対パスを使用すると、stormをインストールした場所(STORM_HOME)からの相対的なパスになります。
空にするとデフォルト値`$STORM_HOME/storm-local`が使用されます。

3) **nimbus.seeds**: ワーカーは、トポロジーのjarとconfをダウンロードするために、どのマシンがマスターの候補かを知る必要があります。例えば:

```yaml
nimbus.seeds: ["111.222.333.44"]
```
**マシンのFQDN**の値を記入することをお勧めします。Nimbus H/Aを設定する場合は、すべてのマシンのFQDNをアドレス指定する必要があります。'疑似分散'クラスタを設定するだけの場合は、デフォルト値のままでも構いませんが、それでもFQDNを記入することをお勧めします。

4) **supervisor.slots.ports**: ワーカーマシンごとに、このマシンで実行するワーカーの数を設定します。各ワーカーはメッセージを受信するために単一のポートを使用し、この設定では、使用するために開いているポートを定義します。ここに5つのポートを定義すると、Stormはこのマシン上で実行するワーカーを5つまで割り当てます。 3つのポートを定義すると、Stormは最大3つまでしか動作させません。デフォルトでは、この設定はポート6700,6701,6702および6703で4つのワーカーを実行するように設定されています:

```yaml
supervisor.slots.ports:
    - 6700
    - 6701
    - 6702
    - 6703
```

### Monitoring Health of Supervisors

Stormは、ノードが正常であるかどうかを判断するために、管理者が提供するスクリプトを定期的に実行するようスーパーバイザを設定できるメカニズムを提供しています。管理者は、スーパバイザにstorm.health.check.dirにあるスクリプトで任意のチェックを実行させることによって、ノードが健全な状態にあるかどうかを判断させることができます。スクリプトがノードが不健全な状態になったことを検出する際に、文字列ERRORで始まる行を標準出力に出力する必要があります。スーパーバイザは定期的にヘルスチェックディレクトリ内のスクリプトを実行し、出力をチェックします。上記のように、スクリプトの出力にERRORという文字列が含まれている場合、スーパーバイザはワーカーをシャットダウンして終了します。

スーパバイザが"/bin/storm node-health-check"による監視とともに実行されている場合、スーパバイザはそのスクリプトを使って、自身を起動するか、あるいはノードが正常かを判断できます。

ヘルスチェックディレクトリの場所は、次のように設定できます:

```yaml
storm.health.check.dir: "healthchecks"
```

スクリプトには実行権限が必要です。
特定のヘルスチェックスクリプトがタイムアウトにより失敗したとマークされるまでの時間は、次のように設定できます:

```yaml
storm.health.check.timeout.ms: 5000
```

### Configure external libraries and environmental variables (optional)

外部ライブラリやカスタムプラグインのサポートが必要な場合は、そのようなjarファイルをextlib/やextlib-daemon/ディレクトリに置くことができます。extlib-daemon/ディレクトリには、デーモン（Nimbus、Supervisor、DRPC、UI、Logviewer）だけが使用するHDFSやカスタマイズされたスケジューリングライブラリなどが格納されています。したがって、2つの環境変数STORM_EXT_CLASSPATHとSTORM_EXT_CLASSPATH_DAEMONは、外部クラスパスとデーモンのみの外部クラスパスを含むようにユーザーが構成できます。

### Launch daemons under supervision using "storm" script and a supervisor of your choice

最後のステップは、すべてのStormデーモンを起動することです。これらのデーモンのそれぞれを監視下で実行することが重要です。Stormは予期しないエラーが発生した場合にプロセスが停止する__fail-fast__システムです。Stormは、任意の時点で安全に停止し、プロセスの再起動時に正しく回復できるように設計されています。これは、Stormが進行中の状態を保持していない理由です -- Nimbusまたはスーパーバイザが再起動した場合、実行中のトポロジは影響を受けません。Stormデーモンを実行する方法は次のとおりです:

1. **Nimbus**: マスタマシンにおいて監視のもと"bin/storm nimbus"コマンドを実行します。
2. **Supervisor**: 各ワーカーマシンにおいて監督のもと"bin/storm supervisor"コマンドを実行します。スーパバイザデーモンは、そのマシン上のワーカープロセスの起動と停止を行います。
3. **UI**: 監督のもとでコマンド"bin/storm ui"を実行することによって、Storm UI（クラスタとトポロジ上の診断を提供するブラウザからアクセスできるサイト）を実行します。 Uは、Webブラウザでhttp://{ui host}:8080を開くことでアクセスできます。

ご覧のとおり、デーモンの実行は非常に簡単です。デーモンは、Stormリリースを解凍した場所のlogs/ディレクトリにログを記録します。

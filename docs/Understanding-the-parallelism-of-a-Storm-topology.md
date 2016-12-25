---
title: Understanding the Parallelism of a Storm Topology
layout: documentation
documentation: true
---
## What makes a running topology: worker processes, executors and tasks

Stormは、Stormクラスタで実際にトポロジを実行するために使用される、次の3つの主なエンティティを区別します。

1. ワーカープロセス
2. エグゼキュータ（スレッド）
3. タスク

以下が、それらの関係を単純に図示したものです:

![The relationships of worker processes, executors (threads) and tasks in Storm](images/relationships-worker-processes-executors-tasks.png)

_ワーカープロセス_は、トポロジのサブセットを実行します。ワーカープロセスは特定のトポロジに属し、このトポロジの1つまたは複数のコンポーネント（SpoutまたはBolt）に対して1つまたは複数のエグゼキュータを実行できます。実行中のトポロジは、Stormクラスタ内の多くのマシンで実行されている多くのプロセスで構成されています。

_エグゼキュータ_は、ワーカープロセスによって生成されるスレッドです。同じコンポーネント（SpoutまたはBolt）に対して1つ以上のタスクを実行することがあります。

_タスク_は、実際のデータ処理を実行します - アプリケーションコードで実装するSpoutまたはBoltは、クラスタ全体で多くのタスクを実行します。コンポーネントのタスクの数は、トポロジの生存期間を通じて常に同じですが、コンポーネントのエグゼキュータ（スレッド）の数は時間とともに変化します。これは、次の条件が当てはまることを意味します: ``#threads ≤ #tasks``。デフォルトでは、タスクの数はエグゼキュータの数と同じに設定されます。つまり、Stormはスレッドごとに1つのタスクを実行します。

## Configuring the parallelism of a topology

Stormの用語では、"parallelism"とは、コンポーネントのエグゼキュータ（スレッド）の初期数を意味し、いわゆる_parallelism hint_と呼ばれるものを記述するために使用されることに注意してください。このドキュメントでは、エグゼキュータの数だけでなく、ワーカープロセスの数とStormトポロジのタスク数を設定する方法を説明するために、より一般的な意味で"parallelism"という用語を使用します。Stormの狭義の"parallelism"を使用する場合、特に断りを入れます。

以下のセクションでは、さまざまな設定オプションの概要と、それらをコードで設定する方法について説明します。これらのオプションを設定する方法は複数ありますが、表の中にはそれらのオプションのうちのいくつかしか示していません。Stormは今のところ、以下の[設定の優先順位](Configuration.html)を持っています: ``defaults.yaml`` < ``storm.yaml`` < トポロジ固有の設定 < 内部的なコンポーネント固有の設定 < 外部的なコンポーネント固有の設定。

### Number of worker processes

* 説明: クラスタのマシンで_トポロジ_を生成するワーカープロセスの数。
* 設定オプション: [TOPOLOGY_WORKERS](javadocs/org/apache/storm/Config.html#TOPOLOGY_WORKERS)
* あなたのコードで設定する方法（例）:
    * [Config#setNumWorkers](javadocs/org/apache/storm/Config.html)

### Number of executors (threads)

* 説明: _コンポーネントごとに_いくつのエグゼキュータを生成するか指定します。
* 設定オプション: なし(``setSpout``または``setBolt``に``parallelism_hint``パラメータを渡します)
* あなたのコードで設定する方法（例）:
    * [TopologyBuilder#setSpout()](javadocs/org/apache/storm/topology/TopologyBuilder.html)
    * [TopologyBuilder#setBolt()](javadocs/org/apache/storm/topology/TopologyBuilder.html)
    * Storm 0.8の時点では、``parallelism_hint``パラメータは、そのBoltのエグゼキュータの初期数（タスクではありません！）を指定するようになりました。

### Number of tasks

* 説明: _コンポーネントごとに_生成するするタスクの数。
* 設定オプション: [TOPOLOGY_TASKS](javadocs/org/apache/storm/Config.html#TOPOLOGY_TASKS)
* あなたのコードで設定する方法（例）: 
    * [ComponentConfigurationDeclarer#setNumTasks()](javadocs/org/apache/storm/topology/ComponentConfigurationDeclarer.html)

実際に設定をしているコードスニペットの例を次に示します:

```java
topologyBuilder.setBolt("green-bolt", new GreenBolt(), 2)
               .setNumTasks(4)
               .shuffleGrouping("blue-spout");
```

上記のコードでは、エグゼキュータ2つとそれに関連する4つをタスク数の初期値として与えて、Bolt``GreenBolt``を実行するようにStormを設定しました。Stormはエグゼキュータ（スレッド）ごとに2つのタスクを実行します。明示的にタスク数を設定しない場合、Stormはデフォルトで実行プログラムごとに1つのタスクを実行します。

## Example of a running topology

次の図は、シンプルなトポロジでどのように動作するかを示しているものです。このトポロジは、``BlueSpout``と呼ばれる1つのSpuotと、``GreenBolt``および``YellowBolt``と呼ばれる2つのBoltの3つのコンポーネントで構成されています。これらのコンポーネントは、``BlueSpout``がその出力を``GreenBolt``に送るように結節されています。``GreenBolt``はその出力を``YellowBolt``に送ります。

![Example of a running topology in Storm](images/example-of-a-running-topology.png)

``GreenBolt``は上記のコードスニペットに従って設定されていましたが、``BlueSpout``と ``YellowBolt``はparallelism hint（エグゼキュータの数）のみを設定しています 関連するコードは次のとおりです:

```java
Config conf = new Config();
conf.setNumWorkers(2); // use two worker processes

topologyBuilder.setSpout("blue-spout", new BlueSpout(), 2); // set parallelism hint to 2

topologyBuilder.setBolt("green-bolt", new GreenBolt(), 2)
               .setNumTasks(4)
               .shuffleGrouping("blue-spout");

topologyBuilder.setBolt("yellow-bolt", new YellowBolt(), 6)
               .shuffleGrouping("green-bolt");

StormSubmitter.submitTopology(
        "mytopology",
        conf,
        topologyBuilder.createTopology()
    );
```

もちろん、Stormには、トポロジの並列性を制御するための追加の設定があります:

* [TOPOLOGY_MAX_TASK_PARALLELISM](javadocs/org/apache/storm/Config.html#TOPOLOGY_MAX_TASK_PARALLELISM): この設定は、単一のコンポーネントに対して生成できるエグゼキュータの数に上限を設定します。これは通常、ローカルモードでトポロジを実行するときに生成されるスレッドの数を制限するために、テスト中に使用されます。このオプションは、[Config#setMaxTaskParallelism()](javadocs/org/apache/storm/Config.html#setMaxTaskParallelism(int))で設定できます。 

## How to change the parallelism of a running topology

Stormのすばらしい特徴は、クラスタまたはトポロジを再起動する必要なく、ワーカープロセスやエグゼキュータの数を増減できることです。これはリバランスといいます。

トポロジをリバランスするには、次の2つの方法があります:

1. Storm Web UIを使用して、トポロジをリバランスします。
2. 以下に説明するように、CLIツールからstorm rebalanceを使用します。

CLIツールの使用例を次に示します:

```
## Reconfigure the topology "mytopology" to use 5 worker processes,
## the spout "blue-spout" to use 3 executors and
## the bolt "yellow-bolt" to use 10 executors.

$ storm rebalance mytopology -n 5 -e blue-spout=3 -e yellow-bolt=10
```

## References

* [Concepts](Concepts.html)
* [Configuration](Configuration.html)
* [Running topologies on a production cluster](Running-topologies-on-a-production-cluster.html)]
* [Local mode](Local-mode.html)
* [Tutorial](Tutorial.html)
* [Storm API documentation](javadocs/), 特に``Config``クラス


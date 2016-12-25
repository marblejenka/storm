---
title: Configuration
layout: documentation
documentation: true
---
Stormには、nimbus、supervisors、および実行中のトポロジの動作を調整するためのさまざまな設定があります。一部の設定はシステムに関する設定でありトポロジごとに変更することはできません。一方で、他の設定はトポロジごとに変更することができます。

すべての設定には、Stormのコードベースにある[defaults.yaml]({{page.git-blob-base}}/conf/defaults.yaml)で定義されているデフォルト値があります。これらの設定を上書きするには、Nimbusとsupervisorのクラスパスにあるstorm.yamlで定義します。最後に、[StormSubmitter](javadocs/org/apache/storm/StormSubmitter.html)を使用する際に、トポロジとともに送信するトポロジ固有の設定を定義することができます。ただし、トポロジ固有の設定では、接頭辞が"TOPOLOGY"である設定のみを上書きできます。

Storm 0.7.0以降では、Bolt単位/Bolt単位で設定を上書きにすることができます。このように上書きできるのは、次のもの設定飲みです:

1. "topology.debug"
2. "topology.max.spout.pending"
3. "topology.max.task.parallelism"
4. "topology.kryo.register": これは、シリアライゼーションがトポロジ内のすべてのコンポーネントで使用可能になるため、これは他のものと少し異なります。さらなる詳細は、[Serialization](Serialization.html)にあります。

Java APIを使用すると、コンポーネント固有の構成を2つの方法で指定できます:

1. *Internally:* 任意のSpoutまたはBoltで`getComponentConfiguration`を上書きし、コンポーネント固有のコンフィグレーションの対応付けを返します。
2. *Externally:* `TopologyBuilder`の`setSpout`と`setBolt`は、コンポーネントの設定を上書きするために使用できる`addConfiguration`と`addConfigurations`メソッドを持つオブジェクトを返します。

設定値の優先順位は、defaults.yaml < storm.yaml < トポロジ固有の設定 < 内部コンポーネント固有の設定 < 外部コンポーネント固有の設定、です。

**Resources:**

* [Config](javadocs/org/apache/storm/Config.html): すべての設定の一覧と、トポロジ固有の設定を生成するためのヘルパークラス
* [defaults.yaml]({{page.git-blob-base}}/conf/defaults.yaml): すべての設定のデフォルト値
* [Setting up a Storm cluster](Setting-up-a-Storm-cluster.html): Stormクラスタの構築と設定方法を説明しています
* [Running topologies on a production cluster](Running-topologies-on-a-production-cluster.html): クラスた上でトポロジを実行する場合の便利な設定を挙げています
* [Local mode](Local-mode.html): ローカルモードを使用する場合の便利な設定を挙げています

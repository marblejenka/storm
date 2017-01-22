---
title: Local Mode
layout: documentation
documentation: true
---
ローカルモードは処理中のStormクラスタをシミュレートするもので、トポロジの開発とテストに役立ちます。ローカルモードでのトポロジの実行は、[クラスタ上](Running-topologies-on-a-production-cluster.html)で実行するのと同様です。

インプロセスクラスタを作成するには、単に`LocalCluster`クラスを使用します。例えば:

```java
import org.apache.storm.LocalCluster;

LocalCluster cluster = new LocalCluster();
```

次に、`LocalCluster`オブジェクトの`submitTopology`メソッドを使ってトポロジーを送信することができます。[StormSubmitter](javadocs/org/apache/storm/StormSubmitter.html)の対応するメソッドと同様に、`submitTopology`は名前、トポロジの設定、およびトポロジのオブジェクトを取ります。次に、トポロジ名を引数とする`killTopology`メソッドを使用してトポロジを終了させることができます。

ローカルクラスタをシャットダウンするには、単純に以下のように呼び出します:

```java
cluster.shutdown();
```

### Common configurations for local mode

[ここ](javadocs/org/apache/storm/Config.html)で設定の完全なリストを見ることができます。

1. **Config.TOPOLOGY_MAX_TASK_PARALLELISM**: この設定は、単一のコンポーネントのために生成されたスレッドの数に上限を設定します。 多くの場合、プロダクショントポロジはローカルモードでトポロジをテストしようとすると不当な負荷をかける並列処理（数百スレッド）となってしまいます。この設定で、その並列性を簡単に制御できます。
2. **Config.TOPOLOGY_DEBUG**: これがtrueに設定されている場合、SpoutまたはBoltからタプルがemitされるたびに、Stormはメッセージを記録します。これはデバッグに非常に便利です。

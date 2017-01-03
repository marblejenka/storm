---
title: Trident Spouts
layout: documentation
documentation: true
---
# Trident spouts

通常のStorm APIのように、SpoutはTridentトポロジのストリームのソースです。通常のStormのSpoutの上に、Tridentはより洗練されたSpoutのための追加のAPIを公開します。

データストリームのソースと、それらのデータストリームに基づいて状態(データベースなど)を更新する方法との間には複雑な関係があります。この説明については、[Trident state doc](Trident-state.html)を参照してください - 利用可能なSpoutのオプションを理解するには、この関係性を理解することが不可欠です。

通常のStormのSpoutはTridentトポロジではnon-transactional spoutです。通常のStormのIRichSpoutを使用するには、TridentTopologyで次のようなストリームを作成します:

```java
TridentTopology topology = new TridentTopology();
topology.newStream("myspoutid", new MyRichSpout());
```

TridentトポロジのすべてのSpoutには、ストリームの一意の識別子が必要です - この識別子は、クラスタで実行されるすべてのトポロジで一意でなければなりません。 Tridentは、この識別子を使用して、SpoutがZookeeperで消費したことに関するメタデータ(txidとSpoutに関連付けられているメタデータなど)を格納します。

SpoutメタデータのZookeeperストレージを設定するには、次の設定オプションを使用できます:

1. `transactional.zookeeper.servers`: Zookeeperのホスト名のリスト
2. `transactional.zookeeper.port`: Zookeeperクラスターのポート
3. `transactional.zookeeper.root`: Zookeeperのルートディレクトリで、メタデータが格納されます。メタデータは<root path>/<spout id>のパスに格納されます

## Pipelining

デフォルトでは、Tridentは一度に1つのバッチを処理し、別のバッチを試す前にバッチの成功または失敗を待ちます。バッチをパイプライン化することで、スループットが大幅に向上し、各バッチの処理待ち時間が短縮されます。"topology.max.spout.pending"プロパティを使用して同時に処理するバッチの最大量を設定します。

複数のバッチを同時に処理している最中でも、Tridentはバッチ間でトポロジ内で発生する状態の更新を指示します。 たとえば、グローバル集計をデータベースに格納しているとします。バッチ1のデータベースのカウントを更新している間も、バッチ2〜10の部分カウントを計算することができます。Tridentは、バッチ1の状態更新に成功するまでバッチ2の状態更新に移行しません。これは、exactly-onceの処理セマンティクスを達成するために不可欠で、概要は[Trident state doc](Trident-state.html)で読めます。

## Trident spout types

以下に、使用可能なSpout APIがあります:

1. [ITridentSpout]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/trident/spout/ITridentSpout.java): 最も一般的なAPIで、transactionalとopaque transactionalのセマンティクスをサポートします。一般的に、このAPIを直接使うのではなく、このAPIのパーティション化されたフレーバーを使用します。
2. [IBatchSpout]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/trident/spout/IBatchSpout.java): 一度にタプルをまとめてemitするnon-transactional spout
3. [IPartitionedTridentSpout]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/trident/spout/IPartitionedTridentSpout.java): パーティション化されたデータソース(Kafkaサーバーのクラスタのようなもの)から読み取るtransactional spout
4. [IOpaquePartitionedTridentSpout]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/trident/spout/IOpaquePartitionedTridentSpout.java): パーティション化されたデータソースから読み取るopaque transactional spout

また、このチュートリアルの冒頭で述べたように、通常のIRichSpoutも使用できます。
 


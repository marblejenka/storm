---
title: Distributed RPC
layout: documentation
documentation: true
---
分散RPC(DRPC)の背後にあるアイデアは、Stormを使用して実際に強力な関数の計算を並列で並列化することです。Stormのトポロジは、関数の引数をストリームの入力として取り込み、それらの関数呼び出しのそれぞれの結果についての出力ストリームをemitします。

DRPCはそれほどStormの機能らしい機能ではありません。Stormのストリーム、Spout、Bolt、トポロジなどのプリミティブによって表現されるパターンです。DRPCはStormとは別のライブラリとしてパッケージ化されていたかもしれませんが、Stormにバンドルされているのでとても便利です。

### High level overview

分散RPCは"DRPCサーバー"によって調整されます(Stormにはこの実装がパッケージ化されています)。DRPCサーバーは、RPC要求の受信、Stormトポロジへの要求の送信、Stormトポロジからの結果の受信、および結果を待機中のクライアントに返すことなどを調整します。クライアントから見ると、分散RPC呼び出しは通常のRPC呼び出しと同じように見えます。たとえば、クライアントが引数 "http://twitter.com" を使用して "reach" 関数の結果を計算する方法は次のとおりです。

```java
DRPCClient client = new DRPCClient("drpc-host", 3772);
String result = client.execute("reach", "http://twitter.com");
```

分散RPCのワークフローは次のようになります:

![Tasks in a topology](images/drpc-workflow.png)

クライアントは、実行する関数の名前とその関数の引数をDRPCサーバーに送信します。その機能を実装するトポロジは、DRPCサーバからの関数呼び出しのストリームを受信するために`DRPCSpout`を使用します。各関数の呼び出しは、DRPCサーバーによって一意のIDでタグ付けされます。次に、トポロジは結果を計算し、トポロジの最後で`ReturnResults`と呼ばれるBoltがDRPCサーバに接続し、それに関数呼び出しのIDに対する結果を与えます。DRPCサーバーはそのIDを使用して、結果を待機しているクライアントを照合し、その待機中のクライアントのブロックを解除し、結果を送信します。

### LinearDRPCTopologyBuilder

Stormには、[LinearDRPCTopologyBuilder](javadocs/org/apache/storm/drpc/LinearDRPCTopologyBuilder.html)というtopology builderが付属しており、DRPCを実行するためのほとんどすべてのステップを自動化します。これらには以下のものが含まれます:

1. Spoutのセットアップ
2. 結果をDRPCサーバーに戻す
3. タプルのグループに対して有限の集約を行うためのボルトへの機能の提供

簡単な例を見てみましょう。以下は、入力引数に"！"をつけて返すDRPCトポロジの実装です:

```java
public static class ExclaimBolt extends BaseBasicBolt {
    public void execute(Tuple tuple, BasicOutputCollector collector) {
        String input = tuple.getString(1);
        collector.emit(new Values(tuple.getValue(0), input + "!"));
    }

    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("id", "result"));
    }
}

public static void main(String[] args) throws Exception {
    LinearDRPCTopologyBuilder builder = new LinearDRPCTopologyBuilder("exclamation");
    builder.addBolt(new ExclaimBolt(), 3);
    // ...
}
```

見ての通り、やらないといけないことはほとんどありません。`LinearDRPCTopologyBuilder`を作成するときは、トポロジのDRPC関数の名前を指定します。1つのDRPCサーバーで多くの関数を扱うことができ、関数名で関数を互いに区別します。宣言した最初のBoltは入力として対のタプルをとります。最初のフィールドは要求IDで、2番目のフィールドはその要求の引数です。`LinearDRPCTopologyBuilder`は最後のBoltが[id、result]という形式の対のタプルを含む出力ストリームを出力することを期待しています。最後に、すべての中間タプルには、最初のフィールドとして要求IDが含まれていなければなりません。

この例では、`ExclaimBolt`は単にタプルの2番目のフィールドに"！"を付けて渡します。`LinearDRPCTopologyBuilder`は、DRPCサーバに接続して結果を返す調整の残りの部分を処理します。

### Local mode DRPC

DRPCはローカルモードで実行できます。上記の例をローカルモードで実行する方法は次のとおりです:

```java
LocalDRPC drpc = new LocalDRPC();
LocalCluster cluster = new LocalCluster();

cluster.submitTopology("drpc-demo", conf, builder.createLocalTopology(drpc));

System.out.println("Results for 'hello':" + drpc.execute("exclamation", "hello"));

cluster.shutdown();
drpc.shutdown();
```

最初に`LocalDRPC`オブジェクトを作成します。このオブジェクトは、`LocalCluster`が処理中のStormクラスタをシミュレートするのと同様に、処理中のDRPCサーバをシミュレートします。次に、ローカルモードでトポロジを実行するために `LocalCluster`を作成します。`LinearDRPCTopologyBuilder`は、ローカルトポロジーとリモートトポロジーを作成するための別個のメソッドを持っています。ローカルモードでは、`LocalDRPC`オブジェクトはどのポートにもバインドしないので、トポロジは通信すべきオブジェクトを知る必要があります。これは、`createLocalTopology`が`LocalDRPC`オブジェクトを入力として取り込む理由です。

トポロジを起動した後、`LocalDRPC`で`execute`メソッドを使ってDRPC呼び出しを行うことができます。

### Remote mode DRPC

実際のクラスタでDRPCを使用することも簡単です。3つのステップがあります:

1. DRPCサーバーを起動します
2. DRPCサーバーの場所を設定します
3. DRPCトポロジをStormクラスタに送信する

DRPCサーバの起動は`storm`スクリプトで行うことができ、NimbusやUIを起動するのとだいたい同じです:

```
bin/storm drpc
```

次に、DRPCサーバーの場所をStormクラスターに設定する必要があります。これは、`DRPCSpout`がどこから読み込みの関数呼び出しをするか知る方法です。これは、 `storm.yaml`ファイルまたはトポロジの設定によって行うことができます。`storm.yaml`でこれを設定すると、次のようになります:

```yaml
drpc.servers:
  - "drpc1.foo.com"
  - "drpc2.foo.com"
```

最後に、他のトポロジーを起動するのと同じように、`StormSubmitter`を使用してDRPCトポロジを起動します。リモートモードで上記の例を実行するには、次のようにします:

```java
StormSubmitter.submitTopology("exclamation-drpc", conf, builder.createRemoteTopology());
```

`createRemoteTopology`は、ストームクラスタに適したトポロジを作成するために使用されます。

### A more complex example

exclamationを付けるDRPCの例は、DRPCの概念を説明するための簡単なものの例でした。ストーム・クラスタがDRPC機能を計算するために提供する並列処理が本当に必要な、より複雑な例を見てみましょう。ここでは、TwitterでURL​​のReachを計算する例を示します。

URLのReachとは、特定のURLが提示されたユニークユーザーの数です。リーチを計算するには、以下が必要です:

1. あるURLをツイートしたすべての人を取得する
2. その人たちのフォロワーを全て取得する
3. そのフォロワーの集合を一意にする
4. 一意になったフォロワーの数を数える

1回のReachの計算には、計算中に何千ものデータベース呼び出しと数千万のフォロワーの記録が含まれることがあります。それは実際には本当に激しい計算です。あなたが目にしているように、Stormの上にこの機能を実装するのは簡単ではありません。単一のマシンでは、Reachの計算に数分かかることがあります; Stormクラスターでは、数秒で最も難しいURLについてのReachを計算することができます。

Reachを計算するトポロジのサンプルは、storm-starterの[ここ]({{page.git-blob-base}}/examples/storm-starter/src/jvm/org/apache/storm/starter/ReachTopology.java)で定義されていますeachを計算するトポロジを定義する方法は次のとおりです:

```java
LinearDRPCTopologyBuilder builder = new LinearDRPCTopologyBuilder("reach");
builder.addBolt(new GetTweeters(), 3);
builder.addBolt(new GetFollowers(), 12)
        .shuffleGrouping();
builder.addBolt(new PartialUniquer(), 6)
        .fieldsGrouping(new Fields("id", "follower"));
builder.addBolt(new CountAggregator(), 2)
        .fieldsGrouping(new Fields("id"));
```

トポロジは4つのステップで実行されます:

1. `GetTweeters`はURLをツイートしたユーザーを取得します。`[id, url]`の入力ストリームを`[id, tweeter]`の出力ストリームに変換します。各`url`タプルは複数の` tweeter`タプルにマップされます。
2. `GetFollowers`はツイーターのフォロワーを取得します。`[id, tweeter]`の入力ストリームを`[id, follower]`の出力ストリームに変換します。すべてのタスクにわたって、同じURLをツイートした複数の人をフォローしている人がいると、フォロワータプルが重複することがあります。
3. `PartialUniquer`は、フォロワーのストリームをフォロワーIDでグループ化します。これによって、同じフォロワーを同じタスクに集約せることができます。したがって、 `PartialUniquer`の各タスクは、互いにに独立したフォロワーのセットを受け取ります。`PartialUniquer`は、要求IDに対するすべてのフォロワーのタプルを受け取ると、そのフォロワーの部分集合に対する一意のカウントを出力します
4. 最後に、`CountAggregator`は、`PartialUniquer`タスクのそれぞれから部分カウントを受け取り、それを合計してReachの計算が完了します

`PartialUniquer`のBoltを見てみましょう:

```java
public class PartialUniquer extends BaseBatchBolt {
    BatchOutputCollector _collector;
    Object _id;
    Set<String> _followers = new HashSet<String>();
    
    @Override
    public void prepare(Map conf, TopologyContext context, BatchOutputCollector collector, Object id) {
        _collector = collector;
        _id = id;
    }

    @Override
    public void execute(Tuple tuple) {
        _followers.add(tuple.getString(1));
    }
    
    @Override
    public void finishBatch() {
        _collector.emit(new Values(_id, _followers.size()));
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("id", "partial-count"));
    }
}
```

`PartialUniquer`は`BaseBatchBolt`を継承して`IBatchBolt`を実装します。バッチボルトは、具象的なまとまりとしてタプルのバッチを処理するためのファーストクラスのAPIを提供します。バッチボルトの新しいインスタンスが各要求IDごとに作成され、Stormは必要な場合にインスタンスをクリーンアップします。

`PartialUniquer`は、`execute`メソッドでフォロワーのタプルを受け取ると、それを内部の`HashSet`内のリクエストIDの集合に追加します。

バッチボルトは、このタスクが対象とするバッチのすべてのタプルが処理された後に呼び出される`finishBatch`メソッドを提供します。そのコールバックでは、 `PartialUniquer`は、フォロワーIDのサブセットに対する一意としたカウントを含む単一のタプルを出力します。

内部的には、指定された要求IDに対して与えられたボルトがすべてのタプルを受け取ったことを検出するため`CoordinatedBolt`が使用されます。`CoordinatedBolt`は、この調整を管理するために直接ストリームを利用します。

残りのトポロジは自明です。ご覧のように、Reach計算のすべてのステップが並行して行われ、DRPCトポロジの定義は非常に簡単でした。

### Non-linear DRPC topologies

`LinearDRPCTopologyBuilder`は、"linear"なDRPCトポロジしか処理しません。ここで、計算は一連のステップ（リーチなど）として表現されます。Boltの分岐やマージなど複雑なトポロジを必要とする機能を想像するのは難しいことではありません。今のところ、これを行うには `CoordinatedBolt`を直接使用する必要があります。 DRPCトポロジのより一般的な抽象概念をつくるために、メーリングリスト上でnon-linearなDRPCトポロジのユースケースについて話し合ってください。

### How LinearDRPCTopologyBuilder works

* DRPCSpoutは[args, return-info]を発行します。return-infoは、DRPCサーバーのホストとポート、およびDRPCサーバーによって生成されたIDです
* トポロジを構築するには以下のものが使用できます:
  * DRPCSpout
  * PrepareRequest (要求IDを生成し、return-infoのストリームとargsのストリームを作成します)
  * CoordinatedBoltのラッパーと直接グループ化
  * JoinResult (戻り値とreturn-infoを結合する)
  * ReturnResult (DRPCサーバーに接続して結果を返します)
* LinearDRPCTopologyBuilderは、Stormのプリミティブの上に構築されたより高いレベルの抽象化の良い例です

### Advanced
* 複数のリクエストの処理を同時に行えるKeyedFairBolt
*  `CoordinatedBolt`を直接使う方法

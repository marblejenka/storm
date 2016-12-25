---
title: Concepts
layout: documentation
documentation: true
---

このページには、Stormの主な概念と、詳細情報があるリソースへのリンクが掲載されています。議論される概念は次のとおりです:

1. Topologies
2. Streams
3. Spouts
4. Bolts
5. Stream groupings
6. Reliability
7. Tasks
8. Workers

### Topologies

リアルタイムアプリケーションのロジックは、Stormのトポロジにパッケージ化されています。Stormのトポロジは、MapReduceジョブに似ています。主な相違点の1つは、MapReduceジョブがいつかは終了するのに対し、トポロジは永遠に実行されることです（もちろん、それを強制終了するまで）。トポロジは、ストリームの集合に接続されたSpoutとBoltのグラフです。これらの概念については以下で説明します。

**Resources:**

* [TopologyBuilder](javadocs/org/apache/storm/topology/TopologyBuilder.html): このクラスを使用して、Javaでトポロジを構築します
* [Running topologies on a production cluster](Running-topologies-on-a-production-cluster.html)
* [Local mode](Local-mode.html): これを読んで、ローカルモードでトポロジを開発してテストする方法を学んでください。

### Streams

ストリームはStormが行っている抽象化の中核にあるものです。ストリームは、分散処理に適するよう並列に処理され生成される、境界付けられていないタプルの列です。ストリームは、ストリームのタプル内のフィールドに名前付けられたスキーマで定義されます。デフォルトでは、タプルにはintegers, longs, shorts, bytes, strings, doubles, floats, booleans, byte arraysが含まれます。カスタム型をタプル内でネイティブに使用できるように、独自のシリアライザを定義することもできます。

ストリームは、宣言の際にIDが与えられます。単一ストリームのSpoutとBoltは非常に一般的なので、[OutputFieldsDeclarer](javadocs/org/apache/storm/topology/OutputFieldsDeclarer.html)には、idを指定せずに単一のストリームを宣言するための便利なメソッドがあります。この場合、ストリームにはデフォルトのID「default」が与えられます。

**Resources:**

* [Tuple](javadocs/org/apache/storm/tuple/Tuple.html): ストリームはタプルで構成されています
* [OutputFieldsDeclarer](javadocs/org/apache/storm/topology/OutputFieldsDeclarer.html): ストリームとそのスキーマの宣言に使用されます
* [Serialization](Serialization.html): Stormのタプルに対する動的型付けならびにカスタムシリアル化の宣言に関する情報

### Spouts

Spoutは、トポロジにおけるストリームの源泉です。一般に、Spoutは外部の資源（例えば、KestrelキューまたはTwitter API）からタプルを読み取り、それらをトポロジに送出します。Spoutは__reliable__か__unreliable__のいずれかです。信頼できるSpoutは、Stormによって処理されなかった場合にはタプルを再実行することができますが、信頼性の低いスパウトはタプルが送出されるとすぐにそれを忘れます。

Spoutは複数のストリームを送出することができます。これを行うには、[OutputFieldsDeclarer](javadocs/org/apache/storm/topology/OutputFieldsDeclarer.html)の `declareStream`メソッドを使用して複数のストリームを宣言し、[SpoutOutputCollector](javadocs/org/apache/storm/spout/SpoutOutputCollector.html)の`emit`メソッドに出力するストリームとして指定します。

Spoutにおける中核的なメソッドは、`nextTuple`です。`nextTuple`は新しいタプルをトポロジーに送出し、送出する新たなタプルがない場合にはreturnします。Stormは同じスレッド上のすべてのSpoutのメソッドを呼び出すため、`nextTuple`はSpoutにおける実装をブロックしないようにしなければなりません。

Spoutにおけるその他の中核的なメソッドは、`ack`と`fail`です。これらは、Spoutが発生したタプルがトポロジで正常に完了したか、完了できなかったことをStormが検出したときに呼び出されます。 `ack`と`fail`は信頼できるSpoutだけにが必要です。詳細は、[the Javadoc](javadocs/org/apache/storm/spout/ISpout.html)を参照してください。

**Resources:**

* [IRichSpout](javadocs/org/apache/storm/topology/IRichSpout.html): Spoutが実装しなければならないインターフェースです。
* [Guaranteeing message processing](Guaranteeing-message-processing.html)

### Bolts

トポロジにおけるすべての処理はBoltで行われます。Boltは、フィルタリング、関数適用、集約、結合、データベースとの会話など、何でもできます。

Boltは簡単なストリーム変換を行うことができます。複雑なストリーム変換を行うには、しばしば複数のステップが必要であり、したがって複数のBoltが必要です。たとえば、つぶやきのストリームを流行の画像のストリームに変換するには、少なくとも2つのステップが必要です: 各画像のリツイートのローリングカウントを行うBoltと、1つ以上のBoltがトップXの画像を出力します（2つのBoltではなく3つのBoltを使用する方法で、よりスケーラブルなストリーム変換を行うことができます）。

Boltは複数のストリームを送出できます。これを行うには、[OutputFieldsDeclarer](javadocs/org/apache/storm/topology/OutputFieldsDeclarer.html)の `declareStream`メソッドを使用して複数のストリームを宣言し、[OutputCollector](javadocs/org/apache/storm/task/OutputCollector.html)の`emit`メソッドに出力するストリームとして指定します。

Boltの入力ストリームを宣言際には、常に別のコンポーネントの特定のストリームをサブスクライブさせます。別のコンポーネントのすべてのストリームをサブスクライブするには、それぞれのストリームを個別にサブスクライブする必要があります。[InputDeclarer](javadocs/org/apache/storm/topology/InputDeclarer.html)には、デフォルトのストリームIDで宣言されたストリームをサブスクライブするための構文糖衣があります。`declarer.shuffleGrouping("1")`では、コンポーネント"1"におけるデフォルトストリームをサブスクライブし、これは`declarer.shuffleGrouping("1", DEFAULT_STREAM_ID)`と同等です。

Boltにおける中核的なメソッドは、入力として新しいタプルを取り込む`execute`メソッドです。 Boltは、[OutputCollector](javadocs/org/apache/storm/task/OutputCollector.html) オブジェクトを使用して新しいタプルを送出します。Boltは、それらが処理する全てのタプルについて`OutputCollector`の`ack`メソッドを呼び出さなければならないので、Stormはタプルが完了したことを知ることができます（そして、いつかはオリジナルのSpoutタプルに対して安全にackすることができます）。入力タプルを処理し、そのタプルに基づいて0個以上のタプルを送出し、入力タプルをackする一般的なケースに対して、Stormは自動的にackを行う[IBasicBolt](javadocs/org/apache/storm/topology/IBasicBolt.html)インターフェイスを提供します。

Boltで新しいスレッドを起動すし、非同期に処理することにまったく問題はありません。[OutputCollector](javadocs/org/apache/storm/task/OutputCollector.html)はスレッドセーフであり、いつでも呼び出すことができます。

**Resources:**

* [IRichBolt](javadocs/org/apache/storm/topology/IRichBolt.html): Boltの一般的なインターフェースです。
* [IBasicBolt](javadocs/org/apache/storm/topology/IBasicBolt.html): フィルタリングや単純な関数を実行するボルトを定義するための便利なインターフェイスです。
* [OutputCollector](javadocs/org/apache/storm/task/OutputCollector.html): ボルトは、このクラスのインスタンスを使用して出力ストリームにタプルを送出します
* [Guaranteeing message processing](Guaranteeing-message-processing.html)

### Stream groupings

トポロジの定義の一部では、各Boltに対して入力として受け取るべきストリームを指定します。ストリームグループ化は、そのストリームをBoltのタスク間でどのように分割すべきかを定義します。

Stormには8つのビルトインストリームグルーピングがあり、[CustomStreamGrouping](javadocs/org/apache/storm/grouping/CustomStreamGrouping.html)インターフェイスを実装することで、カスタムストリームグループを実装できます。

1. **Shuffle grouping**: タプルは、各Boltが同数のタプルを取得することを保証するように、Boltのタスクにランダムに分配されます。
2. **Fields grouping**: グループ化で指定されたフィールドによってストリームが分割されます。たとえば、ストリームが"user-id"フィールドでグループ化されている場合、同じ"user-id"を持つタプルは常に同じタスクにいきますが、異なる"user-id"を持つタプルは異なるタスクにいきます。
3. **Partial Key grouping**: フィールドグループ化のように、グループ化で指定されたフィールドによってストリームが分割されますが、2つの下流のBoltの間で負​​荷分散されます。[この論文](https://melmeric.files.wordpress.com/2014/11/the-power-of-both-choices-practical-load-balancing-for-distributed-stream-processing-engines.pdf)は、どのように機能するかと、それがもたらす利点についての良い説明です。
4. **All grouping**: ストリームはすべてのBoltのタスクにわたって複製されます。このグループは注意して使用してください。
5. **Global grouping**: ストリーム全体がボルトのタスクの1つに移動します。具体的には、IDが最も小さいタスクに移動します。
6. **None grouping**: このグループ化では、ストリームのグループ化に気にしないことを指定します。今のところ、None groupingはShuffle groupingと同等です。結局のところ、StormはNone groupingなBoltをプッシュダウンし、（可能であれば）それらがサブスクライブしているBoltまたはSpoutと同じスレッドで実行します。
7. **Direct grouping**: これは特別な種類のグループ化です。このようにグループ化されたストリームは、タプルの__プロデューサ__がこのタプルを受け取るコンシューマのタスクを決定します。Direct groupingは、ダイレクトストリームとして宣言されたストリームでのみ宣言できます。ダイレクトストリームに出力されるタプルは、[emitDirect](javadocs/org/apache/storm/task/OutputCollector.html#emitDirect(int, int, java.util.List)メソッドの1つを使用して出力する必要があります。提供された[TopologyContext](javadocs/org/apache/storm/task/TopologyContext.html)を使用するか、タプルが送信されたタスクIDを返す[OutputCollector](javadocs/org/apache/storm/task/OutputCollector.html)の`emit`メソッドの出力を追跡することによって、BoltはコンシューマのタスクIDを取得できます。
8. **Local or shuffle grouping**: 対象とするBoltの同じワーカープロセスで1つ以上のタスクがある場合、タプルはそれらのうちの処理中のタスクだけにシャッフルされます。それ以外の場合、これは通常のShuffle groupingのように動作します。

**Resources:**

* [TopologyBuilder](javadocs/org/apache/storm/topology/TopologyBuilder.html): このクラスを使用してトポロジを定義します
* [InputDeclarer](javadocs/org/apache/storm/topology/InputDeclarer.html): このオブジェクトは`TopologyBuilder`で`setBolt`が呼び出されたときに返され、Boltの入力ストリームと、ストリームをグループ化する方法を宣言します

### Reliability

StormはすべてのSpoutタプルがトポロジによって全て処理されることを保証します。これは、すべてのSpoutタプルによってトリガーされたタプルのツリーを追跡し、タプルのツリーがいつ正常に完了したかを判断することによって行います。すべてのトポロジには、"message timeout"が関連付けられています。StormはSpoutタプルがそのタイムアウト時間内に完了したことを検出できない場合、失敗したものとして後でそれを再送します。

Stormの信頼性機能を利用するには、タプルツリーにおいて新しい葉が生成されていることをStormに伝え、個々のタプルの処理が終わるたびにStormに伝える必要があります。これらは、タプルを送出するために使用される[OutputCollector](javadocs/org/apache/storm/task/OutputCollector.html)オブジェクトを使用して行われます。Anchoringは`emit`メソッドで行い、`ack`メソッドを使ってタプルを処理し終えたことを宣言します。

この機能に関する詳細な解説は、[Guaranteeing message processing](Guaranteeing-message-processing.html).を参照してください。

### Tasks

各SpoutまたはBoltは、クラスタ全体で多くのタスクを実行します。 各タスクは1つの実行スレッドに対応し、ストリームグループはタプルをあるタスクセットから別のタスクセットに送信する方法を定義します。[TopologyBuilder](javadocs/org/apache/storm/topology/TopologyBuilder.html)の`setSpout`メソッドと`setBolt`メソッドで、各SpoutやBoltのparallelismを設定します。

### Workers

トポロジは、1つまたはそれ以上のワーカープロセスで実行されます。各ワーカープロセスは物理的にはJVMであり、トポロジのすべてのタスクのサブセットを実行します。たとえば、トポロジにおける合算したparallelismが300で、50のワーカーが割り当てられている場合、各ワーカーは6つのタスクをワーカー内のスレッドとして実行します。Stormはすべてのワーカーににタスクを均等に配分しようとします。

**Resources:**

* [Config.TOPOLOGY_WORKERS](javadocs/org/apache/storm/Config.html#TOPOLOGY_WORKERS): この設定では、トポロジを実行するために割り当てるワーカーの数をセットします

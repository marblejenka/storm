---
title: Storm Kafka Integration
layout: documentation
documentation: true
---

Apache Kafka 0.8.xからデータをコンシュームするためのStormコアとTridentのSpout実装を提供します。

##Spouts
TridentとStormコアのSpoutの両方をサポートしています。両方のSpoutの実装では、Kafkaブローカのホストからパーティションへの対応付けを追跡するBrokerHostインターフェイスと、Kafka関連のパラメータを制御するkafkaConfigを使用しています。
 
###BrokerHosts
Kafka spout/emitterを初期化するには、マーカーインタフェースBrokerHostsを実装するインスタンスを生成する必要があります。
現在、次の2つの実装をサポートしています:

####ZkHosts
Kafkaブローカーからパーティションへの対応付けを動的に追跡する場合は、ZkHostsを使用する必要があります。
このクラスは、KafkaのZooKeeperエントリを使用してブローカーのホスト->パーティションの対応付けを追跡します。以下のメソッドを呼び出してオブジェクトをインスタンス化することができます

```java
    public ZkHosts(String brokerZkStr, String brokerZkPath) 
    public ZkHosts(String brokerZkStr)
```
上記においてbrokerZkStrはip:portです(例 localhost:2181)。brokerZkPathは、すべてのトピックとパーティション情報が格納されるルートディレクトリです。
デフォルトでは、Kafkaのデフォルト実装が使用する /brokers です。

デフォルトでは、ブローカとパーティションのマッピングはZooKeeperから60秒ごとに更新されます。
これを変更する場合は、host.refreshFreqSecsを選択した値に設定する必要があります。

####StaticHosts
これは、broker->パーティションが静的である代替の実装です。
このクラスのインスタンスを構築するには、まずGlobalPartitionInformationのインスタンスを構築する必要があります。

```java
    Broker brokerForPartition0 = new Broker("localhost");//localhost:9092
    Broker brokerForPartition1 = new Broker("localhost", 9092);//localhost:9092 but we specified the port explicitly
    Broker brokerForPartition2 = new Broker("localhost:9092");//localhost:9092 specified as one string.
    GlobalPartitionInformation partitionInfo = new GlobalPartitionInformation();
    partitionInfo.addPartition(0, brokerForPartition0);//mapping from partition 0 to brokerForPartition0
    partitionInfo.addPartition(1, brokerForPartition1);//mapping from partition 1 to brokerForPartition1
    partitionInfo.addPartition(2, brokerForPartition2);//mapping from partition 2 to brokerForPartition2
    StaticHosts hosts = new StaticHosts(partitionInfo);
```

###KafkaConfig
kafkaSpoutを構築するために必要なもう一つは、KafkaConfigのインスタンスです。

```java
    public KafkaConfig(BrokerHosts hosts, String topic)
    public KafkaConfig(BrokerHosts hosts, String topic, String clientId)
```

BrokerHostsは、上記のようにBrokerHostsインターフェイスの実装にすることができます。topicはKafkaにおけるトピック名です。
オプションのClientIdは、Spoutの現在のコンシュームしたオフセットが格納されているZooKeeperパスの一部として使用されます。

現在使用されているKafkaConfigには2つの拡張機能があります。

Spoutconfigは、KafkaConfigの拡張で、ZooKeeper接続情報を持つ追加のフィールドをサポートし、KafkaSpout固有の動作を制御します。
Zkrootは、コンシューマのオフセットを格納するためのルートとして使用されます。idは、あなたのSpoutを一意に識別する必要があります。

```java
public SpoutConfig(BrokerHosts hosts, String topic, String zkRoot, String id);
public SpoutConfig(BrokerHosts hosts, String topic, String id);
```
これらのパラメータに加えて、SpoutConfigには、KafkaSpoutの動作を制御する次のフィールドが含まれています:

```java
    // setting for how often to save the current Kafka offset to ZooKeeper
    public long stateUpdateIntervalMs = 2000;

    // Exponential back-off retry settings.  These are used when retrying messages after a bolt
    // calls OutputCollector.fail().
    // Note: be sure to set org.apache.storm.Config.MESSAGE_TIMEOUT_SECS appropriately to prevent
    // resubmitting the message while still retrying.
    public long retryInitialDelayMs = 0;
    public double retryDelayMultiplier = 1.0;
    public long retryDelayMaxMs = 60 * 1000;

    // if set to true, spout will set Kafka topic as the emitted Stream ID
    public boolean topicAsStreamId = false;
```
コアであるKafkaSpoutはSpoutConfigのインスタンスのみを受け入れます。

TridentKafkaConfigはKafkaConfigのもう一つの拡張です。
TridentKafkaEmitterはTridentKafkaConfigのみを受け入れます。

KafkaConfigクラスには、アプリケーションの動作を制御するパブリック変数があります。ここにデフォルトがあります:

```java
    public int fetchSizeBytes = 1024 * 1024;
    public int socketTimeoutMs = 10000;
    public int fetchMaxWait = 10000;
    public int bufferSizeBytes = 1024 * 1024;
    public MultiScheme scheme = new RawMultiScheme();
    public boolean ignoreZkOffsets = false;
    public long startOffsetTime = kafka.api.OffsetRequest.EarliestTime();
    public long maxOffsetBehind = Long.MAX_VALUE;
    public boolean useStartOffsetTimeIfOffsetOutOfRange = true;
    public int metricsTimeBucketSizeInSecs = 60;
```

それらのほとんどは、MultiSchemeを除いて自明であると思います。
###MultiScheme
MultiSchemeは、Kafkaからコンシュームされたバイト配列がどのようにStormのタプルに変換されるかを指示するインタフェースです。
また、出力フィールドの命名も制御します。

```java
  public Iterable<List<Object>> deserialize(byte[] ser);
  public Fields getOutputFields();
```

デフォルトの`RawMultiScheme`は単に`byte[]`をとり、`byte[]`をそのままタプルとして返します。 outputFieldの名前は"bytes"です。
`SchemeAsMultiScheme`や`KeyValueSchemeAsMultiScheme`のような別の実装があります。これは`byte[]`を`String`に変換することができます。


### Examples

#### Core Spout

```java
BrokerHosts hosts = new ZkHosts(zkConnString);
SpoutConfig spoutConfig = new SpoutConfig(hosts, topicName, "/" + topicName, UUID.randomUUID().toString());
spoutConfig.scheme = new SchemeAsMultiScheme(new StringScheme());
KafkaSpout kafkaSpout = new KafkaSpout(spoutConfig);
```

#### Trident Spout
```java
TridentTopology topology = new TridentTopology();
BrokerHosts zk = new ZkHosts("localhost");
TridentKafkaConfig spoutConf = new TridentKafkaConfig(zk, "test-topic");
spoutConf.scheme = new SchemeAsMultiScheme(new StringScheme());
OpaqueTridentKafkaSpout spout = new OpaqueTridentKafkaSpout(spoutConf);
```


### How KafkaSpout stores offsets of a Kafka topic and recovers in case of failures

上記のKafkaConfigプロパティに示されているように、KafkaトピックのどこからSpoutが開始するかを制御するには、
`KafkaConfig.startOffsetTime`を以下のように設定します:

1. `kafka.api.OffsetRequest.EarliestTime()`: トピックの先頭から（つまり、最も古いメッセージから）読み込みます。
2. `kafka.api.OffsetRequest.LatestTime()`: トピックの最後から読み込みます（つまり、トピックに書き込まれているなんらかの新しいメッセージ）
3. エポックを基準としているUnixタイムスタンプ(例えば、`System.currentTimeMillis()`を介して):
   KafkaのFAQにある[OffsetRequestを使用して特定のタイムスタンプのメッセージの正確なオフセットを取得するにはどうすればよいですか?](https://cwiki.apache.org/confluence/display/KAFKA/FAQ#FAQ-HowdoIaccuratelygetoffsetsofmessagesforacertaintimestampusingOffsetRequest?)を参照してください。

トポロジが実行されると、KafkaSpoutは、状態情報をZooKeeperパスの`SpoutConfig.zkRoot+ "/" + SpoutConfig.id`配下に保存することによって、読み取りや送出したオフセットを追跡します。障害が発生した場合は、ZooKeeperで最後に書き込まれたオフセットから回復します。

> **Important:**  トポロジを再配備するときは、`SpoutConfig.zkRoot`と`SpoutConfig.id`の設定が変更されていないことを確認してください。
> そうでなければ、Spoutは以前のコンシューマ状態情報(オフセットなど)をZooKeeperから読み込むことができません。
> -- これは、ユースケースによりますが、予期しない動作やデータの消失につながる可能性があります。

これは、トポロジがいったん実行されれば、トポロジはZooKeeperのコンシューマ状態情報(オフセット)に依存して読み込みを開始 (より正確にはレジューム)する必要がある場所を決定するため、`KafkaConfig.startOffsetTime`の設定はトポロジーのその後の実行に影響しないことを意味します。
SpoutがZooKeeperに保存されているどのコンシューマの状態情報をも無視するよう強制したい場合は、パラメータ`KafkaConfig.ignoreZkOffsets`を`true`に設定する必要があります。
`true`の場合、Spoutは常に前述のように`KafkaConfig.startOffsetTime`で定義されたオフセットからの読み込みを開始します。


## Using storm-kafka with different versions of Scala

Storm-kafkaのKafkaに対する依存関係は、Mavenの`provided`スコープとして定義されています。つまり、推移的依存関係として引き込まれません。
これにより、特定のScalaバージョンに対してビルドされたKafkaのバージョンを使用することができます。

storm-kafkaでプロジェクトをビルドするときは、Kafkaの依存関係を明示的に追加する必要があります。
たとえば、Scala 2.10に対してビルドされたKafka 0.8.1.1を使用するには、`pom.xml`に次の依存関係を使用します:

```xml
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka_2.10</artifactId>
            <version>0.8.1.1</version>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.zookeeper</groupId>
                    <artifactId>zookeeper</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>log4j</groupId>
                    <artifactId>log4j</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
```

ZooKeeperとlog4jに対する依存関係は、Stormの依存関係とのバージョンの競合を防ぐためにexcludeされています。

##Writing to Kafka as part of your topology
org.apache.storm.kafka.bolt.KafkaBoltのインスタンスを生成し、コンポーネントとしてトポロジにアタッチすることもできます。
トライデントを使用している場合は、org.apache.storm.kafka.trident.TridentState, org.apache.storm.kafka.trident.TridentStateFactoryおよびorg.apache.storm.kafka.trident.TridentKafkaUpdaterを使ってください。

次の2つのインターフェイスの実装を提供する必要があります

###TupleToKafkaMapper and TridentTupleToKafkaMapper
これらのインタフェースには2つのメソッドが定義されています:

```java
    K getKeyFromTuple(Tuple/TridentTuple tuple);
    V getMessageFromTuple(Tuple/TridentTuple tuple);
```

名前が示すように、これらのメソッドは、タプルをKafkaのkeyとmessageに対応付けするために呼び出されます。
単に1つのフィールドをキーとして、1つのフィールドを値として使用する場合は、提供されているFieldNameBasedTupleToKafkaMapper.javaの実装を使用できます。
KafkaBoltでは、後方互換性の理由から、デフォルトのコンストラクタを使用してFieldNameBasedTupleToKafkaMapperを構築する場合、実装は常にフィールド名が"key"と"message"であるフィールドを探します。
また、デフォルト以外のコンストラクタを使用して、異なるキーとメッセージフィールドを指定することもできます
TridentKafkaStateでは、デフォルトコンストラクタがないため、keyとmessageのフィールド名を指定する必要があります。
これらは、FieldNameBasedTupleToKafkaMapperの生成時に指定する必要があります。

###KafkaTopicSelector and trident KafkaTopicSelector
このインタフェースには1つのメソッドしかありません

```java
public interface KafkaTopicSelector {
    String getTopics(Tuple/TridentTuple tuple);
}
```
このインタフェースの実装は、パブリッシュされるトピックとタプルのkey/messageの対応付けを返す必要があります
nullを返すことができ、メッセージは無視されます。1つの静的トピック名がある場合、DefaultTopicSelector.javaを使用して、コンストラクタ内でトピックの名前を設定できます。

### Specifying Kafka producer properties
Stormトポロジの設定で、キーのkafka.broker.propertiesを使用してプロパティのマップを設定することによって、すべてのプロダクションプロパティを提供できます(「プロデューサの重要なコンフィグレーションプロパティ」 http://kafka.apache.org/documentation.html#producerconfigs を参照)。

###Putting it all together

Boltについて:

```java
        TopologyBuilder builder = new TopologyBuilder();
    
        Fields fields = new Fields("key", "message");
        FixedBatchSpout spout = new FixedBatchSpout(fields, 4,
                    new Values("storm", "1"),
                    new Values("trident", "1"),
                    new Values("needs", "1"),
                    new Values("javadoc", "1")
        );
        spout.setCycle(true);
        builder.setSpout("spout", spout, 5);
        KafkaBolt bolt = new KafkaBolt()
                .withTopicSelector(new DefaultTopicSelector("test"))
                .withTupleToKafkaMapper(new FieldNameBasedTupleToKafkaMapper());
        builder.setBolt("forwardToKafka", bolt, 8).shuffleGrouping("spout");
        
        Config conf = new Config();
        //set producer properties.
        Properties props = new Properties();
        props.put("metadata.broker.list", "localhost:9092");
        props.put("request.required.acks", "1");
        props.put("serializer.class", "kafka.serializer.StringEncoder");
        conf.put(KafkaBolt.KAFKA_BROKER_PROPERTIES, props);
        
        StormSubmitter.submitTopology("kafkaboltTest", conf, builder.createTopology());
```

Tridentでは:

```java
        Fields fields = new Fields("word", "count");
        FixedBatchSpout spout = new FixedBatchSpout(fields, 4,
                new Values("storm", "1"),
                new Values("trident", "1"),
                new Values("needs", "1"),
                new Values("javadoc", "1")
        );
        spout.setCycle(true);

        TridentTopology topology = new TridentTopology();
        Stream stream = topology.newStream("spout1", spout);

        TridentKafkaStateFactory stateFactory = new TridentKafkaStateFactory()
                .withKafkaTopicSelector(new DefaultTopicSelector("test"))
                .withTridentTupleToKafkaMapper(new FieldNameBasedTupleToKafkaMapper("word", "count"));
        stream.partitionPersist(stateFactory, fields, new TridentKafkaUpdater(), new Fields());

        Config conf = new Config();
        //set producer properties.
        Properties props = new Properties();
        props.put("metadata.broker.list", "localhost:9092");
        props.put("request.required.acks", "1");
        props.put("serializer.class", "kafka.serializer.StringEncoder");
        conf.put(TridentKafkaState.KAFKA_BROKER_PROPERTIES, props);
        StormSubmitter.submitTopology("kafkaTridentTest", conf, topology.build());
```

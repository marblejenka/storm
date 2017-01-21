---
title: Storm State Management
layout: documentation
documentation: true
---
# State support in core storm
Storm coreには、操作の状態を保存して取得するためのBoltの抽象があります。
デフォルトのインメモリベースの状態実装と、状態の永続性を提供するRedisバックアップ実装があります。

## State management
フレームワークによってその状態を管理・永続化する必要のあるBoltは、
`IStatefulBolt`インタフェースを実装するか、`BaseStatefulBolt`を継承し、`void initState(T state)`メソッドを実装する必要があります。
`initState`メソッドは、以前に保存されたBoltの状態にするため、Boltの初期化中にフレームワークによって呼び出されます。
これは、prepareが完了した後で、Boltがタプルの処理を開始する前に、呼び出されます。

現在サポートされている`State`実装の唯一の種類は、Key-Valueマッピングを提供する`KeyValueState`です

例えば、ワードカウントを行うBoltは、以下のように、単語のカウントに対してキー値の状態抽象化を使用することができます。

1. BaseStatefulBoltを拡張し、型パラメーターとして単語とカウントのマッピングを格納するKeyValueStateを指定します。
2. Boltはinitメソッドで以前に保存された状態で初期化されます。
これには前回実行時にフレームワークによって最後にコミットされた単語数が含まれます。
3. executeメソッドで単語数を更新します。

 ```java
 public class WordCountBolt extends BaseStatefulBolt<KeyValueState<String, Long>> {
 private KeyValueState<String, Long> wordCounts;
 private OutputCollector collector;
 ...
     @Override
     public void prepare(Map stormConf, TopologyContext context, OutputCollector collector) {
       this.collector = collector;
     }
     @Override
     public void initState(KeyValueState<String, Long> state) {
       wordCounts = state;
     }
     @Override
     public void execute(Tuple tuple) {
       String word = tuple.getString(0);
       Integer count = wordCounts.get(word, 0);
       count++;
       wordCounts.put(word, count);
       collector.emit(tuple, new Values(word, count));
       collector.ack(tuple);
     }
 ...
 }
 ```
4. フレームワークは、Boltの状態を定期的にチェックポイントします（デフォルトは毎秒）。Stormの設定 `topology.state.checkpoint.interval.ms`を設定することで頻度を変更することができます
5. 状態の永続性を維持するには、Stormの設定で`topology.state.provider`を設定して永続性をサポートする状態プロバイダを使用します。
例えば、Redisベースのキーとバリューについての状態の実装を使用するには、storm.yamlで`topology.state.provider：org.apache.storm.redis.state.RedisKeyValueStateProvider`を設定します。
プロバイダ実装のjarはクラスパスになければなりません。この場合、`storm-redis-*.jar`をextlibディレクトリに置くことを意味します。
6. 状態プロバイダのプロパティは、`topology.state.provider.config`を設定することで上書きできます。Redisのステートでは、これは以下のプロパティを持つjson設定です。

 ```
 {
   "keyClass": "Optional fully qualified class name of the Key type.",
   "valueClass": "Optional fully qualified class name of the Value type.",
   "keySerializerClass": "Optional Key serializer implementation class.",
   "valueSerializerClass": "Optional Value Serializer implementation class.",
   "jedisPoolConfig": {
     "host": "localhost",
     "port": 6379,
     "timeout": 2000,
     "database": 0,
     "password": "xyz"
     }
 }
 ```

## Checkpoint mechanism
チェックポイントは、指定された`topology.state.checkpoint.interval.ms`で、内部チェックポイントSpoutによってトリガされます。
トポロジ内に少なくとも1つの`IStatefulBolt`がある場合、チェックポイントSpoutはトポロジビルダーによって自動的に追加されます。
ステートフルトポロジーの場合、トポロジービルダーは、チェックポイントタプルの受信時に状態をコミットする`StatefulBoltExecutor`に`IStatefulBolt`をラップします。
非ステートフルボルトは、チェックポイントタプルがトポロジのDAGを流れることができるように、チェックポイントタプルを単に転送する`CheckpointTupleForwarder`でラップされます。
チェックポイントタプルは別の内部ストリーム、すなわち`$checkpoint`を通って流れます。トポロジ・ビルダーは、チェックポイント・ストリームをトポロジ全体に結線し、ルートにチェックポイント・スパウトを配置します。

```
              default                         default               default
[spout1]   ---------------> [statefulbolt1] ----------> [bolt1] --------------> [statefulbolt2]
                          |                 ---------->         -------------->
                          |                   ($chpt)               ($chpt)
                          |
[$checkpointspout] _______| ($chpt)
```

チェックポイント間隔で、チェックポイントタプルはチェックポイントSpoutによって放出されます。チェックポイントタプルを受け取ると、Boltの状態が保存され、チェックポイントタプルが次のコンポーネントに転送されます。
各Boltは、状態が保存される前にすべての入力ストリームにチェックポイントが到着するのを待って、状態がトポロジ全体で一貫した状態を表すようにしています。
チェックポイントSpoutがすべてのBoltからackを受信すると、状態のコミットは完了し、トランザクションはチェックポイントSpoutによってコミットされたものとして記録されます。

状態チェックポイントは現在、Spoutの状態をチェックポイントしていません。しかし、いったんすべてのBoltの状態がチェックポイントされ、チェックポイントタプルがackされると、Spoutからemitされたタプルもackされます。また、`topology.state.checkpoint.interval.ms`は`topology.message.timeout.secs`よりも低いことを想定します。

状態のコミットは、トポロジ全体の状態が一貫したアトミックな方法で保存されるように、
prepareフェーズとcommitフェーズを用いた3フェーズコミットプロトコルのように機能します。

### Recovery
リカバリフェーズは、トポロジが初めて開始されたときにトリガされます。前のトランザクションが正常にprepareされなかった場合、
トポロジ全体に`rollback`メッセージが送信され、Boltにprepareなトランザクションがあれば破棄することができます。
前のトランザクションが正常にprepareされたがコミットされていない場合、prepareされたトランザクションをコミットできるように、
トポロジー全体に渡って`commit`メッセージが送信されます。これらのステップが完了すると、Boltはステートによって初期化されます。

Boltの1つがチェックポイントメッセージを確認できなかったり、ワーカーが途中でクラッシュしたりすると、リカバリがトリガーされます。
したがって、スーパバイザによってワーカーが再起動されると、チェックポイント機構はBoltを以前の状態で初期化し、
以前のチェックポイントの時点から再開されるようにします。

### Guarantee
Stormは、障害時にタプルをリプレイするackingメカニズムに依存しています 状態はコミットされている可能性がありますが、タプルをackする前にワーカーがクラッシュしてしまう可能性があります。
この場合、タプルがリプレイされ、重複した状態の更新が発生します。
現在、StatefulBoltExecutorは、状態を保存するためにチェックポイントが他の入力ストリームに到着するのを待っている間に、
ストリームからチェックポイントタプルを受け取った後、タプルをストリームから処理し続けます。
これにより、リカバリ中に状態の更新が重複する可能性もあります。

状態の抽象化は重複した評価を排除するものではなく、現在ではat-least onceの保証しか提供していません。

ステートフルトポロジ内のすべてのBoltは、at-least onceの保証を提供するために、タプルをanchorするとともに入力タプルを処理した後にそのタプルをackすることが期待されます。非ステートフルボルトの場合、anchoring/ackingは`BaseBasicBolt`を拡張することによって自動的に管理できます。ステートフルボルトは、上記の状態管理セクションの`WordCountBolt`の例のように処理した後にタプルをanchorし、タプルをackすることが期待されます。

### IStateful bolt hooks
IStateful Boltインターフェイスは、ステートフルボルトでカスタムアクションを実装できるフックメソッドを提供します。

```java
    /**
     * This is a hook for the component to perform some actions just before the
     * framework commits its state.
     */
    void preCommit(long txid);

    /**
     * This is a hook for the component to perform some actions just before the
     * framework prepares its state.
     */
    void prePrepare(long txid);

    /**
     * This is a hook for the component to perform some actions just before the
     * framework rolls back the prepared state.
     */
    void preRollback();
```
これはオプションであり、ステートフルボルトは実装を提供するものではありません。
これは、ステートフルなBoltの状態がprepareされ、コミットまたはロールバックされる前にいくつかのアクションをとることが必要であるような、
ステートフルな抽象化の上に他のシステムレベルのコンポーネントを構築できるように提供されています。

## Providing custom state implementations
現在サポートされている唯一の`State`実装は、Key-Valueマッピングを提供する`KeyValueState`です。

カスタムの状態の実装は`org.apache.storm.State`インタフェースで定義されたメソッドの実装を提供する必要があります。
それは`void prepareCommit(long txid)`, `void commit(long txid)`, `rollback()`メソッドです。`commit()`メソッドはオプションで、Boltが単独で状態を管理する場合に便利です。
これは、現在、例えば内部システムボルトによってのみ使用されています。例えば、CheckpointSpoutは状態を保存します。

`KeyValueState`の実装は`org.apache.storm.state.KeyValueState`インタフェースで定義されたメソッドも実装する必要があります。

### State provider
フレームワークは、対応する`StateProvider`実装を介して状態をインスタンス化します。カスタムの状態は、名前空間に基づいて状態をロードして返すことができる `StateProvider`実装も提供する必要があります。各状態は一意の名前空間に属します。
名前空間は、通常、タスクごとに一意であるため、各タスクは独自の状態を持つことができます。StateProviderと対応するStateの実装は、Stormのクラスパスで利用できるようにする必要があります（extlibディレクトリに配置するなどして）。

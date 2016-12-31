---
title: Troubleshooting
layout: documentation
documentation: true
---

このページには、Stormを使用する際に人々が遭遇した問題ならびにその解決方法が記載されています。

### Worker processes are crashing on startup with no stack trace

考えられる症状:

 * トポロジは1つのノードで動作しますが、複数のノードではクラッシュする

解決方法:

 * サブネットが誤って設定されている可能性があります。サブネットでは、ホスト名に基づいてノードを見つけることができません。 ZeroMQは、ホストを解決できない場合にプロセスをクラッシュさせることがあります。2つのソリューションがあります。
  * /etc/hostsでホスト名からIPアドレスへのマッピングを行う
  * ノードがホスト名に基づいてお互いを見つけることができるように内部DNSを設定する。
  
### Nodes are unable to communicate with each other

考えられる症状:

 * すべてのスパウトタプルが失敗しています
 * 処理が動いていない

解決方法:

 * ストームはipv6では動作しません ipv4を強制するには、スーパーバイザのchild optionに`-Djava.net.preferIPv4Stack=true`を追加し、スーパーバイザを再起動します。
 * サブネットの設定が間違っている可能性があります。`Worker processes are crashing on startup with no stack trace`の解決策を参照してください。

### Topology stops processing tuples after awhile

症状:

 * 処理はしばらくの間うまくいき、その後突然停止しSpoutタプルが大量に失敗する。
 
解決方法:

 * これはZeroMQ 2.1.10の既知の問題です。ZeroMQ 2.1.7にダウングレードしてください。
 
### Not all supervisors appear in Storm UI

症状:
 
 * いくつかのスーパバイザプロセスがStorm UIで見つからない
 * Storm UIをリフレッシュするとスーパバイザが違うものになる

解決方法:

 * スーパーバイザのローカルディレクトリが独立していることを確認する(NFS上でローカルディレクトリを共有していないかなど)
 * スーパーバイザのローカルディレクトリを削除し、デーモンを再起動してください。スーパーバイザは、ユニークなIDを作成してローカルに格納します。そのIDが他のノードにコピーされると、Stormは混乱します。

### "Multiple defaults.yaml found" error

症状:

 * "storm jar"でトポロジをデプロイすると、このエラーが発生する

解決方法:

 * トポロジのjarの中にStormのjarを入れている可能性が最も高いです。あなたのトポロジをjarにパッケージするときは、Stormのjarを含めないでください。Stormのjarはクラスパスに入るはずです。

### "NoSuchMethodError" when running storm jar

症状:

 * Stormのjarを実行する際に、謎の"NoSuchMethodError"が発生する

解決方法:

 * あなたはあなたのトポロジをビルドしたのとは異なるバージョンのStormに、あなたのトポロジをデプロイしています。使用するStormクライアントが、トポロジをコンパイルしたバージョンと同じバージョンであることを確認してください。


### Kryo ConcurrentModificationException

症状:

 * 実行時に、次のようなスタックトレースが出てくる:

```
java.lang.RuntimeException: java.util.ConcurrentModificationException
	at org.apache.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:84)
	at org.apache.storm.utils.DisruptorQueue.consumeBatchWhenAvailable(DisruptorQueue.java:55)
	at org.apache.storm.disruptor$consume_batch_when_available.invoke(disruptor.clj:56)
	at org.apache.storm.disruptor$consume_loop_STAR_$fn__1597.invoke(disruptor.clj:67)
	at org.apache.storm.util$async_loop$fn__465.invoke(util.clj:377)
	at clojure.lang.AFn.run(AFn.java:24)
	at java.lang.Thread.run(Thread.java:679)
Caused by: java.util.ConcurrentModificationException
	at java.util.LinkedHashMap$LinkedHashIterator.nextEntry(LinkedHashMap.java:390)
	at java.util.LinkedHashMap$EntryIterator.next(LinkedHashMap.java:409)
	at java.util.LinkedHashMap$EntryIterator.next(LinkedHashMap.java:408)
	at java.util.HashMap.writeObject(HashMap.java:1016)
	at sun.reflect.GeneratedMethodAccessor17.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:616)
	at java.io.ObjectStreamClass.invokeWriteObject(ObjectStreamClass.java:959)
	at java.io.ObjectOutputStream.writeSerialData(ObjectOutputStream.java:1480)
	at java.io.ObjectOutputStream.writeOrdinaryObject(ObjectOutputStream.java:1416)
	at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1174)
	at java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:346)
	at org.apache.storm.serialization.SerializableSerializer.write(SerializableSerializer.java:21)
	at com.esotericsoftware.kryo.Kryo.writeClassAndObject(Kryo.java:554)
	at com.esotericsoftware.kryo.serializers.CollectionSerializer.write(CollectionSerializer.java:77)
	at com.esotericsoftware.kryo.serializers.CollectionSerializer.write(CollectionSerializer.java:18)
	at com.esotericsoftware.kryo.Kryo.writeObject(Kryo.java:472)
	at org.apache.storm.serialization.KryoValuesSerializer.serializeInto(KryoValuesSerializer.java:27)
```

解決方法: 

 * これは出力タプルとして可変オブジェクトを出力していることを意味します。出力コレクターにemitするすべてのものはimmutableでなければなりません。何が起こっているのは、あなたのBoltが、ネットワーク上で送信されるようにシリアル化されている間にオブジェクトを修正しているということです。


### NullPointerException from deep inside Storm

症状:

 * 次のようなNullPointerExceptionが出てくる:

```
java.lang.RuntimeException: java.lang.NullPointerException
    at org.apache.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:84)
    at org.apache.storm.utils.DisruptorQueue.consumeBatchWhenAvailable(DisruptorQueue.java:55)
    at org.apache.storm.disruptor$consume_batch_when_available.invoke(disruptor.clj:56)
    at org.apache.storm.disruptor$consume_loop_STAR_$fn__1596.invoke(disruptor.clj:67)
    at org.apache.storm.util$async_loop$fn__465.invoke(util.clj:377)
    at clojure.lang.AFn.run(AFn.java:24)
    at java.lang.Thread.run(Thread.java:662)
Caused by: java.lang.NullPointerException
    at org.apache.storm.serialization.KryoTupleSerializer.serialize(KryoTupleSerializer.java:24)
    at org.apache.storm.daemon.worker$mk_transfer_fn$fn__4126$fn__4130.invoke(worker.clj:99)
    at org.apache.storm.util$fast_list_map.invoke(util.clj:771)
    at org.apache.storm.daemon.worker$mk_transfer_fn$fn__4126.invoke(worker.clj:99)
    at org.apache.storm.daemon.executor$start_batch_transfer__GT_worker_handler_BANG_$fn__3904.invoke(executor.clj:205)
    at org.apache.storm.disruptor$clojure_handler$reify__1584.onEvent(disruptor.clj:43)
    at org.apache.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:81)
    ... 6 more
```

または、 

```
java.lang.RuntimeException: java.lang.NullPointerException
        at
org.apache.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:128)
~[storm-core-0.9.3.jar:0.9.3]
        at
org.apache.storm.utils.DisruptorQueue.consumeBatchWhenAvailable(DisruptorQueue.java:99)
~[storm-core-0.9.3.jar:0.9.3]
        at
org.apache.storm.disruptor$consume_batch_when_available.invoke(disruptor.clj:80)
~[storm-core-0.9.3.jar:0.9.3]
        at
org.apache.storm.disruptor$consume_loop_STAR_$fn__759.invoke(disruptor.clj:94)
~[storm-core-0.9.3.jar:0.9.3]
        at org.apache.storm.util$async_loop$fn__458.invoke(util.clj:463)
~[storm-core-0.9.3.jar:0.9.3]
        at clojure.lang.AFn.run(AFn.java:24) [clojure-1.5.1.jar:na]
        at java.lang.Thread.run(Thread.java:745) [na:1.7.0_65]
Caused by: java.lang.NullPointerException: null
        at clojure.lang.RT.intCast(RT.java:1087) ~[clojure-1.5.1.jar:na]
        at
org.apache.storm.daemon.worker$mk_transfer_fn$fn__3548.invoke(worker.clj:129)
~[storm-core-0.9.3.jar:0.9.3]
        at
org.apache.storm.daemon.executor$start_batch_transfer__GT_worker_handler_BANG_$fn__3282.invoke(executor.clj:258)
~[storm-core-0.9.3.jar:0.9.3]
        at
org.apache.storm.disruptor$clojure_handler$reify__746.onEvent(disruptor.clj:58)
~[storm-core-0.9.3.jar:0.9.3]
        at
org.apache.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:125)
~[storm-core-0.9.3.jar:0.9.3]
        ... 6 common frames omitted
```

解決方法:

 * これは、複数のスレッドが`OutputCollector`に何かしらのメソッドを発行することによって発生します。すべてのemit、ack、およびfailは、同じスレッドで発生する必要があります。これが起こる可能性のある微妙な方法の1つは、別のスレッドでemitを行う`IBasicBolt`を作成する場合です。`IBasicBolt`では、executeの後に自動的にackが呼び出されるので、複数のスレッドで`OutputCollector`を使用するとこの例外を起こしえます。Basic Boltを使用する場合、すべてのemitは`execute`を実行するのと同じスレッドで行わなければなりません。

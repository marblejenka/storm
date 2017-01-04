---
title: Windowing Support in Core Storm
layout: documentation
documentation: true
---

Stormコアは、ウィンドウ内にあるタプルのグループを処理するためのサポートを備えています。
Windowsは、次の2つのパラメータで指定されます。

1. Window length - ウィンドウの長さまたは持続時間
2. Sliding interval - ウィンドウがスライドする間隔

## Sliding Window

タプルはウィンドウ内にグループ分けされ、ウィンドウはスライドのインターバルごとにスライドします。タプルはひとつ以上のウィンドウに属することができます。

長さが10秒でスライディングインターバルが5秒である持続時間ベースのスライドウィンドウの例です。

```
| e1 e2 | e3 e4 e5 e6 | e7 e8 e9 |...
0       5             10         15    -> time

|<------- w1 -------->|
        |------------ w2 ------->|
```

ウィンドウは5秒ごとに評価され、第1のウィンドウ内のタプルのいくつかは第2のウィンドウと重複します。

## Tumbling Window

タプルは、時間またはカウントに基づいて1つのウィンドウにグループ化されます。どのタプルも1つのウィンドウにしか属しません。

長さが5秒の持続時間に基づくタンブリングウィンドウの例です。

```
| e1 e2 | e3 e4 e5 e6 | e7 e8 e9 |...
0       5             10         15    -> time
   w1         w2            w3
```

ウィンドウは5秒ごとに評価され、ウィンドウが重なることはありません。

Stormは、タプル数のカウントまたは持続時間で、ウィンドウの長さとスライド間隔を指定することをサポートしています。

Boltインタフェース`IWindowedBolt`は、ウィンドウ処理のサポートが必要なBoltによって実装されています。

```java
public interface IWindowedBolt extends IComponent {
    void prepare(Map stormConf, TopologyContext context, OutputCollector collector);
    /**
     * Process tuples falling within the window and optionally emit 
     * new tuples based on the tuples in the input window.
     */
    void execute(TupleWindow inputWindow);
    void cleanup();
}
```

ウィンドウがアクティブになるたびに、`execute`メソッドが呼び出されます。
TupleWindowパラメータで、ウィンドウ内の現在のタプル、有効期限が切れたタプル、および最後のウィンドウが計算された後に追加された新しいタプルにアクセスできるため、
効率的なウィンドウ処理の計算に役立ちます。

ウィンドウのサポートを必要とするBoltは、通常、
ウィンドウの長さとスライドの間隔を指定するためのAPIを持つ`BaseWindowedBolt`を拡張します。

例えば、

```java
public class SlidingWindowBolt extends BaseWindowedBolt {
	private OutputCollector collector;
	
    @Override
    public void prepare(Map stormConf, TopologyContext context, OutputCollector collector) {
    	this.collector = collector;
    }
	
    @Override
    public void execute(TupleWindow inputWindow) {
	  for(Tuple tuple: inputWindow.get()) {
	    // do the windowing computation
		...
	  }
	  // emit the results
	  collector.emit(new Values(computedValue));
    }
}

public static void main(String[] args) {
    TopologyBuilder builder = new TopologyBuilder();
     builder.setSpout("spout", new RandomSentenceSpout(), 1);
     builder.setBolt("slidingwindowbolt", 
                     new SlidingWindowBolt().withWindow(new Count(30), new Count(10)),
                     1).shuffleGrouping("spout");
    Config conf = new Config();
    conf.setDebug(true);
    conf.setNumWorkers(1);

    StormSubmitter.submitTopologyWithProgressBar(args[0], conf, builder.createTopology());
	
}
```

以下のウィンドウ設定がサポートされています。

```java
withWindow(Count windowLength, Count slidingInterval)
Tuple count based sliding window that slides after `slidingInterval` number of tuples.

withWindow(Count windowLength)
Tuple count based window that slides with every incoming tuple.

withWindow(Count windowLength, Duration slidingInterval)
Tuple count based sliding window that slides after `slidingInterval` time duration.

withWindow(Duration windowLength, Duration slidingInterval)
Time duration based sliding window that slides after `slidingInterval` time duration.

withWindow(Duration windowLength)
Time duration based window that slides with every incoming tuple.

withWindow(Duration windowLength, Count slidingInterval)
Time duration based sliding window configuration that slides after `slidingInterval` number of tuples.

withTumblingWindow(BaseWindowedBolt.Count count)
Count based tumbling window that tumbles after the specified count of tuples.

withTumblingWindow(BaseWindowedBolt.Duration duration)
Time duration based tumbling window that tumbles after the specified time duration.

```

## Tuple timestamp and out of order tuples
デフォルトでは、ウィンドウ内で追跡されるタイムスタンプは、タプルがBoltで処理される時刻です。ウィンドウの計算は、processing timestampに基づいて実行されます。
Stormは、source generated timestampに基づいてウィンドウを追跡する機能をサポートしています。

```java
/**
* Specify a field in the tuple that represents the timestamp as a long value. If this
* field is not present in the incoming tuple, an {@link IllegalArgumentException} will be thrown.
*
* @param fieldName the name of the field that contains the timestamp
*/
public BaseWindowedBolt withTimestampField(String fieldName)
```

上記の`fieldName`の値は入力タプルからルックアップされ、ウィンドウ計算で考慮されます。
フィールドがタプルに存在しない場合、例外がスローされます。タイムスタンプのフィールド名とともに、タイムスタンプの順序が乱れたタプルの最大時間制限を示すタイムラグパラメータも指定できます。

例えば、lagが5秒で、タプル`t1`がタイムスタンプ`06：00：05`で到着した場合、他のタプルがタイムスタンプ`06：00：00`より早く到着することはありません。
あるタプルが`t1`の後にタイムスタンプ05:59:59で到着し、ウィンドウが`t1`を過ぎて移動した場合、それは遅いタプルとして扱われ、処理されません。現在、遅れてきたタプルは、ワーカーのログファイルにINFOレベルで記録されています。

```java
/**
* Specify the maximum time lag of the tuple timestamp in milliseconds. It means that the tuple timestamps
* cannot be out of order by more than this amount.
*
* @param duration the max lag duration
*/
public BaseWindowedBolt withLag(Duration duration)
```

### Watermarks
タイムスタンプのフィールドを有するタプルを処理するために、Stormは入力タプルのタイムスタンプに基づいてwatermarkを内部的に計算します。
Watermarkは、すべての入力ストリームにわたる最新のタプルタイムスタンプの最小値(から、ラグを差し引いたもの)です。
より高いレベルでは、これは、イベントベースのタイムスタンプを追跡するためにFlinkおよびGoogleのMillWheelによって使用されるwatermarkのコンセプトと似たものです。

タプルベースのタイムスタンプが使用されている場合は、定期的に(デフォルトでは毎秒)、watermarkのタイムスタンプがemitされ、これはウィンドウ計算のクロックティックとみなされます。 
watermarkがemitされる間隔は、下記のAPIで変更できます。
 
```java
/**
* Specify the watermark event generation interval. For tuple based timestamps, watermark events
* are used to track the progress of time
*
* @param interval the interval at which watermark events are generated
*/
public BaseWindowedBolt withWatermarkInterval(Duration interval)
```


watermarkが受信されると、そのタイムスタンプまでのすべてのウィンドウが評価されます。

たとえば、次のウィンドウパラメーターでタプルのタイムスタンプに基づく処理を行うとすると、

`Window length = 20s, sliding interval = 10s, watermark emit frequency = 1s, max lag = 5s`

```
|-----|-----|-----|-----|-----|-----|-----|
0     10    20    30    40    50    60    70
````

現時刻を ts = `09:00:00` として、

タプル`e1(6:00:03), e2(6:00:05), e3(6:00:07), e4(6:00:18), e5(6:00:26), e6(6:00:36)`が`9:00:00`から`9:00:01`の間に受信されたとすると、

時刻 t = `09:00:01` において、`6:00:31`より早くタプルが到達することはないので、 watermark w1 = `6:00:31` がemitされます。

3つのウィンドウが評価されます。最初のウィンドウ終了時刻 ts(06:00:10) は、最も早いイベントタイムスタンプ (06:00:03) をとることと、スライド間隔(10s)に基づいてceilingを計算することによって導かれます。

1. `5:59:50 - 06:00:10` with tuples e1, e2, e3
2. `6:00:00 - 06:00:20` with tuples e1, e2, e3, e4
3. `6:00:10 - 06:00:30` with tuples e4, e5

e6は、watermarkタイムスタンプ`6:00:31`がタプルのタイムスタンプts`6:00:36`より古いため、評価されません。

タプル `e7(8:00:25), e8(8:00:26), e9(8:00:27), e10(8:00:39)` は、 `9:00:01` から `9:00:02` の間に受信されます。

時刻 t = `09:00:02` において、`8:00:34`より早くタプルが到達することはないので、別の watermark w2 = `08:00:34` がemitされます。

3つのウィンドウが評価され、

1. `6:00:20 - 06:00:40` with tuples e5, e6 (from earlier batch)
2. `6:00:30 - 06:00:50` with tuple e6 (from earlier batch)
3. `8:00:10 - 08:00:30` with tuples e7, e8, e9

e10は、タプルのタイムスタンプ `8:00:39` が、watermarkのタイムスタンプ`8:00:34`を超えているため、評価されません。

ウィンドウは、時間のギャップを考慮し、タプルのタイムスタンプに基づいてウィンドウを計算することによって導出されています。

## Guarentees
Stormコアのウインドウ機能は、現在、at-least onceを保証しています。Boltの`execute(TupleWindow inputWindow)`メソッドからemitされた値は自動的にinputWindowのすべてのタプルにanchorされます。下流のBoltは、受け取ったタプル（すなわちwindow化されたBoltからemitされたタプル）にackし、タプルツリーを完了させる必要があります。そうでない場合、タプルがリプレイされ、ウィンドウ演算が再評価されます。

ウィンドウ内のタプルは、有効期限が切れたとき、すなわち、`windowLength + slidingInterval`経過してウィンドウから抜けたときに、自動的にackされます。つまり、ウィンドウのタプルは、`windowLength + slidingInterval`です。設定`topology.message.timeout.secs`は、時間ベースのウィンドウの場合は` windowLength + slidingInterval`よりも十分に大きくなければならないことに注意してください; そうしないと、タプルはタイムアウトしてリプレイされ、重複した評価が行われる可能性があります。カウントベースのウィンドウでは、`windowLength + slidingInterval`を、タプルをタイムアウト時間内に受信できるように同じ設定について調整する必要があります。

## Example topology
例`SlidingWindowTopology`は、
これらのAPIを使用してスライディングウィンドウによる合計とタンブリングウィンドウによる平均を計算する方法を示しています。


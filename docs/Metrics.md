---
title: Storm Metrics
layout: documentation
documentation: true
---
Stormは、トポロジ全体の要約統計情報をレポートするために、メトリクスについてのインターフェイスを公開しています。
この情報はNimbusのUIコンソールに表示される数字をトラッキングするために内部的に使用されています。処理した数やackの数; boltあたりの平均プロセス待ち時間; ワーカーヒープの使用状況; 等々。

### Metric Types

メトリクスは、ただ一つのメソッドを持つ[`IMetric`]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/IMetric.java)を実装する必要があります。そのメソッドは、`getValueAndReset`です - 要約値を見つけるための作業を行い、初期状態に戻します。たとえば、MeanReducerは、累積合計を累積の個数で除算して平均を求め、次に両方の値をゼロに初期化します。

Stormは以下のようなメトリクスを提供します:

* [AssignableMetric]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/AssignableMetric.java) -- 指定した明示的な値にメトリクスを設定する。それが外部の値である場合や、すでに要約統計量を自分で計算している場合に役立ちます。
* [CombinedMetric]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/CombinedMetric.java) -- 関連付けて更新可能なメトリクスの汎用インタフェース。
* [CountMetric]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/CountMetric.java) -- 指定された値の実行時の合計です。`incr()`を呼び出すとインクリメントし、`incrBy(n)`は与えられた数値を加算/減算します。
  - [MultiCountMetric]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/MultiCountMetric.java) -- CountMetricのハッシュマップ。
* [ReducedMetric]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/ReducedMetric.java)
  - [MeanReducer]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/MeanReducer.java) -- `reduce()`メソッドに与えられた値の移動平均をトラッキングします。(`Double`、`Integer`または`Long`の値を受け取り、内部平均を「Double」として維持します。)彼の評判に反して、MeanReducerは実際にはかなり良い人です。
  - [MultiReducedMetric]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/MultiReducedMetric.java) -- ReducedMetricのハッシュマップ。


### Metrics Consumer

トポロジにMetrics Consumerを登録することで、トポロジに対するのメトリクスをリッスン・処理できます。

Metrics Consumerをトポロジに登録するには、トポロジの設定に次のように追加します:

```java
conf.registerMetricsConsumer(org.apache.storm.metric.LoggingMetricsConsumer.class, 1);
```

[Config#registerMetricsConsumer](javadocs/org/apache/storm/Config.html#registerMetricsConsumer-java.lang.Class-) と、javadocにあるオーバーロードされたメソッドを参照できます。

あるいは、storm.yaml設定ファイルを編集します:

```yaml
topology.metrics.consumer.register:
  - class: "org.apache.storm.metric.LoggingMetricsConsumer"
    parallelism.hint: 1
  - class: "org.apache.storm.metric.HttpForwardingMetricsConsumer"
    parallelism.hint: 1
    argument: "http://example.com:8080/metrics/my-topology/"
```

Stormは、登録された各メトリック・コンシューマごとにMetricsConsumerBoltをトポロジに内部的に追加し、各MetricsConsumerBoltはすべてのタスクからメトリックを受信するようにサブスクライブします。そのBoltの並列性は `parallelism.hint`に設定され、そのBoltの`component id`は`__metrics_ <metrics consumer class name>`に設定されます。同じクラス名を複数回登録すると、コンポーネントIDの末尾に`#<sequence number>`が追加されます。

Stormには、トポロジにどんなメトリックが提供されているかを試すための組込みメトリックコンシューマが用意されています。

* [`LoggingMetricsConsumer`]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/LoggingMetricsConsumer.java) -- すべてのメトリックをリッスンし、TSV（タブ区切り値）ファイルにログを記録します 。
* [`HttpForwardingMetricsConsumer`]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/HttpForwardingMetricsConsumer.java) -- HTTP経由で設定されたURLにシリアル化されたすべてのメトリックとPOSTをリッスンします。Stormは抽象クラスとして[`HttpForwardingMetricsServer`]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/HttpForwardingMetricsServer.java)を提供ており、これを継承したクラスをHTTPサーバーとして実行することができ、HttpForwardingMetricsConsumerによって送信されたメトリックを処理します。

また、StormはMetrics Consumerを実装するためのインタフェース[`IMetricsConsumer`]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/IMetricsConsumer.java)を公開しているため、カスタムのMetrics Consumerを作成してトポロジにアタッチしたり、Stormコミュニティが提供するMetrics Consumersの他の優れた実装を使用することができます。例として、[versign/storm-graphite](https://github.com/verisign/storm-graphite)や[storm-metrics-statsd](https://github.com/endgameinc/storm-metrics-statsd)があります。

独自のメトリックスコンシューマを実装する場合、[IMetricsConsumer#prepare](javadocs/org/apache/storm/metric/api/IMetricsConsumer.html#prepare-java.util.Map-java.lang.Object-org.apache.storm.task.TopologyContext-org.apache.storm.task.IErrorReporter-)が呼び出されたときに`argument`がオブジェクトに渡されるので、yamlに推測したJavaの型を設定し、明示的なキャストを行う必要があります。

MetricsConsumerBoltはBoltの一種であり、登録されたMetrics Consumerが受信したメトリックを処理できない場合にトポロジ全体のスループットが低下するため、通常のBoltのようにこれらのBoltを処理したい場合があります。これを避けるためのアイデアの1つは、Metrics Consumerの実装を`non-blocking`方式にすることです。


### Build your own metric (task level)

Metric Registryに`IMetric`を登録することで、独自のメトリックを測定できます。

Bolt＃executeの実行回数を計測したいとします。メトリックインスタンスの定義から始めましょう。CountMetricは私たちのユースケースに合っているようです。

```java
private transient CountMetric countMetric;
```

上記ではtransientとして定義しています。IMerticはシリアライズできないため、シリアライゼーションの問題を避けるためにtransientを使用しています。

次に、メトリックインスタンスを初期化して登録しましょう。

```java
@Override
public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
	// other intialization here.
	countMetric = new CountMetric();
	context.registerMetric("execute_count", countMetric, 60);
}
```

1番目と2番目のパラメータの意味は、IMetricの単純なメトリック名とインスタンスです。[TopologyContext#registerMetric](javadocs/org/apache/storm/task/TopologyContext.html#registerMetric-java.lang.String-T-int-)の3番目のパラメータは、メトリックをパブリッシュおよびリセットする期間（秒）です。

最後に、Bolt.execute()が実行されたときに値を増やしましょう。

```java
public void execute(Tuple input) {
	countMetric.incr();
	// handle tuple here.	
}
```

トポロジメトリックのサンプルレートは、独自のメトリックには適用されません。これは、自らincr()を呼び出しているためです。

完了!`countMetric.getValueAndReset()`は、登録された間隔である60秒ごとに呼び出され、 ("execute_count", value)のペアはMetricsConsumerにプッシュされます。


### Build your own metrics (worker level)

独自のワーカー・レベル・メトリックは、`Config.WORKER_METRICS`に追加するとクラスタ内のすべてのワーカーに対して、または、`Config.TOPOLOGY_WORKER_METRICS`に追加することで特定のトポロジ内のすべてのワーカー用に、登録できます。

たとえば、clusterのstorm.yamlに`worker.metrics`を追加することができます。

```yaml
worker.metrics: 
  metricA: "aaa.bbb.ccc.ddd.MetricA"
  metricB: "aaa.bbb.ccc.ddd.MetricB"
  ...
```

または、設定のマップにキー`Config.TOPOLOGY_WORKER_METRICS`を``Map<String, String>`（メトリック名、メトリッククラス名）としてputしてください。

ワーカーレベルのメトリックインスタンスにはいくつかの制限があります:

A) ワーカーレベルのメトリックは、SystemBoltから初期化されて登録され、ユーザータスクには公開されていないため、計測器の一種である必要があります。

B) メトリックはデフォルトのコンストラクタで初期化され、コンフィグレーションやオブジェクトの注入は行われません。

C) メトリックのバケットサイズ（秒）は、`Config.TOPOLOGY_BUILTIN_METRICS_BUCKET_SIZE_SECS`に固定されます。


### Builtin Metrics

[組み込みのメトリクス]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/daemon/builtin_metrics.clj)はStormに備えられています。

[builtin_metrics.clj]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/daemon/builtin_metrics.clj)は、組み込みメトリックのデータ構造、および他のフレームワークコンポーネントがそれらを更新するために使用できるファサードメソッドを用意しています。
メトリック自体は、呼び出しているコードで計算されます -- たとえば、`clj/b/s/daemon/daemon/executor.clj`の[`ack-spout-msg`]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/daemon/executor.clj#358)を参照。


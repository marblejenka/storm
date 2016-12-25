---
title: Common Topology Patterns
layout: documentation
documentation: true
---

このページには、Stormトポロジのさまざまな共通パターンが挙げられています。

1. Streaming joins
2. Batching
3. BasicBolt
4. In-memory caching + fields grouping combo
5. Streaming top N
6. TimeCacheMap for efficiently keeping a cache of things that have been recently updated
7. CoordinatedBolt and KeyedFairBolt for Distributed RPC

### Joins

ストリーミング結合は、共通のフィールドに基づいて2つ以上のデータストリームを結合します。通常のデータベースの結合は入力が有限でありクリアなセマンティクスであると言えますが、ストリーミング結合は入力が無限であり不明瞭なセマンティクスであると言えます。

必要な結合の種類は、アプリケーションごとに異なります。一部のアプリケーションは、有限のタイムウィンドウにおいて2つのストリームのすべてのタプルをjoinしますが、他のアプリケーションでは、各結合フィールドの各側にただ1つのタプルを期待してjoinします。さらに他のアプリケーションでは、完全に異なる結合を行いたい場合があります。これらすべての結合タイプ間の共通パターンは、複数の入力ストリームを同じ方法で分割できることです。これは、Stormでは、Joiner Boltへのいくつかの入力ストリームに対して、同じフィールドでFields Groupingを使用することで簡単に実行できます。例えば:

```java
builder.setBolt("join", new MyJoiner(), parallelism)
  .fieldsGrouping("1", new Fields("joinfield1", "joinfield2"))
  .fieldsGrouping("2", new Fields("joinfield1", "joinfield2"))
  .fieldsGrouping("3", new Fields("joinfield1", "joinfield2"));
```

もちろん、異なるストリームが同じフィールド名を持つ必要はありません。


### Batching

効率やその他の理由から、タプルのグループを個別に処理するのではなくバッチで処理したいことがよくあります。たとえば、データベースへの更新をバッチ処理したり、ある種のストリーミング集約を実行したいときです。

データ処理の信頼性を望む場合、これを行う正しい方法は、バッチ処理を行うのを待っている間にインスタンス変数のタプルを保持することです。一度バッチ処理を実行すると、保持していたすべてのタプルをackできます。

Boltがタプルを出力する際に、multi-anchoringを使用して信頼性を確保することができます。それはすべてアプリケーション固有の条件に依存します。信頼性の仕組みの詳細については、[メッセージ処理の保証](Guaranteeing-message-processing.html) を参照してください。

### BasicBolt
多くのボルトは、入力タプルを読み取って、その入力タプルに基づいてゼロ個以上のタプルを出力し、次にexecuteメソッドの終了時にその入力タプルを即座にackするという同様のパターンに従います。 このパターンに合ったBoltは、関数やフィルターのようなものです。これは、Stormがこのパターンを自動化する[IBasicBolt](javadocs/org/apache/storm/topology/IBasicBolt.html) というインターフェースで公開しているような共通のパターンです。 詳細については、[メッセージ処理の保証](Guaranteeing-message-processing.html)を参照してください。

### In-memory caching + fields grouping combo

Stormボルトでキャッシュをメモリ内に保持するのは一般的です。キャッシングは、フィールドのグループ化と組み合わせると特に強力になります。たとえば、短いURL(bit.ly、t.coなど)を長いURLに展開するBoltがあるとします。短いURLから展開された長いURLのLRUキャッシュを保持することによって同じHTTP要求を何度も繰り返さないようにし、パフォーマンスを向上させることができます。コンポーネント"urls"が短いURLを送出し、コンポーネント"expand"が短いURLを長いURLに展開するとともに内部的にキャッシュを保持すると仮定します。以下の2つのコードスニペットの違いを考えてみましょう。

```java
builder.setBolt("expand", new ExpandUrl(), parallelism)
  .shuffleGrouping(1);
```

```java
builder.setBolt("expand", new ExpandUrl(), parallelism)
  .fieldsGrouping("urls", new Fields("url"));
```

2番目の方法は、同じURLが常に同じタスクになるので、より効果的なキャッシュを実現します。これにより、タスク内のどのキャッシュにおいても重複が起こることがなくなり、短いURLがキャッシュにヒットする可能性が高くなります。

### Streaming top N

Stormで行われる一般的な連続計算は、"streaming top N"的なものです。["value", "count"]という形式のタプルを送出するBoltがあり、カウントに基づいてTop N個のタプルを出力するBoltを必要としているとします。これを行う最も簡単な方法は、ストリーム上でグローバルなグループ化を行い、上位N項目のリストをメモリ内に保持するBoltを持つことです。

このアプローチは、ストリーム全体が1つのタスクを通過しなければならないため、大きなストリームに対応するようスケールさせることはできません。計算を行うより良い方法は、ストリームの複数のパーティションにわたって多数のTop Nを並列に実行し、それらのTop Nを結合してグローバルなTop Nを得ることです。パターンは次のようになります:

```java
builder.setBolt("rank", new RankObjects(), parallelism)
  .fieldsGrouping("objects", new Fields("value"));
builder.setBolt("merge", new MergeObjects())
  .globalGrouping("rank");
```

このパターンは、最初のBoltによってFields Groupingをしているため、意味的に正しいパーティショニングを行えており、機能します。storm-starterの[これ]({{page.git-blob-base}}/examples/storm-starter/src/jvm/org/apache/storm/starter/RollingTopWords.java)でこのパターンの例を見ることができます。 。

ただし、処理中のデータに既知のスキューがある場合は、fieldsGroupingの代わりにpartialKeyGroupingを使用すると便利です。 これにより、1つの下流のBoltの代わりに、2つの下流のBoltに負荷が分散されます。

```java
builder.setBolt("count", new CountObjects(), parallelism)
  .partialKeyGrouping("objects", new Fields("value"));
builder.setBolt("rank" new AggregateCountsAndRank(), parallelism)
  .fieldsGrouping("count", new Fields("key"))
builder.setBolt("merge", new MergeRanksObjects())
  .globalGrouping("rank");
``` 

トポロジでは、上流のボルトからの部分カウントを集計するために余分な処理層が必要ですが、これは集計された値を処理するだけなので、Boltはスキューしたデータによる負荷の影響を受けません。storm-starterの[ここ]({{page.git-blob-base}}/examples/storm-starter/src/jvm/org/apache/storm/starter/SkewedRollingTopWords.java)でこのパターンの例を見ることができます。 。

### TimeCacheMap for efficiently keeping a cache of things that have been recently updated

直近で"active"だったアイテムをキャッシュに残しておき、しばらくの間アクティブでないアイテムを自動的に期限切れにしたい場合があります。[TimeCacheMap](javadocs/org/apache/storm/utils/TimeCacheMap.html)は、これを行うための効率的なデータ構造であり、アイテムが期限切れになった際のコールバックを挿入できるフックを提供します。

### CoordinatedBolt and KeyedFairBolt for Distributed RPC

Stormの上に分散RPCアプリケーションを構築する場合、通常は2つの共通パターンが必要です。これらはStormのコードベースに同梱されている"標準ライブラリ"の一部である[CoordinatedBolt](javadocs/org/apache/storm/task/CoordinatedBolt.html)と[KeyedFairBolt](javadocs/org/apache/storm/task/KeyedFairBolt.html)でカプセル化されています。

`CoordinatedBolt`はロジックを含むBoltをラップし、あなたのBoltが任意のリクエストに対してすべてのタプルを受け取ったときに出力されます。これはDirect Streamを頻繁に使用します。

`KeyedFairBolt`はロジックを含むBoltをラップし、一度に1つずつ直列に行うのではなく、トポロジが同時に複数のDRPC呼び出しを処理するようにします。

詳細は[Distributed RPC](Distributed-RPC.html)を参照してください。

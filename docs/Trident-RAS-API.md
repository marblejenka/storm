---
title: Trident RAS API
layout: documentation
documentation: true
---

## Trident RAS API

Trident RAS(Resource Aware Scheduler)APIは、ユーザーがTridentトポロジのリソース消費量を指定できるようにするメカニズムを提供します。APIはRAS APIとまったく同じように見えますが、BoltとSpoutの代わりにTrident Streamsで呼び出される点のみが異なります。

ドキュメントの重複や矛盾を避けるため、リソース設定の目的と効果についてはここでは説明しませんが、[Resource Aware Scheduler Overview](Resource_Aware_Scheduler_overview.html)に記載されています。

### Use

まずは例から:

```java
    TridentTopology topo = new TridentTopology();
    TridentState wordCounts =
        topology
            .newStream("words", feeder)
            .parallelismHint(5)
            .setCPULoad(20)
            .setMemoryLoad(512,256)
            .each( new Fields("sentence"),  new Split(), new Fields("word"))
            .setCPULoad(10)
            .setMemoryLoad(512)
            .each(new Fields("word"), new BangAdder(), new Fields("word!"))
            .parallelismHint(10)
            .setCPULoad(50)
            .setMemoryLoad(1024)
            .groupBy(new Fields("word!"))
            .persistentAggregate(new MemoryMapState.Factory(), new Count(), new Fields("count"))
            .setCPULoad(100)
            .setMemoryLoad(2048);
```

リソースは、オペレーションごとに設定できます(グループ化、シャッフル、パーティション分割を除く)。
Tridentが単一のボルトに統合したオペレーションは、そのリソースを合計します。

ユーザー設定にかかわらず、すべてのBoltには**少なくとも**デフォルトのリソースが与えられます。

上記の場合、私たちは
In the above case, we end up with


- それぞれ20％のCPUロードと、512MiBのオン・ヒープと256MiBのオフ・ヒープのメモリロードとなるSpoutとSpout Coordinator
- SplitとBangAdderを組み合わせて60％のCPUロード(10％ + 50％)と1536MiB(1024 + 512)のメモリロードとなるBolt
- 100％CPUロードと2048MiBオン・ヒープのメモリロードとなるBoltで、オフ・ヒープについてはデフォルト値。

APIは必要な回数だけ呼び出すことができます。
すべてのオペレーションの後で呼び出されたり、いくつかのオペレーションの後に呼び出される、parallelismHint()と同じ方法で使用されセクション全体のリソースを設定するなどがありえます。
リソース宣言は、並列性ヒントと同じ*境界*を持ちます。グループ化、シャッフル、またはその他の種類のパーティション分割をまたぎません。

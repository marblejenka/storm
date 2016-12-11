---
title: Structure of the Codebase
layout: documentation
documentation: true
---
Stormのコードベースには3つの独立したレイヤーがあります。

第一に、Stormは最初から複数の言語と互換性があるように設計されていました。NimbusはThriftサービスであり、トポロジはThriftの構造として定義されています。Thriftを使用することによって、Stormはどの言語からでも使用できるようになっています。

第二に、StormのすべてのインターフェースはJavaのインターフェースとして明示されています。Stormの実装の多くにClojureが使用されていても、すべての使用法はJava APIを経由しなければなりません。 つまり、Stormのすべての機能は常にJava経由で利用できます。

第3に、Stormの実装は主にClojureにあります。コードの行数では、StormはおおむねJavaのコードの半分、Clojureのコード半分でできています。しかし、Clojureははるかに表現力豊かなので、実際には実装ロジックの大部分がClojureにあります。

以降のセクションでは、これらのレイヤーのそれぞれについて詳しく説明します。

### storm.thrift

Stormのコードベースの構造を理解するための最初の場所は、[storm.thrift]({{page.git-blob-base}}/storm-core/src/storm.thrift) ファイルです。

Stormは、コードを生成するために、Thriftの[フォーク](https://github.com/nathanmarz/thrift/tree/storm)（'storm'ブランチ）を使用します。この "フォーク"は、実際にはThrift 7であり、すべてのJavaパッケージはorg.apache.thrift7という名前に変更されています。 それ以外についてはThrift 7と同じです。このフォークは、Thriftに下位互換性なかったことと、多くの人がStormトポロジで他のバージョンのThriftを使用する必要があったことを理由に行われました。

トポロジ内のすべてのspoutまたはboltには、「コンポーネントID」と呼ばれるユーザー指定の識別子が与えられます。コンポーネントIDは、boltから他のspoutまたはboltの出力ストリームへのサブスクリプションを指定するために使用されます。 [StormTopology]({{page.git-blob-base}}/storm-core/src/storm.thrift#L91) の構造は、コンポーネントIDから、コンポーネントの各タイプ（spoutまたはbolt）への対応付けが含まれています。

spoutとboltはThriftの定義上同一であるため、[Thriftによるboltの定義]({{page.git-blob-base}}/storm-core/src/storm.thrift#L79)を見てみましょう。 それは`ComponentObject`構造体と`ComponentCommon`構造体を含んでいます。

`ComponentObject`はboltの実装を定義します。 次の3つのタイプのいずれかになります:

1. シリアライズされたjavaオブジェクト ([IBolt]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/task/IBolt.java) の実装)
2. 実装が別の言語であることを示す`ShellComponent`オブジェクト。 このようにボルトを指定すると、Stormは[ShellBolt]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/task/ShellBolt.java)オブジェクトをインスタンス化し、JVMベースのワーカープロセスと非JVMベースのコンポーネント実装間の通信を処理します。
3. Stormにそのボルトのインスタンス化に使用するクラス名とコンストラクタ引数を伝える`JavaObject`構造体。 これは、非JVM言語でトポロジを定義する場合に便利です。 こうすることで、Javaオブジェクトを自分で作成してシリアライズすることなく、JVMベースのspoutとboltを利用できます。

`ComponentCommon` このコンポーネントのその他すべてを定義します。 以下のものを含みます:

1. このコンポーネントが送出するストリームと各ストリームのメタデータ（直接ストリームであるかフィールド宣言であるか）
2. このコンポーネントが消費するストリーム（component_id：stream_idから使用するストリームグループへの対応付けとして指定）
3. このコンポーネントのparallelism
4. コンポーネント固有の[設定](Configuration.html)

構造体のspoutも `ComponentCommon`フィールドを持っているので、spoutは他の入力ストリームを消費する宣言を持つこともできます。しかし、Storm Java APIでは、スパウトが他のストリームを消費する方法は提供されていません。spoutに何かしらの入力宣言を書くと、トポロジをsubmitしようとしたときにエラーが発生します。 spoutに入力宣言フィールドがある理由は、ユーザーが使用するのではなく、Storm自身が使用するためです。Stormは、[acking framework](https://github.com/apache/storm/wiki/Guaranteeing-message-processing)をセットアップするためにトポロジに暗黙的なストリームとboltを追加します。これら2つの暗黙的なストリームは、acker boltからトポロジ内の各spoutへ接続します。ackerは、tuple treeが完了または失敗したことが検出されるたびに、これらのストリームに "ack"または "fail"メッセージを送信します。 ユーザーのトポロジを実行時のトポロジに変換するコードは、[ここ]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/daemon/common.clj#L279)にあります。

### Javaインターフェイス

Stormのインターフェースは、通常、Javaインターフェースとして明示されます。 主要なインタフェースは次のとおりです:

1. [IRichBolt](javadocs/org/apache/storm/topology/IRichBolt.html)
2. [IRichSpout](javadocs/org/apache/storm/topology/IRichSpout.html)
3. [TopologyBuilder](javadocs/org/apache/storm/topology/TopologyBuilder.html)

インターフェイスの多くについて、以下の戦略を採っています:

1. Javaのinterfaceを使用してインタフェースを明示する
2. 必要に応じてデフォルトの実装を提供する基本クラスを提供する

この戦略は、[BaseRichSpout](javadocs/org/apache/storm/topology/base/BaseRichSpout.html)クラスで確認できます。

spoutとboltは、上述のようにトポロジのThrift定義にシリアライズされます。

インターフェイスの微妙な側面1つは、`IBolt`、`ISpout`と`IRichBolt`、`IRichSpout`の違いです。それらの主な違いは、インターフェイスの "Rich"バージョンには`declareOutputFields`メソッドが追加されていることです。分割の理由は、各出力ストリームの出力フィールド宣言はThrift構造体の一部である必要がある一方で（したがって、任意の言語から指定できます）、ユーザーとしてはストリームをユーザーのクラスの一部として宣言できるようにするためです。  Thrift表現を構築するときに`TopologyBuilder`が行うのは、`declareOutputFields`を呼び出して宣言を取得し、それをThrift構造体に変換することです。この変換は、`TopologyBuilder`コードの[この部分]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/topology/TopologyBuilder.java#L205)で行われます。

### Implementation

Javaインターフェイス経由ですべての機能を明示することにより、Stormのすべての機能がJavaを介して利用できることを保証しています。Javaインターフェイスに焦点を当てることにより、Javaの世界からのユーザーエクスペリエンスが快適であることを保証します。

一方、Stormの実装は、主にClojureで行われています。コードベースは行数ではJavaが約50％、Clojureが約50％ですが、実装ロジックのほとんどはClojureにあります。これには2つの注目すべき例外があります。それが[DRPC](https://github.com/apache/storm/wiki/Distributed-RPC)と[トランザクショナルなトポロジ](https://github.com/apache/storm/wiki/Transactional-topologies)の実装です。これらは純粋にJavaで実装されています。これは、Stormでより高いレベルの抽象化を実装するための実例に供するために行われました。DRPCとトランザクショナルなトポロジの実装は、 [org.apache.storm.coordination]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/coordination), [org.apache.storm.drpc]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/drpc), ならびに [org.apache.storm.transactional]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/transactional)パッケージにあります。

以下が主なJavaパッケージとClojureネームスペースの目的をまとめたものです:

#### Java packages

[org.apache.storm.coordination]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/coordination): Stormでバッチ処理を実現するために必要な部分を実装しています。DRPCとトランザクショナルなトポロジの両方が使用しています。ここでは、`CoordinatedBolt`が最も重要なクラスです。

[org.apache.storm.drpc]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/drpc): DRPCの高レベルな抽象化の実装

[org.apache.storm.generated]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/generated): 生成されたStorm用のThriftコード（生成されたコードはThriftの[このフォーク](https://github.com/nathanmarz/thrift)を使用して生成されます。単に他のThriftバージョンとの競合を避けるためにパッケージをorg.apache.thrift7にリネームします）

[org.apache.storm.grouping]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/grouping): カスタムストリームグループを作成するためのインターフェイスが含まれています

[org.apache.storm.hooks]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/hooks): Stormのさまざまなイベントにフックするためのインターフェース。タスクがタプルを送出するとき、タプルがackされるときなど。フックのユーザガイドは[こちら](https://github.com/apache/storm/wiki/Hooks)です。

[org.apache.storm.serialization]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/serialization): タプルのシリアライズ/デシリアライズ方法の実装。[Kryo](http://code.google.com/p/kryo/)の上に構築されています。

[org.apache.storm.spout]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/spout): spoutとそれに関連するインタフェースの定義（`SpoutOutputCollector `など）。非JVM言語でspoutを定義するためのプロトコルを実装する `ShellSpout`も含まれています。

[org.apache.storm.task]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/task): boltとそれに関連するインターフェースの定義（`OutputCollector`など）。また、非JVM言語でボルトを定義するためのプロトコルを実装する `ShellBolt`も含まれています。最後に、`TopologyContext`もここに定義されています。これはspotとboltに提供されており、実行時にトポロジとその実行に関するデータを取得できるようにここで定義されています。

[org.apache.storm.testing]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/testing): Stormのユニットテストに使用するためのさまざまなテスト用のboltとユーティリティが含まれています。

[org.apache.storm.topology]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/topology): 基盤となるThrift構造のJavaレイヤーで、クリーンで純粋なJava APIをStormに提供するためのものです（ユーザーはThriftについて知る必要はありません）。 `TopologyBuilder`は、さまざまなspoutとboltの役に立つ基本クラスと同様にここにあります。若干高いレベルの `IBasicBolt`インターフェースがここにあります。これは、特定の種類のボルトを実装するためのシンプルな方法です。

[org.apache.storm.transactional]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/transactional): トランザクショナルなトポロジの実装です。

[org.apache.storm.tuple]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/tuple): Stormのデータモデルであるタプルについての実装です。

[org.apache.storm.utils]({{page.git-tree-base}}/storm-core/src/jvm/org/apache/storm/tuple): コードベース全体で使用される、データ構造やその他のユーティリティです。

#### Clojure namespaces

[org.apache.storm.bootstrap]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/bootstrap.clj): すべてのクラスをインポートするのに役立つマクロや、コードベース全体で使用される名前空間などが含まれています。

[org.apache.storm.clojure]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/clojure.clj): StormのClojure DSLの実装。

[org.apache.storm.cluster]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/cluster.clj): Stormデーモンで使用されるすべてのZookeeperロジックがこのファイルにカプセル化されています。このコードは、クラスタの状態（どのようなタスクがどこで実行されているか、どのspout/boltがどのタスクのために実行されているか）をZookeeperの"filesystem" APIに対応付けされる方法を管理します。

[org.apache.storm.command.*]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/command): これらの名前空間は、 `storm`コマンドラインクライアントのためのコマンドを実装しています。これらの実装は非常に短いです。

[org.apache.storm.config]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/config.clj): Clojureのコード読み取り/解析コードの実装。また、様々な用途で使用されるnimbus/supervisor/daemonsのローカルパスを決定するユーティリティ関数もあります。例えば`master-inbox`関数はjarがアップロードされるときにNimbusが使うべきローカルパスを返します。

[org.apache.storm.daemon.acker]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/daemon/acker.clj): "acker" boltの実装。これはStormがどのようにデータ処理を保証するかの重要な部分です。

[org.apache.storm.daemon.common]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/daemon/common.clj): Stormデーモンで使用される共通関数の実装で、トポロジの名前からIDを取得したり、ユーザーのトポロジを実際に実行するトポロジにマッピングする（暗黙的なackingストリームとacker boltが追加されました - `system-topology!`関数を参照してください）、さまざまなheartbeatやStormによって永続化されたその他の構造体などの定義などがあります。

[org.apache.storm.daemon.drpc]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/daemon/drpc.clj): DRPCトポロジで使用するためのDRPCサーバの実装。

[org.apache.storm.daemon.nimbus]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/daemon/nimbus.clj): Nimbusの実装。

[org.apache.storm.daemon.supervisor]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/daemon/supervisor.clj): スーパーバイザの実装。

[org.apache.storm.daemon.task]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/daemon/task.clj): spoutやboltのなど個々のタスクの実装。メッセージルーティング、シリアライゼーション、UI向けの統計情報の収集、spoutやbolt固有の実行時の処理など。

[org.apache.storm.daemon.worker]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/daemon/worker.clj): ワーカープロセスの実装（これには多くのタスクが含まれます）。メッセージの転送とタスクの起動を実装しています。

[org.apache.storm.event]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/event.clj): シンプルな非同期関数のexecutorを実装しています。NimbusとSupervisorのさまざまな場所で、競合状態を回避するために機能を直列に実行するために使用されます。

[org.apache.storm.log]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/log.clj): log4jにメッセージを記録するための関数を定義しています。

[org.apache.storm.messaging.*]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/messaging): point to pointメッセージングを実装するためのより高いレベルのインタフェースを定義します。ローカルモードでは、Stormはメモリ内のJavaキューを使用してこれを行います。クラスタ上では、ZeroMQを使用します。汎用インタフェースはprotocol.cljで定義されています。

[org.apache.storm.stats]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/stats.clj): UIのためにZKに統計情報を送信する際に使用される統計情報をロールアップするルーチンの実装です。ウィンドウやローリング集約を複数の粒度で扱います。

[org.apache.storm.testing]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/testing.clj): Stormトポロジのテストに使用される機能の実装。時間をシミュレーションする機能や、トポロジーを介して固定セットのタプルを実行し出力をキャプチャする`complete-topology`、クラスタが「アイドル」のとなった検出を細粒度に制御するためのtrackerトポロジ、およびその他のユーティリティが含まれます。

[org.apache.storm.thrift]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/thrift.clj): Thrift構造をより快適に使用するための、生成されたThrift APIに対するClojureラッパー。

[org.apache.storm.timer]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/timer.clj): 未来のタイミング、あるいは定期的に関数を実行するバックグラウンドタイマーの実装。Stormは[Timer](http://docs.oracle.com/javase/1.4.2/docs/api/java/util/Timer.html)クラスを使用できませんでした。なぜならNimbusとSupervisorの単体テストができるようにするには、時間シミュレーションとの統合が必要だったからです。

[org.apache.storm.ui.*]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/ui): Storm UIの実装。残りのコードベースとは完全に独立しており、Nimbus Thrift APIを使用してデータを取得します。

[org.apache.storm.util]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/util.clj): コードベース全体で使用される汎用ユーティリティ関数を含みます。
 
[org.apache.storm.zookeeper]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/zookeeper.clj): ookeeper APIの周りのClojureラッパーで、 "mkdirs"や "delete-recursive"のような "ハイレベル"のものを実装します。
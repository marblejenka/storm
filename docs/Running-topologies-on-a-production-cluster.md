---
title: Running Topologies on a Production Cluster
layout: documentation
documentation: true
---
本場クラスタでトポロジを実行するのは、[ローカルモード](Local-mode.html)で実行するのと同様です。以下がその手順です:

1) Javaを使用して定義する場合は、[TopologyBuilder](javadocs/org/apache/storm/topology/TopologyBuilder.html)でトポロジを定義します

2）[StormSubmitter](javadocs/org/apache/storm/StormSubmitter.html)を使用して、クラスタにトポロジを送信します。`StormSubmitter`は、入力としてトポロジーの名前、トポロジーの設定、トポロジーそのものを受け取ります。例えば:

```java
Config conf = new Config();
conf.setNumWorkers(20);
conf.setMaxSpoutPending(5000);
StormSubmitter.submitTopology("mytopology", conf, topology);
```

3) あなたのコードとあなたのコードのすべての依存関係を含むjarファイルを作成します(Stormを除く -- Storm jarファイルはワーカーノードのクラスパスに追加されます)。

Mavenを使用している場合は、[Maven Assembly Plugin](http://maven.apache.org/plugins/maven-assembly-plugin/)でパッケージ化できます。 これをpom.xmlに追加してください：

```xml
  <plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
      <descriptorRefs>  
        <descriptorRef>jar-with-dependencies</descriptorRef>
      </descriptorRefs>
      <archive>
        <manifest>
          <mainClass>com.path.to.main.Class</mainClass>
        </manifest>
      </archive>
    </configuration>
  </plugin>
```
次に、mvn assembly:assemblyを実行して、適切にパッケージされたjarを取得します。 クラスタにおけるクラスパスにはすでにStormがあるので、Storm jarが[除外](http://maven.apache.org/plugins/maven-assembly-plugin/examples/single/including-and-excluding-artifacts.html)されていることをを必ず確認してください。

4) jarへのパス、実行するクラス名、および使用する引数を指定して、 `storm`クライアントを使用してトポロジをクラスタに送信します:

`storm jar path/to/allmycode.jar org.me.MyTopology arg1 arg2 arg3`

`storm jar`はクラスタにjarファイルを送り、`StormSubmitter`クラスを設定して正しいクラスタと通信します。この例では、jarをアップロードした後、`storm jar`は、引数"arg1", "arg2", ならびに "arg3"を持つ`org.me.MyTopology`のmain関数を呼び出します。

`storm`クライアントがStormクラスタと対話するように[開発環境の設定](Setting-up-development-environment.html)を設定する方法を知ることができます。

### Common configurations

トポロジごとに設定できるさまざまな構成があります。設定できるすべての設定のリストは、[こちら](javadocs/org/apache/storm/Config.html)にあります。"TOPOLOGY"という接頭辞が付いているものは、トポロジ固有の基準で上書きできます(他はクラスタに対する設定であり、上書きすることはできません)。トポロジに設定されている一般的なものを次に示します:

1. **Config.TOPOLOGY_WORKERS**: トポロジーの実行に使用するワーカープロセスの数を設定します。たとえば、これを25に設定すると、すべてのタスクを実行するためにクラスタ全体で25のJavaプロセスが起動することになります。トポロジ内のすべてのコンポーネントにわたって150のparallelismを組み合わせた場合、各ワーカー・プロセスは6つのタスクをスレッドとして実行します。
2. **Config.TOPOLOGY_ACKER_EXECUTORS**: タプルツリーを追跡し、スパウトタプルが完全に処理されたときを検出するエグゼキュータの数を設定します。ackerはStormの信頼性モデルの不可欠な部分であり、詳細は[メッセージ処理の保証](Guaranteeing-message-processing.html)で読むことができます。この変数を設定しないか、またはnullに設定することで、Stormはackerエグゼキュータの数をこのトポロジ用に設定されたワーカーの数に等しく設定します。この変数が0に設定されている場合、StormはSpoutから出てすぐにタプルをackし、信頼性を効果的に無効にします。
3. **Config.TOPOLOGY_MAX_SPOUT_PENDING**: これは、一度に1つのSpoutタスクにpendingされることができるSpoutタプルの最大数を設定します(pendingは、タプルがまだ確認されていないか、またはまだ失敗していないことを意味します)。キューの爆発を防ぐため、この設定を強くお勧めします。
4. **Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS**: Spoutタプルが完全に完了する前に失敗したとみなされる最大時間です。この値のデフォルトは30秒で、ほとんどのトポロジで要件を満たします。 Stormの信頼性モデルの仕組みの詳細については、[メッセージ処理の保証](Guaranteeing-message-processing.html)を参照してください。
5. **Config.TOPOLOGY_SERIALIZATIONS**: タプル内でカスタム型を使用できるように、この設定を使用して、より多くのシリアライザをStormに登録することができます。


### Killing a topology

トポロジを強制終了するには、次のコマンドを実行します:

`storm kill {stormname}`

トポロジをsubmitするときに使用したのと同じ名前を`storm kill`に付けます。

ストームはすぐにトポロジを強制終了しません。代わりに、Spoutがすべてタプルを放出しないようにすべてのスパウトを無効にしてから、StormはConfig.TOPOLOGY_MESSAGE_TIMEOUT_SECS秒待機してからすべてのワーカーを破棄します。これにより、トポロジには、強制終了された時点でに処理していたタプルを完了するのに十分な時間が与えられます。

### Updating a running topology

実行中のトポロジを更新するために、今時点でできることは、トポロジを強制終了して新しいトポロジとして再送信することです。実行中のトポロジーを新しいものと交換する`storm swap`コマンドを実装し、ダウンタイムを最小限に抑え、双方のトポロジがタプルを同時に処理する可能性をなくす機能が計画されています。

### Monitoring topologies

トポロジを監視する最適な場所は、Storm UIを使用することです。Storm UIには、実行中の各トポロジの各コンポーネントのスループットとレイテンシのパフォーマンスに関する、タスクにおけるエラーやきめ細かな統計情報が表示されます。

また、クラスタマシンのワーカーログを見ることもできます。

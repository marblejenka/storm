---
title: Flux
layout: documentation
documentation: true
---

Apache Stormによるストリーム計算をより少ない手間で生成し、デプロイするためのフレームワークです。

## Definition
**flux** |fləks| _noun_

1. The action or process of flowing or flowing out
2. Continuous change
3. In physics, the rate of flow of a fluid, radiant energy, or particles across a given area
4. A substance mixed with a solid to lower its melting point

## Rationale
構成がハードコードされていると、悪いことが起こります。
構成を変更するためにアプリケーションを再コンパイルまたは再パッケージする必要はありません。

## About
Fluxは、Apache Stormトポロジーの定義とデプロイメントを苦労なく行える、デベロッパー志向なフレームワークと一連のユーティリティーです。

あなたは何回このパターンを繰り返すことしてきましたか？:

```java

public static void main(String[] args) throws Exception {
    // logic to determine if we're running locally or not...
    // create necessary config options...
    boolean runLocal = shouldRunLocal();
    if(runLocal){
        LocalCluster cluster = new LocalCluster();
        cluster.submitTopology(name, conf, topology);
    } else {
        StormSubmitter.submitTopology(name, conf, topology);
    }
}
```

こんな感じの方が簡単ではないでしょうか:

```bash
storm jar mytopology.jar org.apache.storm.flux.Flux --local config.yaml
```

とか:

```bash
storm jar mytopology.jar org.apache.storm.flux.Flux --remote config.yaml
```

また、トポロジのグラフを結線するのにJavaコードを使用するため、変更にはトポロジーjarファイルの再コンパイルと再パッケージングが必要であるという点もペインポイントとして取りざたされます。Fluxは、すべてのStormコンポーネントを単一のjarファイルにパッケージ化し、外部テキストファイルを使用してトポロジのレイアウトと設定を定義することで、その苦痛を軽減することを目指しています。

## Features

 * トポロジのコードに設定を埋め込むことなく、Stormトポロジ(StormコアとMicrobatch APIの両方)を簡単に設定およびデプロイできます。
 *  既存のトポロジコードのサポート(下記参照)
 * Storm Core API(Spouts/Bolts)を柔軟なYAML DSLを使用して定義する
 * ほとんどのStormコンポーネント(storm-kafka, storm-hdfs, storm-hbaseなど)のYAML DSLサポート
 * Multi-langコンポーネントのための便利なサポート
 * 設定/環境を簡単に切り替えるための外部プロパティの置換/フィルタリング(Mavenの`${variable.name}を置換するスタイルに似ています)

## Usage

Fluxを使用するには、それを依存関係として追加し、すべてのStormコンポーネントをfat jarにパッケージングし、次にYAMLドキュメントを作成してトポロジを定義します(YAML設定オプションは以下を参照)。

### Building from Source
Fluxを使用する最も簡単な方法は、後述のようにプロジェクトにMaven依存関係を追加することです。

ソースからFluxをビルドし、ユニット/統合テストを実行する場合は、システムに次のものがインストールされている必要があります:

* Python 2.6.x またはそれ以降
* Node.js 0.10.x またはそれ以降

#### Building with unit tests enabled:

```
mvn clean install
```

#### Building with unit tests disabled:
PythonやNode.jsをインストールせずにFluxを構築したい場合は、ユニットテストをスキップするだけです:

```
mvn clean install -DskipTests=true
```

Fluxを使用してリモートクラスタにトポロジをデプロイする場合は、Apache Stormが必要とするため、Pythonをインストールする必要があります。


#### Building with integration tests enabled:

```
mvn clean install -DskipIntegration=false
```


### Packaging with Maven
あなたのStormコンポーネントでFluxを有効にするには、それをStormトポロジのjarに含まれるように依存関係として追加する必要があります。これは、Maven shade plugin（推奨）またMaven assembly plugin（推奨しません）を使用して実行できます。

#### Flux Maven Dependency
Fluxの現在のバージョンは、Maven Centralで次の指定により使用できます:

```xml
<dependency>
    <groupId>org.apache.storm</groupId>
    <artifactId>flux-core</artifactId>
    <version>${storm.version}</version>
</dependency>
```

#### Creating a Flux-Enabled Topology JAR
以下の例は、Maven shadeプラグインを使用したFluxの使用法を示しています:

 ```xml
<!-- include Flux and user dependencies in the shaded jar -->
<dependencies>
    <!-- Flux include -->
    <dependency>
        <groupId>org.apache.storm</groupId>
        <artifactId>flux-core</artifactId>
        <version>${storm.version}</version>
    </dependency>

    <!-- add user dependencies here... -->

</dependencies>
<!-- create a fat jar that includes all dependencies -->
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>1.4</version>
            <configuration>
                <createDependencyReducedPom>true</createDependencyReducedPom>
            </configuration>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <transformers>
                            <transformer
                                    implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                            <transformer
                                    implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                <mainClass>org.apache.storm.flux.Flux</mainClass>
                            </transformer>
                        </transformers>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
 ```

### Deploying and Running a Flux Topology
トポロジのコンポーネントがFluxの依存関係とともいパッケージ化されたら、`storm jar`コマンドを使用してローカルまたはリモートの異なるトポロジを実行できます。たとえば、fat jarの名前が `myTopology-0.1.0-SNAPSHOT.jar`である場合、以下のコマンドを使用してローカルで実行できます:

```bash
storm jar myTopology-0.1.0-SNAPSHOT.jar org.apache.storm.flux.Flux --local my_config.yaml
```

### Command line options
```
usage: storm jar <my_topology_uber_jar.jar> org.apache.storm.flux.Flux
             [options] <topology-config.yaml>
 -d,--dry-run                 トポロジを実行またはデプロイしません。トポロジに関する情報を構築、検証、表示するだけです。
 -e,--env-filter              環境変数の置換を実行します。`${ENV-[NAME]}`で指定された置換キーは、対応する`NAME`の環境変数に置き換えられます
 -f,--filter <file>           プロパティ置換を実行します。指定されたファイルをプロパティのソースとして使用し、
                              $[property name]}で識別されるキーをプロパティファイルで定義された値に置き換えます。
 -i,--inactive                トポロジをデプロイしますが、有効化しません。
 -l,--local                   ローカルモードでトポロジを実行します。
 -n,--no-splash               スプラッシュ画面の表示を抑止します。
 -q,--no-detail               トポロジの詳細の表示を抑止します。
 -r,--remote                  トポロジをリモートのクラスタにデプロイします。
 -R,--resource                指定されたパスを、ファイルではなくクラスパスリソースとして扱います。
 -s,--sleep <ms>              ローカルで実行しているときに、トポロジを強制終了してローカルクラスタをシャットダウンするまでのスリープ時間(ms)。
 -z,--zookeeper <host:port>   ローカルモードで実行する際に、インプロセスZooKeeperの代わりに、
                              指定した<host>:<port>のZooKeeperを使用します。(Stormの0.9.3以降が必要です)
```

**注意:** Fluxは`storm`コマンドのコマンドラインスイッチの衝突を回避しようとしており、他のコマンドラインスイッチを`storm`コマンドに渡すことができます。

たとえば、`storm`コマンドスイッチ`-c`を使用して、トポロジの設定プロパティを上書きすることができます。
次のコマンド例はFluxを実行し、`nimbus.seeds`設定を上書きします:

```bash
storm jar myTopology-0.1.0-SNAPSHOT.jar org.apache.storm.flux.Flux --remote my_config.yaml -c 'nimbus.seeds=["localhost"]'
```

### Sample output
```
███████╗██╗     ██╗   ██╗██╗  ██╗
██╔════╝██║     ██║   ██║╚██╗██╔╝
█████╗  ██║     ██║   ██║ ╚███╔╝
██╔══╝  ██║     ██║   ██║ ██╔██╗
██║     ███████╗╚██████╔╝██╔╝ ██╗
╚═╝     ╚══════╝ ╚═════╝ ╚═╝  ╚═╝
+-         Apache Storm        -+
+-  data FLow User eXperience  -+
Version: 0.3.0
Parsing file: /Users/hsimpson/Projects/donut_domination/storm/shell_test.yaml
---------- TOPOLOGY DETAILS ----------
Name: shell-topology
--------------- SPOUTS ---------------
sentence-spout[1](org.apache.storm.flux.spouts.GenericShellSpout)
---------------- BOLTS ---------------
splitsentence[1](org.apache.storm.flux.bolts.GenericShellBolt)
log[1](org.apache.storm.flux.wrappers.bolts.LogInfoBolt)
count[1](org.apache.storm.testing.TestWordCounter)
--------------- STREAMS ---------------
sentence-spout --SHUFFLE--> splitsentence
splitsentence --FIELDS--> count
count --SHUFFLE--> log
--------------------------------------
Submitting topology: 'shell-topology' to remote cluster...
```

## YAML Configuration
Fluxのトポロジは、トポロジを記述するYAMLファイルで定義されます。
Fluxのトポロジの定義は、次のもので構成されます:

  1. トポロジ名
  2. トポロジの"コンポーネント"のリスト(環境内で使用可能になる名前付けられたJavaオブジェクト)
  3. **いずれかの** (DSLによるトポロジ定義):
      * Spoutのリストで、それぞれが一意のIDで識別されているもの
      * Boltのリストで、それぞれが一意のIDで識別されているもの
      * SpoutとBoltの間のタプルの流れを表す"ストリーム"オブジェクトのリスト
  4. **オプショナルな** (`org.apache.storm.generated.StormTopology`のインスタンスを生成できるJVMクラス
      * `topologySource`の定義

例として、YAML DSLを使用したワードカウントをおこなうトポロジの簡単な定義を次に示します:

```yaml
name: "yaml-topology"
config:
  topology.workers: 1

# spout definitions
spouts:
  - id: "spout-1"
    className: "org.apache.storm.testing.TestWordSpout"
    parallelism: 1

# bolt definitions
bolts:
  - id: "bolt-1"
    className: "org.apache.storm.testing.TestWordCounter"
    parallelism: 1
  - id: "bolt-2"
    className: "org.apache.storm.flux.wrappers.bolts.LogInfoBolt"
    parallelism: 1

#stream definitions
streams:
  - name: "spout-1 --> bolt-1" # name isn't used (placeholder for logging, UI, etc.)
    from: "spout-1"
    to: "bolt-1"
    grouping:
      type: FIELDS
      args: ["word"]

  - name: "bolt-1 --> bolt2"
    from: "bolt-1"
    to: "bolt-2"
    grouping:
      type: SHUFFLE


```
## Property Substitution/Filtering
開発において、開発者は開発環境と本番環境の間での展開の切り替えなど、設定を簡単に切り替えたくなることが普通です。
これは、別のYAML設定ファイルを使用することで実現できますが、その方法だと、特にStormトポロジが変更されず、ホスト名、ポート、およびparallelismについてのパラメータなどの設定を変更する場合に、不要な重複が発生します。

この場合、Fluxは`.properties`ファイルに値を2つの外部化し、`.yaml`ファイルがパースされる前にそれらを置換するためのプロパティフィルタリングを提供します。

プロパティフィルタリングを有効にするには、`--filter`コマンドラインオプションを使用し、`.properties`ファイルを指定します。たとえば、次のようにfluxを呼び出したとします:

```bash
storm jar myTopology-0.1.0-SNAPSHOT.jar org.apache.storm.flux.Flux --local my_config.yaml --filter dev.properties
```
以下の`dev.properties`ファイルをがあるとして:

```properties
kafka.zookeeper.hosts: localhost:2181
```

`${}`構文を使うと、`.yaml`ファイルのキーからこれらのプロパティを参照することができます：

```yaml
  - id: "zkHosts"
    className: "org.apache.storm.kafka.ZkHosts"
    constructorArgs:
      - "${kafka.zookeeper.hosts}"
```

この場合、FluxはYAMLの内容をパースする前に`${kafka.zookeeper.hosts}`を`localhost:2181`に置き換えます。

### Environment Variable Substitution/Filtering
Fluxでは、環境変数の置換も可能です。
たとえば、`ZK_HOSTS`という名前の環境変数が定義されている場合、FluxのYAMLファイルでは次の構文で参照することができます:

```
${ENV-ZK_HOSTS}
```

## Components
コンポーネントは基本的に名前付けられたオブジェクトインスタンスで、SpoutとBoltの構成におけるオプションとして使用できます。 
あなたがSpringフレームワークに精通しているなら、コンポーネントはSpring Beanとほぼ類似していることがわかるでしょう。

すべてのコンポーネントは、最小では、一意の識別子(String)とクラス名(String)によって識別されます。
たとえば、以下の例では`org.apache.storm.kafka.StringScheme`クラスのインスタンスをキー`"stringScheme"`で参照として利用できます。
これは、`org.apache.storm.kafka.StringScheme`がデフォルトコンストラクタを持っていることを前提としています。

```yaml
components:
  - id: "stringScheme"
    className: "org.apache.storm.kafka.StringScheme"
```

### Contructor Arguments, References, Properties and Configuration Methods

####Constructor Arguments
クラスのコンストラクタへの引数は、コンポーネントに`contructorArgs`要素を追加することで設定できます。
`constructorArgs`は、クラスのコンストラクタに渡されるオブジェクトのリストです。次の例では、単一の文字列を引数として取るコンストラクタを呼び出してオブジェクトを作成します:

```yaml
  - id: "zkHosts"
    className: "org.apache.storm.kafka.ZkHosts"
    constructorArgs:
      - "localhost:2181"
```

####References
各コンポーネントのインスタンスは、他のコンポーネントによって使用/再利用されることを可能にする一意のIDによって識別されます。
既存のコンポーネントを参照するには、`ref`タグを使ってコンポーネントのidを指定します。

次の例では、id`"stringScheme"`を持つコンポーネントが生成され、
後で別のコンポーネントのコンストラクタへの引数として参照できまます:

```yaml
components:
  - id: "stringScheme"
    className: "org.apache.storm.kafka.StringScheme"

  - id: "stringMultiScheme"
    className: "org.apache.storm.spout.SchemeAsMultiScheme"
    constructorArgs:
      - ref: "stringScheme" # component with id "stringScheme" must be declared above.
```
**N.B.:** 参照は、それらが指し示すオブジェクトが宣言された後(下)でしか使用できません。

####Properties
異なる引数を持つコンストラクタを呼び出すことに加えて、
Fluxでは、`public`として宣言されたJavaBean風のsetterメソッドとフィールドを使用してコンポーネントを構成することもできます:

```yaml
  - id: "spoutConfig"
    className: "org.apache.storm.kafka.SpoutConfig"
    constructorArgs:
      # brokerHosts
      - ref: "zkHosts"
      # topic
      - "myKafkaTopic"
      # zkRoot
      - "/kafkaSpout"
      # id
      - "myId"
    properties:
      - name: "ignoreZkOffsets"
        value: true
      - name: "scheme"
        ref: "stringMultiScheme"
```

上記の例では、`properties`宣言により、Fluxはシグネチャ`setForceFromStart(boolean b)`を持つ`SpoutConfig`内のパブリックメソッドを探し、それを呼び出そうとします。setterメソッドが見つからない場合、Fluxは`ignoreZkOffsets`という名前のパブリックインスタンス変数を探し、その値を設定しようとします。

参照はプロパティ値として使用することもできます。

####Configuration Methods
概念的には、Configuration MethodはPropertiesやConstructor引数に似ています -- オブジェクトの作成後に任意のメソッドを呼び出すことができます。
Configuration Methodsは、JavaBeanメソッドを公開しないクラスや、オブジェクトを完全に構成できるコンストラクタを持つクラスを操作する場合に便利です。
一般的な例だと、設定/合成にBuilderパターンを使用するクラスが含まれます。

次のYAMLの例は、Boltを生成し、いくつかのメソッドを呼び出すことによってそれを設定します:

```yaml
bolts:
  - id: "bolt-1"
    className: "org.apache.storm.flux.test.TestBolt"
    parallelism: 1
    configMethods:
      - name: "withFoo"
        args:
          - "foo"
      - name: "withBar"
        args:
          - "bar"
      - name: "withFooBar"
        args:
          - "foo"
          - "bar"
```

対応するメソッドのシグネチャは次のとおりです:

```java
    public void withFoo(String foo);
    public void withBar(String bar);
    public void withFooBar(String foo, String bar);
```

Configuration Methodに渡される引数は、コンストラクタ引数と同じように動作し、Referenceもサポートします。

### Using Java `enum`s in Contructor Arguments, References, Properties and Configuration Methods
単に `enum`の名前を参照するだけで、Flux YAMLファイルの引数としてJavaの`enum`値を簡単に使うことができます。

たとえば、[Storm's HDFS module]()には、以下の `enum`定義が含まれています（簡略さために単純化されています）。

```java
public static enum Units {
    KB, MB, GB, TB
}
```

そして、`org.apache.storm.hdfs.bolt.rotation.FileSizeRotationPolicy`クラスは次のコンストラクタを持っています:

```java
public FileSizeRotationPolicy(float count, Units units)

```
次のFlux`component`定義を使用してコンストラクタを呼び出すことができます:

```yaml
  - id: "rotationPolicy"
    className: "org.apache.storm.hdfs.bolt.rotation.FileSizeRotationPolicy"
    constructorArgs:
      - 5.0
      - MB
```

上記の定義は、次のJavaコードと機能的に同等です:

```java
// rotate files when they reach 5MB
FileRotationPolicy rotationPolicy = new FileSizeRotationPolicy(5.0f, Units.MB);
```

## Topology Config
`config`セクションは`org.apache.storm.Config`クラスのインスタンスとして`org.apache.storm.StormSubmitter`に渡されるStormトポロジ設定パラメータの対応付けです:

```yaml
config:
  topology.workers: 4
  topology.max.spout.pending: 1000
  topology.message.timeout.secs: 30
```

# Existing Topologies
既存のStormトポロジーを使用している場合でも、Fluxを使用してデプロイ/実行/テストすることができます。
この機能を使用すると、既存のトポロジクラスについてFluxのコンストラクタの引数、参照、プロパティ、およびトポロジのコンフィグレーションの宣言を利用できます。

既存のトポロジクラスを使用する最も簡単な方法は、次のシグネチャのいずれかを使って`getTopology()`インスタンスメソッドを定義することです:

```java
public StormTopology getTopology(Map<String, Object> config)
```
あるいは:

```java
public StormTopology getTopology(Config config)
```

次に、以下のYAMLを使用してトポロジを構成できます:

```yaml
name: "existing-topology"
topologySource:
  className: "org.apache.storm.flux.test.SimpleTopology"
```

トポロジのソースとして使用したいクラスが異なるメソッド名(例えば`getTopology`ないもの)である場合は、それを上書きすることができます:

```yaml
name: "existing-topology"
topologySource:
  className: "org.apache.storm.flux.test.SimpleTopology"
  methodName: "getTopologyWithDifferentMethodName"
```

__N.B.:__ 指定されたメソッドは`java.util.Map <String、Object>`または `org.apache.storm.Config`型の単一の引数を受け入れ、
`org.apache.storm.generated.StormTopology`オブジェクトを返す必要があります。

# YAML DSL
## Spouts and Bolts
SpoutとBoltは、YAML設定のそれぞれのセクションで設定されます。
SpoutとBoltの定義は、トポロジのデプロイ時にコンポーネントの並列性を設定する`parallelism`パラメータを追加する`component`定義の拡張です。

SpoutとBoltの定義は`component`を拡張するので、コンストラクタ引数、参照、プロパティもサポートされます。

シェルSpoutの例:

```yaml
spouts:
  - id: "sentence-spout"
    className: "org.apache.storm.flux.spouts.GenericShellSpout"
    # shell spout constructor takes 2 arguments: String[], String[]
    constructorArgs:
      # command line
      - ["node", "randomsentence.js"]
      # output fields
      - ["word"]
    parallelism: 1
```

Kafka Spoutの例:

```yaml
components:
  - id: "stringScheme"
    className: "org.apache.storm.kafka.StringScheme"

  - id: "stringMultiScheme"
    className: "org.apache.storm.spout.SchemeAsMultiScheme"
    constructorArgs:
      - ref: "stringScheme"

  - id: "zkHosts"
    className: "org.apache.storm.kafka.ZkHosts"
    constructorArgs:
      - "localhost:2181"

# Alternative kafka config
#  - id: "kafkaConfig"
#    className: "org.apache.storm.kafka.KafkaConfig"
#    constructorArgs:
#      # brokerHosts
#      - ref: "zkHosts"
#      # topic
#      - "myKafkaTopic"
#      # clientId (optional)
#      - "myKafkaClientId"

  - id: "spoutConfig"
    className: "org.apache.storm.kafka.SpoutConfig"
    constructorArgs:
      # brokerHosts
      - ref: "zkHosts"
      # topic
      - "myKafkaTopic"
      # zkRoot
      - "/kafkaSpout"
      # id
      - "myId"
    properties:
      - name: "ignoreZkOffsets"
        value: true
      - name: "scheme"
        ref: "stringMultiScheme"

config:
  topology.workers: 1

# spout definitions
spouts:
  - id: "kafka-spout"
    className: "org.apache.storm.kafka.KafkaSpout"
    constructorArgs:
      - ref: "spoutConfig"

```

Boltの例:

```yaml
# bolt definitions
bolts:
  - id: "splitsentence"
    className: "org.apache.storm.flux.bolts.GenericShellBolt"
    constructorArgs:
      # command line
      - ["python", "splitsentence.py"]
      # output fields
      - ["word"]
    parallelism: 1
    # ...

  - id: "log"
    className: "org.apache.storm.flux.wrappers.bolts.LogInfoBolt"
    parallelism: 1
    # ...

  - id: "count"
    className: "org.apache.storm.testing.TestWordCounter"
    parallelism: 1
    # ...
```
## Streams and Stream Groupings
Fluxのストリームは、関連付けられたグループ化定義とともに、トポロジ内のSpoutとBoltの間の接続（グラフのエッジ、データフローなど）のリストとして表されます。

ストリームの定義には、次のプロパティがあります:

**`name`:** 接続の名前（オプション、現在は未使用）

**`from`:** ソース（パブリッシャ）であるSpoutまたはBoltの`id`

**`to`:** 行き先（サブスクライバ）であるSpoutまたはBoltの`id`

**`grouping`:** ストリームのストリームグループ化定義

グループ化定義には、次のプロパティがあります。:

**`type`:** グループ化のタイプ。`ALL`,`CUSTOM`,`DIRECT`,`SHUFFLE`,`LOCAL_OR_SHUFFLE`,`FIELDS`,`GLOBAL`,または`NONE`のいずれか。

**`streamId`:** StormのストリームID(省略可能。未指定の場合、デフォルトのストリームを使用します)

**`args`:** `FIELDS`グループの場合、フィールド名のリスト。

**`customClass`** `CUSTOM`グルーピングの場合、カスタムグルーピングクラスインスタンスの定義

以下の`streams`定義の例は、以下のような結線でトポロジーを設定します:

```
    kafka-spout --> splitsentence --> count --> log
```


```yaml
#stream definitions
# stream definitions define connections between spouts and bolts.
# note that such connections can be cyclical
# custom stream groupings are also supported

streams:
  - name: "kafka --> split" # name isn't used (placeholder for logging, UI, etc.)
    from: "kafka-spout"
    to: "splitsentence"
    grouping:
      type: SHUFFLE

  - name: "split --> count"
    from: "splitsentence"
    to: "count"
    grouping:
      type: FIELDS
      args: ["word"]

  - name: "count --> log"
    from: "count"
    to: "log"
    grouping:
      type: SHUFFLE
```

### Custom Stream Groupings
カスタムストリームグループを定義するには、グループ化タイプを`CUSTOM`に設定し、カスタムクラスをインスタンス化する方法をFluxに指示する`customClass`パラメータを定義します。
`customClass`定義は`component`を拡張するので、コンストラクタの引数、参照、プロパティもサポートします。

次の例は、`org.apache.storm.testing.NGrouping`カスタムストリームグループ化クラスのインスタンスを持つStreamを作成します。

```yaml
  - name: "bolt-1 --> bolt2"
    from: "bolt-1"
    to: "bolt-2"
    grouping:
      type: CUSTOM
      customClass:
        className: "org.apache.storm.testing.NGrouping"
        constructorArgs:
          - 1
```

## Includes and Overrides
Fluxでは、他のYAMLファイルの内容を含めることができ、同じファイルに定義されているかのように扱うことができます。
Includeは、ファイルまたはクラスパスリソースのいずれかです。

Includeはマップのリストとして指定されます：

```yaml
includes:
  - resource: false
    file: "src/test/resources/configs/shell_test.yaml"
    override: false
```

`resource`プロパティが`true`に設定されている場合、Includeはクラスパスリソースとして`file`属性の値からロードされます。
そうでない場合、それは通常のファイルとして扱われます。

`override`プロパティは、現在のファイルに定義されている値をインクルードする方法を制御します。
`override`が`true`に設定されている場合、Includeされたファイルの値は、解析中の現在のファイルの値を置き換えます。
`override`が`false`に設定されていると、解析中の現在のファイルの値が優先され、パーサはそれらを置き換えることをしません。

**N.B.:** Includeは今のところ再帰的ではありません。IncludeされているファイルからのIncludeは無視されます。


## Basic Word Count Example

この例では、JavaScriptで実装されたSpout、Pythonで実装されたボルト、Javaで実装されたボルトを使用しています

トポロジのYAMLによる設定:

```yaml
---
name: "shell-topology"
config:
  topology.workers: 1

# spout definitions
spouts:
  - id: "sentence-spout"
    className: "org.apache.storm.flux.spouts.GenericShellSpout"
    # shell spout constructor takes 2 arguments: String[], String[]
    constructorArgs:
      # command line
      - ["node", "randomsentence.js"]
      # output fields
      - ["word"]
    parallelism: 1

# bolt definitions
bolts:
  - id: "splitsentence"
    className: "org.apache.storm.flux.bolts.GenericShellBolt"
    constructorArgs:
      # command line
      - ["python", "splitsentence.py"]
      # output fields
      - ["word"]
    parallelism: 1

  - id: "log"
    className: "org.apache.storm.flux.wrappers.bolts.LogInfoBolt"
    parallelism: 1

  - id: "count"
    className: "org.apache.storm.testing.TestWordCounter"
    parallelism: 1

#stream definitions
# stream definitions define connections between spouts and bolts.
# note that such connections can be cyclical
# custom stream groupings are also supported

streams:
  - name: "spout --> split" # name isn't used (placeholder for logging, UI, etc.)
    from: "sentence-spout"
    to: "splitsentence"
    grouping:
      type: SHUFFLE

  - name: "split --> count"
    from: "splitsentence"
    to: "count"
    grouping:
      type: FIELDS
      args: ["word"]

  - name: "count --> log"
    from: "count"
    to: "log"
    grouping:
      type: SHUFFLE
```


## Micro-Batching (Trident) API Support
現在、Flux YAML DSLはCore Storm APIのみをサポートしていますが、Stormのマイクロバッチ処理APIのサポートが計画されています。

FluentをTridentトポロジで使用するには、トポロジにgetterメソッドを定義し、YAMLコンフィグレーションでそれを参照します:

```yaml
name: "my-trident-topology"

config:
  topology.workers: 1

topologySource:
  className: "org.apache.storm.flux.test.TridentTopologySource"
  # Flux will look for "getTopology", this will override that.
  methodName: "getTopologyWithDifferentMethodName"
```

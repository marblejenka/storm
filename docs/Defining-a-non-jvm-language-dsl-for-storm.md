---
title: Defining a Non-JVM DSL for Storm
layout: documentation
documentation: true
---
Stormのための非JVM DSLの作成方法を学ぶための適切な場所は、[storm-core/src/storm.thrift]({{page.git-blob-base}}/storm-core/src/storm.thrift)です。StormトポロジはThrift構造にすぎず、NimbusはThriftデーモンであるため、任意の言語でトポロジを作成して送信できます。

スパウトとボルト用のThrift構造体を作成するとき、スパウトまたはボルトのコードはComponentObject構造体で定義されます:

```
union ComponentObject {
  1: binary serialized_java;
  2: ShellComponent shell;
  3: JavaObject java_object;
}
```

Python DSLの場合は、"2"と "3"を使いたいでしょう。ShellComponentでは、そのコンポーネント（Pythonコードなど）を実行するためのスクリプトを指定することができます。JavaObjectでは、コンポーネントのネイティブJavaスパウトとボルトを指定できます（また、スパムやボルトを作成するためにリフレクションを使用します）。
For a Python DSL, you would want to make use of "2" and "3". ShellComponent lets you specify a script to run that compone

トポロジのsubmitに役立つ"storm shell"コマンドがあります。その使い方は次のとおりです:

```
storm shell resources/ python topology.py arg1 arg2
```

storm shellはリソースをjarファイルにパッケージ化し、jarをNimbusにアップロードして、topology.pyスクリプトを次のように呼び出します:

```
python topology.py arg1 arg2 {nimbus-host} {nimbus-port} {uploaded-jar-location}
```

次に、Thrift APIを使用してNimbusに接続し、{uploaded-jar-location}をsubmitTopologyメソッドに渡してトポロジを送信します。参考までにsubmitTopologyの定義を以下に示します:

```java
void submitTopology(1: string name, 2: string uploadedJarLocation, 3: string jsonConf, 4: StormTopology topology) throws (1: AlreadyAliveException e, 2: InvalidTopologyException ite);
```

最後に、非JVM DSLで重要なことの1つは、トポロジ全体を1つのファイル（ボルト、スパウト、およびトポロジの定義）に簡単に定義できるようにすることです。

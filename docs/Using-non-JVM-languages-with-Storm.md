---
title: Using non JVM languages with Storm
layout: documentation
---
- 2つの部分: トポロジの生成と他の言語によるSpoutとBoltの実装
- トポロジは単にthriftの構造であるため、別の言語でトポロジを作成するのは簡単です(storm.thriftへのリンク)
- SpoutとBoltを別の言語で実装することを"multilang components"または"shelling"といいます
   - プロトコルの仕様は次のとおりです: [Multilang protocol](Multilang-protocol.html)
   - thrift構造では、プログラムとスクリプト(例えば、PythonとあなたのBoltを実装するファイル)として明示的にmultilangコンポーネントを定義することができます
   - Javaでは、ShellBoltまたはShellSpoutをオーバーライドして、multilangコンポーネントを作成します:
       - 出力フィールド宣言はthriftの構造内で発生するので、Javaでは次のようなmultilangコンポーネントを作成します:
            - javaのフィールドを宣言し、shellboltのコンストラクタで指定して他の言語のコードを処理する
   - multilangはstdin/stdout上のjsonメッセージを使用してサブプロセスと通信します
   - Stormは、Ruby、Python、およびプロトコルを実装するいい感じのアダプタが付属しています。以下がPythonの例です
      - pythonはemit、anchor、ack、ロギングをサポートしています
- "storm shell"コマンドを使うとjarを構築し、nimbusに簡単にアップロードできます
  - jarをビルドし、それをアップロードする
  - nimbusのホスト/ポートとjarファイルのIDでプログラムを呼び出します

## Notes on implementing a DSL in a non-JVM language

開始すべき位置はsrc/storm.thriftです。StormトポロジはThrift構造にすぎず、NimbusはThriftデーモンであるため、任意の言語でトポロジを作成してsubmitすることができます。

SpuotとBolt用のThrift構造体を作成すると、SpoutまたはBoltのコードはComponentObject構造体で指定されます:

```
union ComponentObject {
  1: binary serialized_java;
  2: ShellComponent shell;
  3: JavaObject java_object;
}
```

非JVM DSLの場合、"2"と"3"を使いたいと思うでしょう。ShellComponentでは、そのコンポーネント(Pythonコードなど)を実行するためのスクリプトを指定することができます。JavaObjectでは、コンポーネントのネイティブJavaスパウトとボルトを指定できます(StormはSpoutやBoltを作成するためにリフレクションを使用します)。

トポロジのsubmitに役立つ"storm shell"コマンドがあります。その使い方は次のとおりです:

```
storm shell resources/ python topology.py arg1 arg2
```

storm shellはresources/をjarファイルにパッケージ化し、jarをNimbusにアップロードして、topology.pyスクリプトを次のように呼び出します:

```
python topology.py arg1 arg2 {nimbus-host} {nimbus-port} {uploaded-jar-location}
```

次に、Thrift APIを使用してNimbusに接続し、{uploaded-jar-location}をsubmitTopologyメソッドに渡してトポロジをsubmitします。参考までにsubmitTopologyの定義を以下に示します:

```
void submitTopology(1: string name, 2: string uploadedJarLocation, 3: string jsonConf, 4: StormTopology topology)
    throws (1: AlreadyAliveException e, 2: InvalidTopologyException ite);
```

---
layout: documentation
---
タプルは、あらゆるタイプのオブジェクトで構成できます。Stormは分散システムなので、オブジェクトをタスク間で渡したときにそれらをシリアリライズおよびデシリアライズする方法を知る必要があります。デフォルトでは、Stormはプリミティブ型、文字列、バイト配列、ArrayList、HashMap、HashSet、およびClojureコレクション型をシリアル化できます。タプルで別の型を使用する場合は、カスタムシリアライザを登録する必要があります。

### Dynamic typing

タプルのフィールドの型宣言はありません。フィールドにオブジェクトを入れておけば、Stormは結果を動的に取り出します。シリアライゼーションのインターフェイスについて見る前に、Stormでタプルを動的型付けしている理由を理解しておきましょう。

タプルフィールドを静的型付けとすると、StormのAPIには多大な複雑さが加わります。たとえば、Hadoopはキーやバリューを静的に型付けしますが、ユーザー側で膨大な量のアノテーションを必要とします。HadoopのAPIは使用する負担であり、"型安全であること"には価値がありません。動的型付けはシンプルで使い方が簡単です。

それより、Stormのタプルを任意の方法で静的に型付けすることはできません。Boltが複数のストリームをサブスクライブしているとします。これらすべてのストリームのタプルは、フィールド間で異なるタイプを持つことがあります。Boltが`execute`で`Tuple`を受け取る際に、そのタプルはどんなストリームから来ていてもかまわないので、いろいろな型の組み合わせを持っている可能性があります。Boltがサブスクライブするすべてのタプルストリームに対して異なるメソッドを宣言できるreflection的な魔法があるかもしれませんが、Stormは動的型付けによる単純で直接的なアプローチを選択しています。

最後に、動的型付けを使用するもう一つの理由は、ClojureやJRubyのような動的型付けを行う言語からStormを簡単に使用できることです。

### Custom serialization

Stormのカスタムシリアライゼーションを定義するためのAPIを紹介しましょう。カスタムシリアライゼーションを作成するには、シリアライザを実装し、シリアライザをStormに登録する2つのステップが必要です。

#### Creating a serializer

カスタムシリアライザは、[ISerialization](javadocs/backtype/storm/serialization/ISerialization.html)インタフェースを実装しています。実装では、型をバイナリ形式にシリアライズする方法およびデシリアライズする方法を指定します。

インターフェイスは次のようになります:

```java
public interface ISerialization<T> {
    public boolean accept(Class c);
    public void serialize(T object, DataOutputStream stream) throws IOException;
    public T deserialize(DataInputStream stream) throws IOException;
}
```

Stormは、このシリアライザでタイプをシリアライズできるかどうかを判断するために`accept`メソッドを使います。Stormのタプルは動的に型指定されるため、Stormは実行時にどのシリアライザを使用するかを決定します。

`serialize`はオブジェクトをバイナリ形式で出力ストリームに書き出します。フィールドは、後でデシリアライズできるような方法で記述する必要があります。たとえば、オブジェクトのリストを書き出す場合は、リストのサイズを最初に書き出して、デシリアライズする要素の数を知る必要があります。

`deserialize`は直列化されたオブジェクトをストリームから読み込んで返します。
`deserialize` reads the serialized object off of the stream and returns it.

[SerializationFactory](https://github.com/apache/incubator-storm/blob/0.5.4/src/jvm/backtype/storm/serialization/SerializationFactory.java)のソースで、シリアライズ実装の例を見ることができます。


#### Registering a serializer

シリアライザを作成したら、Stormにそれが存在していることを伝える必要があります。これはStormの設定によって行われます(Stormでの設定の仕組みについては、[Concepts](Concepts.html)を参照してください)。シリアライズを登録するには、トポロジをsubmitするときに指定した設定か、クラスタ全体のstorm.yamlファイルを使用します。

シリアライザの登録は、Config.TOPOLOGY_SERIALIZATIONS設定を介して行われ、それは単なるシリアライゼーションクラス名のリストです。

Stormは、シリアライザをトポロジ設定に登録するためのヘルパーを提供します。[Config](javadocs/backtype/storm/Config.html)クラスには、`addSerialization`というメソッドがあり、シリアライザのクラスをconfigに追加します。

Config.TOPOLOGY_SKIP_MISSING_SERIALIZATIONSという高度な設定があります。これをtrueに設定すると、Stormは登録されているクラスパス上でコードを利用できないシリアライザは無視されます。それ以外の場合、Stormはシリアライザが見つからない場合にエラーをスローします。これは、クラスタ上で異なるシリアライゼーションを持つ多数のトポロジを実行するが、 `storm.yaml`ファイル内ですべてのトポロジについてのすべてのシリアライゼーションを宣言したい場合に便利です。


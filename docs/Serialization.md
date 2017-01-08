---
title: Serialization
layout: documentation
documentation: true
---
このページでは、Stormのシリアライゼーションシステムがバージョン0.6.0以降でどのように動作するか説明します。Stormは、0.6.0以前では、[Serialization (prior to 0.6.0)](Serialization-\(prior-to-0.6.0\).html)に記載のある異なるシリアライゼーションシステムを使用していました。

タプルは、あらゆるタイプのオブジェクトで構成できます。Stormは分散システムなので、オブジェクトをタスク間で渡したときにそれらをシリアリライズおよびデシリアライズする方法を知る必要があります。

Stormはシリアライゼーションに[Kryo](https://github.com/EsotericSoftware/kryo)を使用しています。Kryoは、小さな結果を生成する柔軟で高速なシリアライゼーションライブラリです。

デフォルトでは、Stormはプリミティブ型、文字列、バイト配列、ArrayList、HashMap、HashSet、およびClojureコレクション型をシリアル化できます。タプルで別の型を使用する場合は、カスタムシリアライザを登録する必要があります。

### Dynamic typing

タプルのフィールドの型宣言はありません。フィールドにオブジェクトを入れておけば、Stormは結果を動的に取り出します。シリアライゼーションのインターフェイスについて見る前に、Stormでタプルを動的型付けしている理由を理解しておきましょう。

タプルフィールドを静的型付けとすると、StormのAPIには多大な複雑さが加わります。たとえば、Hadoopはキーやバリューを静的に型付けしますが、ユーザー側で膨大な量のアノテーションを必要とします。HadoopのAPIは使用する負担であり、"型安全であること"には価値がありません。動的型付けはシンプルで使い方が簡単です。

それより、Stormのタプルを任意の方法で静的に型付けすることはできません。Boltが複数のストリームをサブスクライブしているとします。これらすべてのストリームのタプルは、フィールド間で異なるタイプを持つことがあります。Boltが`execute`で`Tuple`を受け取る際に、そのタプルはどんなストリームから来ていてもかまわないので、いろいろな型の組み合わせを持っている可能性があります。Boltがサブスクライブするすべてのタプルストリームに対して異なるメソッドを宣言できるreflection的な魔法があるかもしれませんが、Stormは動的型付けによる単純で直接的なアプローチを選択しています。

最後に、動的型付けを使用するもう一つの理由は、ClojureやJRubyのような動的型付けを行う言語からStormを簡単に使用できることです。

### Custom serialization

前述のように、StormはシリアライゼーションにKryoを使用します。カスタムシリアライザを実装するには、新しいシリアライザをKryoに登録する必要があります。カスタムシリアライゼーションをどのように処理するかについて理解するために、[Kryoのホームページ](https://github.com/EsotericSoftware/kryo)を読むことを強く推奨します。

カスタムシリアライザの追加は、トポロジ設定の"topology.kryo.register"プロパティで行います。プロパティは登録対象ののリストを取り、次の2つの形式のいずれかを取ることができます:

1. クラス名による登録。この場合、StormはKryoの`FieldsSerializer`を使ってクラスをシリアライズします。これはクラスにとって最適かもしれないし、そうでないかもしれません -- 詳細についてはKryoのドキュメントを見てください。
2. 登録するクラス名と[com.esotericsoftware.kryo.Serializer](https://github.com/EsotericSoftware/kryo/blob/master/src/com/esotericsoftware/kryo/Serializer.java)の実装の対応付け

例を見てみましょう。

```
topology.kryo.register:
  - com.mycompany.CustomType1
  - com.mycompany.CustomType2: com.mycompany.serializer.CustomType2Serializer
  - com.mycompany.CustomType3
```

`com.mycompany.CustomType1`と`com.mycompany.CustomType3`は`FieldsSerializer`を使いますが、`com.mycompany.CustomType2`は `com.mycompany.serializer.CustomType2Serializer`を使ってシリアライゼーションします。

Stormは、シリアライザをトポロジ設定に登録するためのヘルパーを提供します。[Config](javadocs/org/apache/storm/Config.html)クラスには、registerSerializationというメソッドがあり、これを使用してレジストリを設定に追加します。

`Config.TOPOLOGY_SKIP_MISSING_KRYO_REGISTRATIONS`という高度な設定があります。これをtrueに設定すると、Stormは登録されているクラスパス上でコードを利用できないシリアライザは無視されます。それ以外の場合、Stormはシリアライザが見つからない場合にエラーをスローします。これは、クラスタ上で異なるシリアライゼーションを持つ多数のトポロジを実行するが、 `storm.yaml`ファイル内ですべてのトポロジについてのすべてのシリアライゼーションを宣言したい場合に便利です。

### Java serialization

Stormがシリアライザが登録されていない型に遭遇した場合、可能であればJavaのシリアライゼーションを使用します。Javaのシリアライゼーションでオブジェクトをシリアライズできない場合、Stormはエラーをスローします。

Javaのシリアライゼーションは、CPUコストとシリアライズされたオブジェクトのサイズの両面で非常に重いことに注意してください。トポロジをプロダクション環境に置くときは、カスタムシリアライザを登録することを強くお勧めします。Javaのシリアライゼーションを使えるようにしているのは、新しいトポロジのプロトタイプ作成を容易にするためです。

`Config.TOPOLOGY_FALL_BACK_ON_JAVA_SERIALIZATION`設定をfalseに設定することによって、Javaのシリアライゼーションにフォールバックする動作をオフにすることができます。

### Component-specific serialization registrations

Storm 0.7.0では、コンポーネント固有の設定を行うことができます(詳細については、[Configuration](Configuration.html)を参照してください)。
もちろん、あるコンポーネントが定義しているシリアライゼーションは、他のBoltでも使えるようになります -- そうしないと、そのコンポーネントからのメッセージを受信できなくなります!

トポロジがsubmitされると、トポロジ内のすべてのコンポーネントがメッセージを送信するために使用されるシリアライゼーションの集合が選択されます。これは、コンポーネント固有のシリアライザのレジストリと、通常のシリアライザのセットをマージすることによって行われます。2つのコンポーネントが同じクラスのシリアライザを定義する場合、シリアライザの1つが任意に選択されます。

2つのコンポーネント固有のレジストリの間に競合がある場合に特定のクラスに対してシリアライザを強制するには、使用したいシリアライザをトポロジ固有の設定として定義します。トポロジ固有の設定は、シリアライゼーションレジストリに関するのコンポーネント固有の構成よりも優先されます。

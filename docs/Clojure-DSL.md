---
title: Clojure DSL
layout: documentation
documentation: true
---
StormにはSpout、Bolt、トポロジを定義するためのClojure DSLが付属しています Clojure DSLはJava APIが公開しているすべてのものにアクセスできるため、ClojureのユーザーはJavaに触れることなくStormトポロジをコーディングすることができます。Clojure DSLは、名前空間[org.apache.storm.clojure]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/clojure.clj)のソースで定義されています。

このページでは、以下を含むClojure DSLのすべての部分を概説します:

1. トポロジを定義する
2. `defbolt`
3. `defspout`
4. トポロジをローカルまたはクラスタモードで実行する
5. トポロジをテストする

### Defining topologies

トポロジを定義するには、`topology`関数を使用します。`topology`は2つの引数を取ります:"spout specs"のマップと"bolt specs"のマップです。各Spout仕様とBolt仕様は、入力とparallelismなどを指定することにより、コンポーネントのコードをトポロジに結線します。

[storm-starterプロジェクトの]({{page.git-blob-base}}/examples/storm-starter/src/clj/org/apache/storm/starter/clj/word_count.clj)トポロジ定義の例を見てみましょう:

```clojure
(topology
 {"1" (spout-spec sentence-spout)
  "2" (spout-spec (sentence-spout-parameterized
                   ["the cat jumped over the door"
                    "greetings from a faraway land"])
                   :p 2)}
 {"3" (bolt-spec {"1" :shuffle "2" :shuffle}
                 split-sentence
                 :p 5)
  "4" (bolt-spec {"3" ["word"]}
                 word-count
                 :p 6)})
```

SpoutとBolt仕様のマップは、コンポーネントIDから対応する仕様へのマップです。コンポーネントIDはマップ全体で一意でなければなりません。Javaでトポロジを定義するのと同じように、コンポーネントIDはトポロジ内のBoltの入力を宣言するときに使用されます。

#### spout-spec

`spout-spec`は、Spoutの実装([IRichSpout](javadocs/org/apache/storm/topology/IRichSpout.html)を実装するオブジェクト)とオプションのキーワード引数を引数としてとります。現在存在する唯一のオプションは、Spoutのparallelismを指定する`:p`オプションです。`:p`を省略すると、Spoutは単一のタスクとして実行されます。

#### bolt-spec

`bolt-spec`は、入力宣言としてBoltの実装([IRichBolt](javadocs/org/apache/storm/topology/IRichBolt.html)を実装するオブジェクト)とオプションのキーワード引数の入力宣言を引数としてとります。

入力宣言は、ストリームIDからストリームグルーピングに対する対応付けです。ストリームIDは、次の2つの形式のいずれかを持つことができます:

1. `[==component id== ==stream id==]`: コンポーネントの特定のストリームをサブスクライブする
2. `==component id==`: コンポーネントのデフォルトストリームをサブスクライブする

ストリームグループは次のいずれかになります:

1. `:shuffle`: Shuffle Groupingでサブスクライブする
2. `["id" "name"]`のようなフィールド名のベクトル: 指定されたフィールドのField Groupingをサブスクライブする
3. `:global`: Global Groupingでサブスクライブする
4. `:all`: All Groupingでサブスクライブする
5. `:direct`: Direct Groupingでサブスクライブする

ストリームのグループ化の詳細については、[Concepts](Concepts.html)を参照してください。次に、入力を宣言するためのさまざまな方法の例を示します。

```clojure
{["2" "1"] :shuffle
 "3" ["field1" "field2"]
 ["4" "2"] :global}
```

この入力宣言は、合計3つのストリームをサブスクライブしています。コンポーネント"2"のストリーム"1"をShuffle Groupingでサブスクライブし、コンポーネント"3"のデフォルトストリームをサブスクライブし、フィールド"field1"および"field2"でField Groupingし、ストリーム"2"にサブスクライブするコンポーネント"4"はGlobal Groupingをしています。

`spout-spec`と同様に、`bolt-spec`で現在サポートされている唯一のキーワード引数は`:p`で、Boltのparallelismを指定します。

#### shell-bolt-spec

`shell-bolt-spec`は非JVM言語で実装されたボルトを定義するために使われます。引数として、入力宣言、実行するコマンドラインプログラム、Boltを実装するファイルの名前、出力仕様、そして`bolt-spec`が受け付けるのと同じキーワード引数を引数としてとります。

次に、`shell-bolt-spec`の例を示します:

```clojure
(shell-bolt-spec {"1" :shuffle "2" ["id"]}
                 "python"
                 "mybolt.py"
                 ["outfield1" "outfield2"]
                 :p 25)
```

出力宣言の構文については、後述の`defbolt`節で詳しく説明しています。Storm内でmultilangがどのように動作するかの詳細については、[Stormで非JVM言語を使用する](Using-non-JVM-languages-with-Storm.html)を参照してください。

### defbolt

`defbolt`はClojureでBoltを定義するために使われます。Boltには直列化が可能でなければならないという制約があります。`IRichBolt`でBoltを実装するのでは不十分なのはこのためです(closureは直列化できない)。`defbolt`はこの制限を回避し、Javaインターフェースを実装するだけでなく、Boltを定義するためのより良い構文を提供します。

`defbolt`はパラメータ化されたBoltをサポートし、Bolt実装周辺のclosureにおける状態を維持します。また、この余分な機能を必要としないBoltを定義するためのショートカットも用意されています。 `defbolt`のシグネチャは次のようになります:

(defbolt _name_ _output-declaration_ *_option-map_ & _impl_)

オプションマップを省略すると、`{:prepare false}`のオプションマップを持つのと同等になります。

#### Simple bolts

最もシンプルな形式の`defbolt`から始めましょう。ここでは、文を含むタプルを各単語のタプルに分割するボルトの例を示します:

```clojure
(defbolt split-sentence ["word"] [tuple collector]
  (let [words (.split (.getString tuple 0) " ")]
    (doseq [w words]
      (emit-bolt! collector [w] :anchor tuple))
    (ack! collector tuple)
    ))
```

オプションマップは省略されているので、これはnon-preparedなBoltです。DSLは`IRichBolt`の`execute`メソッドの実装を単に期待しています。この実装はタプルと`OutputCollector`の2つのパラメータをとり、`execute`関数の本体が続きます。DSLは自動的にパラメータに型ヒントをつけます。したがって、Java連携機能を使用する場合でも、リフレクションについて心配する必要はありません。

この実装は`split-sentence`をトポロジで使うことができる実際の`IRichBolt`オブジェクトにバインドします:

```clojure
(bolt-spec {"1" :shuffle}
           split-sentence
           :p 5)
```


#### Parameterized bolts

多くの場合、他の引数でBoltをパラメータ化したいことがあります。たとえば、受け取ったすべての入力文字列に接尾辞を追加し、その接尾辞を実行時に設定したいと思っているとします。`defbolt`のオプションマップに `:params`オプションを含めるとできます:

```clojure
(defbolt suffix-appender ["word"] {:params [suffix]}
  [tuple collector]
  (emit-bolt! collector [(str (.getString tuple 0) suffix)] :anchor tuple)
  )
```

前の例とは異なり、`suffix-appender`は`IRichBolt`オブジェクトではなく`IRichBolt`を返す関数にバインドされます。これは、オプションマップに`:params`を指定することによって発生します。 よって、`suffix-appender`をトポロジで使うには、次のようにします:

```clojure
(bolt-spec {"1" :shuffle}
           (suffix-appender "-suffix")
           :p 10)
```

#### Prepared bolts

結合やストリーミングアグリゲーションなど複雑なBoltを実行するには、Boltに状態を格納する必要があります。これを行うには、オプションマップに`{:prepare true}`を含めてPrepared Boltを生成します。たとえば、ワードカウントを実装するこのボルトを考えてみましょう:

```clojure
(defbolt word-count ["word" "count"] {:prepare true}
  [conf context collector]
  (let [counts (atom {})]
    (bolt
     (execute [tuple]
       (let [word (.getString tuple 0)]
         (swap! counts (partial merge-with +) {word 1})
         (emit-bolt! collector [word (@counts word)] :anchor tuple)
         (ack! collector tuple)
         )))))
```

Prepared Boltの実装は、トポロジの設定、`TopologyContext`ならびに`OutputCollector`を入力として受け取り、`IBolt`インタフェースの実装を返します。この設計では、`execute`と` cleanup`の実装周辺にclosureを置くことができます。

この例では、単語数は`counts`というマップにおけるclosureに格納されます。`bolt`マクロは`IBolt`実装の生成に使用されます。`bolt`マクロは、インターフェースを実装するための具体的な実装よりも簡潔な方法であり、自動的にすべてのメソッドのパラメータに型ヒントを付けます。このBoltは、マップ内のカウントを更新し、新しい単語カウントを送出するexecuteメソッドを実装しています。

Prepared Boltの`execute`メソッドは、`OutputCollector`がすでにその関数のclosureに入っているので、タプルを入力として受け取るだけです（単純なBoltの場合、コレクタは`execute`関数の第2パラメーターです）。

Prepared Boltは、シンプルなボルトのようにパラメータ化することができます。

#### Output declarations

Clojure DSLには、Boltの出力を宣言するための簡潔な構文があります。出力を宣言する最も一般的な方法は、ストリームIDからストリーム仕様への対応付けです。例えば:

```clojure
{"1" ["field1" "field2"]
 "2" (direct-stream ["f1" "f2" "f3"])
 "3" ["f1"]}
```

ストリームIDは文字列であり、ストリーム仕様はフィールドのベクトルか`direct-stream`で囲まれたフィールドのベクトルです。`direct stream`はストリームをダイレクトストリームとしてマークします(ダイレクトストリームの詳細については、[Concepts](Concepts.html)と[Direct groupings]()を参照してください)。

Boltに出力ストリームが1つしかない場合は、出力宣言にマップではなくベクトルを使用して、Boltのデフォルトストリームを定義できます。例えば:

```clojure
["word" "count"]
```
これは、Boltの出力をデフォルトのストリームIDのフィールド["word" "count"]として宣言します。

#### Emitting, acking, and failing

DSLは`OutputCollector`でJavaのメソッドを直接使用するよりより良い関数群を提供しています: `emit-bolt!`, `emit-direct-bolt!`, `ack!`, ならびに `fail!`です。

1. `emit-bolt!`: `OutputCollector`、送出する値（Clojureのシーケンス）、`:anchor`と`:stream`のキーワード引数をパラメータとして取ります。`:anchor`は単一のタプルまたはタプルのリストであり、`:stream`は送出するストリームのIDです。 キーワード引数を省略すると、アンカーされていないタプルがデフォルトのストリームに送出されます。
2. `emit-direct-bolt!`: `OutputCollector`、タプルを送信するタスクID、する値、`:anchor`と`:stream`のキーワード引数をパラメータとしてとります。この関数は、ダイレクトストリームとして宣言されたストリームにのみ送出できます。
2. `ack!`: `OutputCollector`とackすべきタプルを引数として受け取ります。
3. `fail!`: `OutputCollector`とfailすべきタプルを引数として受け取ります。

ackingとanchoringの詳細については、[メッセージ処理の保証](Guaranteeing-message-processing.html)を参照してください。

### defspout

`defspout`はClojureでスパウトを定義するために使われます。Boltと同様に、Spoutは直列化可能でなければならないので、ClojureでSpoutの実装を行うために単に`IRichSpout`を使用することはできません。`defspout`はこの制限を回避し、Javaインターフェースを実装するだけではなく、Spoutを定義するためのより良い構文を提供します。

`defspout`のシグネチャは次のようになります：

(defspout _name_ _output-declaration_ *_option-map_ & _impl_)

オプションマップを省略すると、デフォルトは{:prepare true}になります。`defspout`の出力宣言は`defbolt`と同じ構文です。

ここでは、[storm-starter]({{page.git-blob-base}}/examples/storm-starter/src/clj/org/apache/storm/starter/clj/word_count.clj)におけるの`defspout`の実装を示します:

```clojure
(defspout sentence-spout ["sentence"]
  [conf context collector]
  (let [sentences ["a little brown dog"
                   "the man petted the dog"
                   "four score and seven years ago"
                   "an apple a day keeps the doctor away"]]
    (spout
     (nextTuple []
       (Thread/sleep 100)
       (emit-spout! collector [(rand-nth sentences)])         
       )
     (ack [id]
        ;; You only need to define this method for reliable spouts
        ;; (such as one that reads off of a queue like Kestrel)
        ;; This is an unreliable spout, so it does nothing here
        ))))
```

実装は、トポロジの設定、`TopologyContext`と`SpoutOutputCollector`を入力として受け取ります。実装は`ISpout`オブジェクトを返します。ここで、`nextTuple`関数は、`sentences`からランダムな文を送出します。

このSpoutはreliableでないので、`ack`メソッドと`fail`メソッドは決して呼び出されません。reliableなSpoutは、タプルを送出するときにメッセージIDを追加し、タプルが完了したときまたは失敗したときに、`ack`または`fail`が呼び出されます。Stormにおけるでの信頼性の仕組みについては、[メッセージ処理の保証](Guaranteeing-message-processing.html)を参照してください。

`emit-spout!`は`SpoutOutputCollector`と送出される新しいタプルをパラメータとして取り込み、キーワード引数`:stream`と`:id`を受け取ります。`:stream`は送出するストリームを指定し、`:id`はタプルのメッセージIDを指定します（`ack`や`fail`コールバックで使用されます）。これらの引数を省略すると、デフォルト出力ストリームにanchorされていないタプルが送出されます。

ダイレクトストリームにタプルを送出し、タプルを送信するタスクIDの第2パラメータとして追加の引数を取る`emit-direct-spout!`関数もあります。

SpoutはBoltのようにパラメータ化することができます。この場合、シンボルは`IRichSpout`自体ではなく`IRichSpout`を返す関数にバインドされています。また、`nextTuple`メソッドだけを定義するunpreparedなSpoutを宣言することもできます。実行時にパラメータ化されたランダムな文章を出力する、unpreparedなSpoutの例を次に示します:

```clojure
(defspout sentence-spout-parameterized ["word"] {:params [sentences] :prepare false}
  [collector]
  (Thread/sleep 500)
  (emit-spout! collector [(rand-nth sentences)]))
```

次の例は、このSpoutを`spout-spec`でどのように使うかを示しています:

```clojure
(spout-spec (sentence-spout-parameterized
                   ["the cat jumped over the door"
                    "greetings from a faraway land"])
            :p 2)
```

### Running topologies in local mode or on a cluster

以上がClojure DSLのすべてです。リモートモードまたはローカルモードでトポロジを送信するには、Javaの場合と同じように`StormSubmitter`または`LocalCluster`クラスを使用してください。

トポロジの設定を生成するには、すべての可能なコンフィグレーションの定数を定義する `org.apache.storm.config`ネームスペースを使うのが最も簡単です。定数は`Config`クラスのスタティック定数と同じですが、アンダースコアの代わりにダッシュを使用します。たとえば、次のトポロジの設定では、ワーカー数を15に設定し、トポロジをデバッグモードで設定します:

```clojure
{TOPOLOGY-DEBUG true
 TOPOLOGY-WORKERS 15}
```

### Testing topologies

[このブログの記事](http://www.pixelmachine.org/2011/12/17/Testing-Storm-Topologies.html)とその[フォローアップ](http://www.pixelmachine.org/2011/12/21/Testing-Storm-Topologies-Part-2.html)は、ClojureのトポロジをテストするためのStormの強力な組み込み機能の概要を示しています。

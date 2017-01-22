---
title: Multi-Lang Protocol
layout: documentation
documentation: true
---
このページでは、Storm 0.7.1のmultilangプロトコルを説明します。0.7.1より前のバージョンでは、[ここ](Storm-multi-language-protocol-\(versions-0.7.0-and-below\).html)に記載されているやや異なるプロトコルを使用していました。

# Storm Multi-Language Protocol

## Shell Components

multipleのサポートは、ShellBolt、ShellSpout、およびShellProcessクラスを介して実装されています。
これらのクラスは、JavaのProcessBuilderクラスを使用し、
シェルを介してスクリプトまたはプログラムを実行するための
IBoltおよびISpoutインターフェースとプロトコルを実装します。

## Output fields

出力フィールドは、トポロジのThrift定義の一部です。つまり、Javaでmultilangする場合、ShellBoltを継承し、IRichBoltを実装し、 `declareOutputFields`（ShellSpoutと同様）でフィールドを宣言するBoltを作成する必要があります。

これについての詳細は、[Concepts](Concepts.html)を参照してください。

## Protocol Preamble

単純なプロトコルは、実行されたスクリプトまたはプログラムのSTDINおよびSTDOUTを介して実装されます。
プロセスと交換されるすべてのデータはJSONでエンコードされているため、
ほぼすべての言語でサポートが可能です。

# Packaging Your Stuff

クラスタ上でシェルコンポーネントを実行するには、
シェルに書き出されるスクリプトが、
マスターにsubmitされたjarファイル内の`resources/`ディレクトリになければなりません。

しかし、ローカルマシン上での開発やテストでは、
リソースディレクトリはクラスパス上にある必要があります

## The Protocol

注意:

* いずれのプロトコルも行の読み取り機構を使用しているので、
入力から改行を削除して出力に追加するようにしてください。
* すべてのJSON入力と出力は"end"を含む1行で終了します。この区切り文字自体はJSONエンコードされていないことに注意してください。
* 以下の箇条書きは、スクリプト作成者のSTDINとSTDOUTの観点から書かれています。

### Initial Handshake

初期化のハンドシェイクは、両方のタイプのシェルコンポーネントで同じです:

* STDIN: セットアップ情報。これはStorm設定、PIDディレクトリ、およびトポロジコンテキストを持つJSONオブジェクトです:

```
{
    "conf": {
        "topology.message.timeout.secs": 3,
        // etc
    },
    "pidDir": "...",
    "context": {
        "task->component": {
            "1": "example-spout",
            "2": "__acker",
            "3": "example-bolt1",
            "4": "example-bolt2"
        },
        "taskid": 3,
        // Everything below this line is only available in Storm 0.10.0+
        "componentid": "example-bolt"
        "stream->target->grouping": {
        	"default": {
        		"example-bolt2": {
        			"type": "SHUFFLE"}}},
        "streams": ["default"],
 		"stream->outputfields": {"default": ["word"]},
	    "source->stream->grouping": {
	    	"example-spout": {
	    		"default": {
	    			"type": "FIELDS",
	    			"fields": ["word"]
	    		}
	    	}
	    }
	    "source->stream->fields": {
	    	"example-spout": {
	    		"default": ["word"]
	    	}
	    }
	}
}
```

スクリプトは、このディレクトリにPIDで指定された空のファイルを作成する必要があります。
例えばPIDは1234なので、1234という名前の空のファイルがディレクトリに作成されます。
このファイルは、後でプロセスをシャットダウンできるようにスーパーバイザにPIDを知らせるものです。

Storm 0.10.0以降、Stormによってシェルコンポーネントに送信されたコンテキストは、
JVMコンポーネントで使用可能なトポロジコンテキストのすべての側面を含むように大幅に強化されました。
1つの重要な追加は、`stream->target->grouping`および`source->stream->grouping`のディクショナリを介して、
トポロジ内のシェルコンポーネントのソースおよびターゲット（すなわち入力および出力）を決定する能力です。
これらのネストされたディクショナリの最も内側のレベルでは、
グループ化は最小限の`type`キーを持つディクショナリとして表現されますが、
`FIELD`グルーピングにどのフィールドを含めるか指定する`fields`キーを持つこともできます。


* STDOUT: あなたのPIDは`{"pid": 1234}`のようなJSONオブジェクトにあります。シェルコンポーネントはPIDをログに記録します。

次にやるべきことは、コンポーネントのタイプによって異なります:

### Spouts

シェルスパウトは同期します。残りはwhile(true)内で実行されます：

* STDIN: next、ack、またはfailコマンドのいずれかです。

"next"はISpoutの`nextTuple`に相当します。それは次のように見えます:

```
{"command": "next"}
```

"ack"は次のようになります:

```
{"command": "ack", "id": "1231231"}
```

"fail"は次のようになります:

```
{"command": "fail", "id": "1231231"}
```

* STDOUT: 前のコマンドのSpoutの結果。これは、一連のemitおよびlogにすることができます。

emitは次のようになります:

```
{
	"command": "emit",
	// The id for the tuple. Leave this out for an unreliable emit. The id can
    // be a string or a number.
	"id": "1231231",
	// The id of the stream this tuple was emitted to. Leave this empty to emit to default stream.
	"stream": "1",
	// If doing an emit direct, indicate the task to send the tuple to
	"task": 9,
	// All the values in this tuple
	"tuple": ["field1", 2, 3]
}
```

直接的にemitを実行しないと、タプルがJSON配列としてSTDINにemitされたタスクIDをすぐに受け取ることになります。

"log"はワーカーログにメッセージを記録します。それは次のように見えます:

```
{
	"command": "log",
	// the message to log
	"msg": "hello world!"
}
```

* STDOUT: "sync"コマンドは、emitとlogのシーケンスを終了します。それは次のように見えます:

```
{"command": "sync"}
```

syncした後、ShellSpoutは別のnext、ack、またはfailコマンドを送信するまで、出力を読み込みません。

ISpoutと同様に、ワーカーのすべてのスパウトは、syncするまでnext、ack、またはfailの後にロックされることに注意してください。また、ISpoutのように、次にemitするタプルがない場合は、同期する前に少しの時間スリープ状態にする必要があります。ShellSpoutは自動的にあなたのためにスリープしません。

### Bolts

シェルボルトのプロトコルは非同期です。STDINが利用可能になるとすぐにタプルを受け取り、次のようにSTDOUTにemit, ack, fail, logを書き込むことができます:

* STDIN: タプル! これは次のようなJSONエンコードされた構造です:

```
{
    // The tuple's id - this is a string to support languages lacking 64-bit precision
	"id": "-6955786537413359385",
	// The id of the component that created this tuple
	"comp": "1",
	// The id of the stream this tuple was emitted to
	"stream": "1",
	// The id of the task that created this tuple
	"task": 9,
	// All the values in this tuple
	"tuple": ["snow white and the seven dwarfs", "field2", 3]
}
```

* STDOUT: emit, ack, fail, またはlogです。emitは次のようになります:

```
{
	"command": "emit",
	// The ids of the tuples this output tuples should be anchored to
	"anchors": ["1231231", "-234234234"],
	// The id of the stream this tuple was emitted to. Leave this empty to emit to default stream.
	"stream": "1",
	// If doing an emit direct, indicate the task to send the tuple to
	"task": 9,
	// All the values in this tuple
	"tuple": ["field1", 2, 3]
}
```

直接的にemitを実行しない場合は、タプルがJSON配列としてSTDINにemitされたタスクIDを受け取ります。
シェルボルトプロトコルの非同期性のため、emit後に読み取っても、タスクIDが受信されないことがあります。
代わりに、以前のemitまたは処理すべき新しいタプルのタスクIDを読み取ります。
ただし、タスクIDのリストは、対応するemitと同じ順序で受け取ることができます。

ackは次のようになります:

```
{
	"command": "ack",
	// the id of the tuple to ack
	"id": "123123"
}
```

failは次のようになります:

```
{
	"command": "fail",
	// the id of the tuple to fail
	"id": "123123"
}
```

"log"はワーカーログにメッセージを記録します。それは次のようになります:

```
{
	"command": "log",
	// the message to log
	"msg": "hello world!"
}
```

* バージョン0.7.1以降、
シェルボルトを'sync'する必要はもうありません。

### Handling Heartbeats (0.9.3 and later)

Storm 0.9.3以降、ハングしたりゾンビになってしまうサブプロセスを検出するために、
ShellSpout/ShellBoltとmulti-langサブプロセスの間でハートビートが行われています。
multi-langを介してStormと接続するためのライブラリは、
ハートビートに関する以下のアクションを実行する必要があります:

#### Spout

シェルスパウトは同期しているので、サブプロセスは常に`next()`の最後に`sync`コマンドを送ります。
したがって、Spoutでハートビートをサポートするために多くのことをする必要はありません。
言い換えれば、`next()`中にサブプロセスをワーカーのタイムアウトよりも長くsleepさせてはいけません。

#### Bolt

シェルボルトは非同期であるため、ShellBoltはハートビートタプルをそのサブプロセスに定期的に送信します。
ハートビートタプルは次のようになります:

```
{
	"id": "-6955786537413359385",
	"comp": "1",
	"stream": "__heartbeat",
	// this shell bolt's system task id
	"task": -1,
	"tuple": []
}
```

サブプロセスがハートビートタプルを受信すると、`sync`コマンドをShellBoltに送り返す必要があります。

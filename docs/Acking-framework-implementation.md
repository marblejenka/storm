---
layout: documentation
---
Stormの[acker](https://github.com/apache/incubator-storm/blob/46c3ba7/storm-core/src/clj/backtype/storm/daemon/acker.clj#L28)は、各タプルツリーの完了をチェックサムハッシュで追跡します。タプルが送信されるたびに、その値はチェックサムにXOR演算され、タプルがackされるたびにその値がXOR演算されます。すべてのタプルが正常にackされた場合、チェックサムはゼロになります（そうでない場合にチェックサムがゼロになる確率はほとんどゼロになります）。

wikiにある[信頼性メカニズム](Guaranteeing-message-processing.html#what-is-storms-reliability-api)についてもう少し詳しく読むことができます。これは内部の詳細を説明しています。

### acker `execute()`

ackerは実際には`mk-acker-bolt`で定義された[executeメソッド](https://github.com/apache/incubator-storm/blob/46c3ba7/storm-core/src/clj/backtype/storm/daemon/acker.clj#L36)を持つ普通のboltです。新しいタプルツリーが生成されたとき、Spoutは各タプルの受信者にXORされたエッジIDを送信し、ackerはその`pending`元帳に記録します。エグゼキュータがタプルをackするたびに、ackerは、タプルが保持するエッジIDのXORである部分チェックサム（元帳からクリアする）と、エグゼキュータが送出した各下流のタプルのエッジIDのを受け取ります（それらを元帳に追加される）。

これは以下のようにして達成されます。

チックタプルでは、保留中のタプルツリーのチェックサムを成功または失敗に向けて進めます。それ以外の場合は、タプルツリーの記録を更新または生成します。

* on init: 与えられたチェックサム値で初期化し、後でspoutのIDを記録します。
* on ack: 既存のチェックサムを部分チェックサムでXOR演算する
* on fail: 失敗したものとしてマークする

次に、RotatingMapに[記録をputし](https://github.com/apache/incubator-storm/blob/46c3ba7/storm-core/src/clj/backtype/storm/daemon/acker.clj#L50))（リセットすることが有効期限までのカウントダウンになる）、アクションを実行します：

* 合計チェックサムがゼロの場合、タプルツリーは完了しています: タプルツリー保留中のコレクションから削除して、Spoutに成功を通知します
* タプルツリーが失敗した場合も、完了になります: タプルツリーを保留中のコレクションから削除し、Spoutに失敗したことを通知します

最後に、ackerそのものにackを伝えます。

### Pending tuples and the `RotatingMap`

ackerは、処理を効率的に期限切れにするために、Storm内のいくつかの場所で使用されるシンプルな仕組みである[`RotatingMap`](https://github.com/apache/incubator-storm/blob/master/storm-core/src/jvm/backtype/storm/utils/RotatingMap.java#L19)に保留中のタプルを格納します。

RotatingMapはHashMapとして動作し、O(1)でデータにアクセスすることを保証します。

内部的には、それは複数のHashMap('buckets')を保持し、それぞれが同時に期限切れになるレコードのまとまりを保持します。最長生存しているbucketをのdeath row、最新のものをnurseryとしましょう。値がRotatingMapに `.put()`されると、それはnurseryに再配置され -- 他のバケットから削除されます（効率的ににdeathした時刻をリセットします）。

その所有者が`.rotate()`を呼び出すたびに、RotatingMapはそれぞれのまとまりを有効期限に向けて進めます。（通常、Stormオブジェクトは、システムからのチックタプルの受信があるたびに、rotateを呼び出します。）前のdeath rowバケットにキーと値のペアがある場合、その所有者に適切な処置（例えば、タプルの失敗）をさせるために、RotatingMapは各キーと値のペアについてコールバック（コンストラクタで指定）を呼び出します。

---
title: Storm SQL integration
layout: documentation
documentation: true
---

Storm SQLの統合により、ユーザはStormのストリーミングデータに対してSQLクエリを実行できます。 SQLインターフェイスによってストリーミング分析の開発サイクルも短縮されるだけでなく、[Apache Hive](///hive.apache.org)のようなバッチ処理とリアルタイムなストリーミングデータ分析を統一する機会も開かれます。

非常に高位のレベルから、StormSQLはSQLクエリを[Trident](Trident-API-Overview.html)のトポロジにコンパイルし、Stormクラスタで実行します。このドキュメントでは、StormSQLをエンドユーザとして使用する方法について説明します。 StormSQLの設計と実装の詳細については、[この](storm-sql-internal.html)のページを参照してください。

## Usage

``storm sql``コマンドを実行してSQL文をTridentトポロジにコンパイルし、それをStormクラスタに送信します

```
$ bin/storm sql <sql-file> <topo-name>
```

`sql-file`は実行されるSQL文のリストを含み、` topo-name`はトポロジの名前です。

## Supported Features

現在のリポジトリでは、次の機能がサポートされています:

* 外部データソースから/外部データソースに対するストリーミング
* タプルのフィルタリング
* 射影

## Specifying External Data Sources

StormSQLでは、データは外部テーブルによって表されます。ユーザーは `CREATE EXTERNAL TABLE`ステートメントを使ってデータソースを指定することができます。`CREATE EXTERNAL TABLE`の構文は、[HiveのDDL](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL)で定義されているものに厳密に従います。

```
CREATE EXTERNAL TABLE table_name field_list
    [ STORED AS
      INPUTFORMAT input_format_classname
      OUTPUTFORMAT output_format_classname
    ]
    LOCATION location
    [ TBLPROPERTIES tbl_properties ]
    [ AS select_stmt ]
```

[HiveのDDL](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL)にプロパティの詳細な説明があります。 たとえば、次の文はKafkaのspoutとsinkを指定します。

```
CREATE EXTERNAL TABLE FOO (ID INT PRIMARY KEY) LOCATION 'kafka://localhost:2181/brokers?topic=test' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.org.apache.storm.kafka.ByteBufferSerializer"}}'
```

## Plugging in External Data Sources

ユーザは`ISqlTridentDataSource`インタフェースを実装して外部データソースをプラグインし、Javaのservice loaderの機能を使用して登録します。 外部データソースは、テーブルのURIのスキームに基づいて選択されます。詳細については、`storm-sql-kafka`の実装を参照してください。

## Example: Filtering Kafka Stream

注文の取引を表すKafkaのストリームがあるとします。ストリーム内の各メッセージには、注文ID、製品の単価、注文の数量が含まれています。ゴールは、重要な注文を特定のうえ、その注文を別の分析のために別のストリームとして挿入することです。

ユーザーは、SQLファイルに次のSQL文を指定できます:

```
CREATE EXTERNAL TABLE ORDERS (ID INT PRIMARY KEY, UNIT_PRICE INT, QUANTITY INT) LOCATION 'kafka://localhost:2181/brokers?topic=orders' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.org.apache.storm.kafka.ByteBufferSerializer"}}'
CREATE EXTERNAL TABLE LARGE_ORDERS (ID INT PRIMARY KEY, TOTAL INT) LOCATION 'kafka://localhost:2181/brokers?topic=large_orders' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.org.apache.storm.kafka.ByteBufferSerializer"}}'
INSERT INTO LARGE_ORDERS SELECT ID, UNIT_PRICE * QUANTITY AS TOTAL FROM ORDERS WHERE UNIT_PRICE * QUANTITY > 50
```

最初のステートメントは入力ストリームを表す`ORDER`テーブルを定義します。`LOCATION`節は、brokerのZkHost(`localhost:2181`)、ZooKeeperのパス(`/brokers`)と、トピック(`orders`)を指定しています。`TBLPROPERTIES`節は[KafkaProducer](http://kafka.apache.org/documentation.html#producerconfigs)の設定です。
`storm-sql-kafka`の現在の実装では、テーブルが読み込み専用または書き込み専用であっても、`LOCATION`節と`TBLPROPERTIES`節の両方を指定する必要があります。

同様に、2番目のステートメントは、出力ストリームを表す `LARGE_ORDERS`テーブルを指定しています。 3番目のステートメントは`SELECT`ステートメントで、トポロジーを定義しています。StormSQLは外部テーブル`ORDERS`のすべての注文をフィルタリングし、合計価格を計算のうえ、条件にマッチしたレコードを`LARGE_ORDER`のKafkaのストリームに挿入します。

この例を実行するには、データソース（この場合は`storm-sql-kafka`）とそのクラスパスへの依存関係を含める必要があります。必要なjarファイルを`extlib`ディレクトリに置くのも一つの方法です。

```
$ cp curator-client-2.5.0.jar curator-framework-2.5.0.jar zookeeper-3.4.6.jar
 extlib/
$ cp scala-library-2.10.4.jar kafka-clients-0.8.2.1.jar kafka_2.10-0.8.2.1.jar metrics-core-2.2.0.jar extlib/
$ cp json-simple-1.1.1.jar extlib/
$ cp jackson-annotations-2.6.0.jar extlib/
$ cp storm-kafka-*.jar storm-sql-kafka-*.jar storm-sql-runtime-*.jar extlib/
```

次のステップでは、SQLステートメントをStormSQLに送信しています。

```
$ bin/storm sql order_filtering order_filtering.sql
```

ここまでで、Storm UIで`order_filtering`トポロジを見ることができます

## Current Limitations

集約、ウィンドウ処理、結合はまだ実装されていません。トポロジに対して並列処理ヒントを指定する機能はまだサポートされていません。すべてのプロセッサのparallelism hintは1です。

ユーザーはまた、`extlib`ディレクトリに外部データソースの依存関係を提供する必要があります。さもなければ、トポロジは `ClassNotFoundException`のために実行に失敗します。

StormSQLのKafkaコネクタの現在の実装では、入力と出力の両方がJSON形式であると想定しています。コネクターは`INPUTFORMAT`と`OUTPUTFORMAT`節をまだ認識していません。

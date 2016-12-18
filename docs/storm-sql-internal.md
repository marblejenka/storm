---
title: The Internals of Storm SQL
layout: documentation
documentation: true
---

このページでは、Storm SQLの設計と実装について説明します。

## Overview

SQLはよく採用されていますが複雑な標準です。Drill、Hive、Phoenix、Sparkを含むいくつかのプロジェクトは、SQLレイヤーに大きな投資を行っています。StormSQLの主な設計目標の1つは、これらのプロジェクトが行った既存の投資を活用することです。StormSQLは[Apache Calcite](///calcite.apache.org)を活用してSQL標準を実装しています。StormSQLは、SQL文をStorm / Tridentのトポロジへコンパイルすることに注力しており、その実装だけでStormクラスタで実行できるようになります。

<div align="center">
<img title="Workflow of StormSQL" src="images/storm-sql-internal-workflow.png" style="max-width: 80rem"/>

<p>Figure 1: Workflow of StormSQL.</p>
</div>

次のステップでは、論理実行計画を物理実行計画にコンパイルします。物理計画は、*StormSQL*でSQLクエリを実行する方法を記述する物理演算子で構成されます。`Filter`、`Projection`、`GroupBy`などの物理演算子は、Tridentトポロジの操作に直接的に対応付けされます。StormSQLはまた、SQL文に置ける式をJavaバイトコードにコンパイルし、それらをTridentトポロジにプラグインします。

最後に、StormSQLは、Javaバイトコードとトポロジの両方をJARにパッケージ化し、Stormクラスタに送信します。Stormは他のStormトポロジを実行するのと同じ方法でJARをスケジュールして実行します。

以下のコードブロックは、Kafkaのストリームから、結果を抽出・射影するサンプルクエリを示しています。

```
CREATE EXTERNAL TABLE ORDERS (ID INT PRIMARY KEY, UNIT_PRICE INT, QUANTITY INT) LOCATION 'kafka://localhost:2181/brokers?topic=orders' ...

CREATE EXTERNAL TABLE LARGE_ORDERS (ID INT PRIMARY KEY, TOTAL INT) LOCATION 'kafka://localhost:2181/brokers?topic=large_orders' ...

INSERT INTO LARGE_ORDERS SELECT ID, UNIT_PRICE * QUANTITY AS TOTAL FROM ORDERS WHERE UNIT_PRICE * QUANTITY > 50
```

最初の2つのSQL文は、外部データの入力と出力を定義します。図2は、StormSQLが最後の`SELECT`クエリを受け取り、それをTridentのトポロジにコンパイルするプロセスを示しています。

<div align="center">
<img title="Compiling the example query to Trident topology" src="images/storm-sql-internal-example.png" style="max-width: 80rem"/>

<p>Figure 2: Compiling the example query to Trident topology.</p>
</div>


## Constraints of querying streaming tables

リアルタイムのデータストリームでできているテーブルに対してクエリする場合、いくつかの制約があります。

* `ORDER BY`節はストリームには適用できません。
* StormSQLがバッファのサイズを制限できるように、GROUP BY句には少なくとも一つのmonotonicなフィールドが必要です。

詳細については、 http：//calcite.apache.org/docs/stream.html を参照してください。

## Dependency

StormSQLは、パッケージ化されたJARに外部データソースの依存関係を格納しません。ユーザーはワーカーノードの`extlib`ディレクトリに依存関係を提供する必要があります。

---
title: Topology event inspector
layout: documentation
documentation: true
---

# Introduction

Topology event inspectorは、タプルがStormのトポロジにおけるさまざまな段階を流れていくのを見る機能を提供します。
これは、トポロジを停止または再配置することなく、トポロジが実行されている間に、トポロジパイプラインのSpoutまたはBoltに送出されたタプルを検査する場合に役立ちます。SpoutからBoltへのタプルの正常な流れは、イベントロギングをオンにすることによって影響を受けません。

## Enabling event logging

注意:イベントロギングを有効にする前に、Stormの設定である"topology.eventlogger.executors"をゼロ以外の値に設定する必要があります。詳細については、[Configuration](#config)を参照してください。

トポロジ・ビューのトポロジ・アクションの下にあるボタン"Debug"をクリックすると、イベントを記録できます。
これにより、指定されたサンプリング率でトポロジ内のすべてのSpoutとBolにおけるタプルがログに記録されます。

<div align="center">
<img title="Enable Eventlogging" src="images/enable-event-logging-topology.png" style="max-width: 80rem"/>

<p>Figure 1: Enable event logging at topology level.</p>
</div>

特定のSpoutまたはBoltレベルでイベントロギングを有効にするには、対応するコンポーネントページに移動し、コンポーネントアクションの下で"Debug"をクリックします。

<div align="center">
<img title="Enable Eventlogging at component level" src="images/enable-event-logging-spout.png" style="max-width: 80rem"/>

<p>Figure 2: Enable event logging at component level.</p>
</div>

## Viewing the event logs
記録されたタプルを表示するには、Stormの"logviewer"が実行されている必要があります。まだ実行していない場合は、Stormをインストールしたディレクトリで"bin/storm logviewer"コマンドを実行してログビューアを起動することができます。 タプルを表示するには、StomのUIで特定のSpoutまたはBoltコンポーネントページに移動し、component summaryの下にある"events"リンクをクリックします（上記のFigure2でハイライトされているもの）。

これにより、以下のようなビューが開き、異なるページ間を移動して記録されたタプルを見ることができます。

<div align="center">
<img title="Viewing logged tuples" src="images/event-logs-view.png" style="max-width: 80rem"/>

<p>Figure 3: Viewing the logged events.</p>
</div>

イベントログの各行には、特定のspout/boltからカンマ区切りの形式で出力されたタプルに対応するエントリが含まれています。

`Timestamp, Component name, Component task-id, MessageId (in case of anchoring), List of emitted values`

## Disabling the event logs

イベントロギングは、Storm UIのトポロジまたはコンポーネントアクションの下にある"Stop Debug"をクリックすることで、特定のコンポーネントまたはトポロジレベルで無効にすることができます。

<div align="center">
<img title="Disable Eventlogging at topology level" src="images/disable-event-logging-topology.png" style="max-width: 80rem"/>

<p>Figure 4: Disable event logging at topology level.</p>
</div>

## <a name="config"></a>Configuration
イベントロギングは、各コンポーネントのイベント（タプル）を内部的なeventlogger boltに送信することによって機能します。デフォルトでは、Stormはイベントロガーのタスクを開始しませんが、これはトポロジーの実行中に以下のパラメーターを設定することで簡単に変更できます（storm.yamlで設定するか、コマンドラインからオプションを渡す）。

| Parameter  | Meaning |
| -------------------------------------------|-----------------------|
| "topology.eventlogger.executors": 0      | イベントロガーのタスクは生成されません（デフォルト）。 |
| "topology.eventlogger.executors": 1      | トポロジーに対するイベント・ロガーのタスクが1つ生成される。 |
| "topology.eventlogger.executors": nil      | ワーカー1つに対してイベントロガーのタスクが1つ生成される。 |


## Extending eventlogging
Stormは、イベントを記録するためにeventlogger boltによって使用される`IEventLogger`インターフェースを提供します。これに対するデフォルトの実装は、events.logファイル（`logs/workers-artifacts/<topology-id>/<worker-port>/events.log`）にイベントを記録するFileBasedEventLoggerです。`IEventLogger`インターフェースの別の実装は、イベントロギング機能を拡張するために追加することができます（例えば、検索インデックスを構築する、またはデータベースにおけるのイベントを記録する）

```java
/**
 * EventLogger interface for logging the event info to a sink like log file or db
 * for inspecting the events via UI for debugging.
 */
public interface IEventLogger {
    /**
    * Invoked during eventlogger bolt prepare.
    */
    void prepare(Map stormConf, TopologyContext context);

    /**
     * Invoked when the {@link EventLoggerBolt} receives a tuple from the spouts or bolts that has event logging enabled.
     *
     * @param e the event
     */
    void log(EventInfo e);

    /**
    * Invoked when the event logger bolt is cleaned up
    */
    void close();
}
```

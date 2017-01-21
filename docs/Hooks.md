---
title: Hooks
layout: documentation
documentation: true
---
Stormは、Storm内の任意の数のイベントで実行するカスタムコードを挿入できるフックを提供します。[BaseTaskHook](javadocs/org/apache/storm/hooks/BaseTaskHook.html)クラスを継承し、キャッチしたいイベントに対して適切なメソッドをオーバーライドすることによってフックを作成します。フックを登録する方法は2つあります:

1. SpoutのopenメソッドまたはBoltのprepareメソッドで、[TopologyContext](javadocs/org/apache/storm/task/TopologyContext.html#addTaskHook)メソッドを使用します。
2. ["topology.auto.task.hooks"](javadocs/org/apache/storm/Config.html#TOPOLOGY_AUTO_TASK_HOOKS)を使ってStormの設定を行います。これらのフックはすべてのSpoutまたはBoltに自動的に登録され、カスタムの監視システムとの統合などの作業に役立ちます。

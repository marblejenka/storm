---
title: Daemon Fault Tolerance
layout: documentation
documentation: true
---
Stormにはいくつかの異なるデーモンプロセスがあります。 ワーカーをスケジュールするNimbus、ワーカーを起動および停止するsupervisors、ログにアクセスできるlog viewer、およびクラスタのステータスを表示するUIが含まれます。

## What happens when a worker dies?

ワーカーが死ぬと、supervisorはそれをリスタートします。起動時に連続して失敗し、かつ、Nimbusにハートビートすることができない場合、Nimbusはワーカーを再スケジュールします。

## What happens when a node dies?

そのマシンに割り当てられたタスクはタイムアウトし、Nimbusはそれらのタスクを他のマシンに再割り当てします。

## What happens when Nimbus or Supervisor daemons die?

NimbusデーモンとSupervisorデーモンは、fail-fast（予期せぬ状況に遭遇した場合は自らデストラクトを実行します）、stateless（すべての状態はZookeeperのディスクに保存されます）に設計されています。[Storm clusterの設定](Setting-up-a-Storm-cluster.html)で説明したように、NimbusとSupervisorデーモンは、daemontoolsやmonitのようなツールを使って監視下で実行する必要があります。したがって、NimbusまたはSupervisorのデーモンが死んでも、何もなかったかのように再起動します。

最も顕著なことに、NimbusやSupervisorが死んでもワーカープロセスには影響しません。これは、JobTrackerが終了すると実行中のジョブがすべて失われるHadoopとは対照的です。

## Is Nimbus a single point of failure?

あなたがNimbusノードを失った場合出会っても、ワーカーは引き続き機能します。さらに、supervisorはワーカーが死ぬとそれを再開させ続けるでしょう。しかし、Nimbusがなければ、必要な時出会ってもワーカーを他のマシンに割り当て直すことはありません（ワーカーマシンをロストした場合など）。

StormのNimbusは、1.0.0以降、HA構成をサポートしています。詳細は[Nimbus HA Design](nimbus-ha-design.html)のドキュメントを参照してください。

## How does Storm guarantee data processing?

Stormは、ノードが消滅したりメッセージが失われてもデータ処理を保証するメカニズムを提供します。詳細については、[メッセージ処理の保証](Guaranteeing-message-processing.html)を参照してください。

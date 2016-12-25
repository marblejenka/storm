---
title: Scheduler
layout: documentation
documentation: true
---

Stormには4種類の組み込みスケジューラがあります: [DefaultScheduler]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/scheduler/DefaultScheduler.clj), [IsolationScheduler]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/scheduler/IsolationScheduler.clj), [MultitenantScheduler]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/scheduler/multitenant/MultitenantScheduler.java), [ResourceAwareScheduler](Resource_Aware_Scheduler_overview.html). 

## Pluggable scheduler
エグゼキュータをワーカーに割り当てるために、独自のスケジューラを実装してデフォルトのスケジューラを置き換えることができます。storm.yamlで"storm.scheduler"に使用したいクラスを設定します。また、スケジューラは[IScheduler]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/scheduler/IScheduler.java)インターフェイスを実装する必要があります。

## Isolation Scheduler
Isolation Schedulerを使用すると、多くのトポロジ間でクラスタを簡単かつ安全に共有できます。Isolation Schedulerでは「隔離」すべきトポロジを指定します。つまり、隔離されたトポロジは他のトポロジが実行されていないクラスタ内の専用マシン上で実行されます。これらの隔離されたトポロジはクラスタ上で優先されるため、隔離されていないトポロジと競合する場合はリソースが隔離されたトポロジに割り当てられ、隔離トポロジがリソースを取得する場合は非隔離トポロジの分から取得します。すべての隔離トポロジが割り当てられると、クラスタ上の残りのマシンはすべての非隔離トポロジで共有されます。

Isolation Schedulerを設定するには、Nimbusの"storm.scheduler"を"org.apache.storm.scheduler.IsolationScheduler"に設定します。次に、"isolation.scheduler.machines"設定を使用して、各トポロジが取得できるマシンの数を指定します。この設定は、トポロジ名とトポロジに割り当てられた独立したマシンの数の対応付けです。例えば:

```
isolation.scheduler.machines: 
    "my-topology": 8
    "tiny-topology": 1
    "some-other-topology": 3
```

クラスタに送信されたトポロジはで、ここにリストされていないものは隔離されません。Stormのユーザーが隔離設定に影響を及ぼす方法はないことに注意してください。これはクラスタの管理者だけに許可されています（意図的にそうなっています）。

Isolation Schedulerは、トポロジを完全に隔離することで、マルチテナント問題（トポロジ間のリソース競合を回避する）を解決します。テストまたは開発中のトポロジではなく、"本番用の"トポロジを隔離設定にリストすることを意図しています。クラスタ上の残りのマシンは、隔離トポロジのfailoverとしての役割と、隔離されていないトポロジを実行する役割の二つを果たします。


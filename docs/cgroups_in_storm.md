---
title: CGroup Enforcement
layout: documentation
documentation: true
---

# CGroups in Storm

CGroupはStormによって、公平性とQOSを保証するためにワーカーのリソース使用を制限するために使用されます。

**注意: CGroupsは現在、Linuxプラットフォーム（カーネルバージョン2.6.24以上）でのみサポートされています** 

## Setup

CGroupを使用するには、cgroupをインストールし、cgroupを正しく設定するようにしてください。設定の詳細については、次をご覧ください:

https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/ch-Using_Control_Groups.html

サンプル/デフォルトのcgconfig.confファイルは、<stormroot>/confディレクトリにあります。内容は次のとおりです:

```
mount {
	cpuset	= /cgroup/cpuset;
	cpu	= /cgroup/storm_resources;
	cpuacct	= /cgroup/cpuacct;
	memory	= /cgroup/storm_resources;
	devices	= /cgroup/devices;
	freezer	= /cgroup/freezer;
	net_cls	= /cgroup/net_cls;
	blkio	= /cgroup/blkio;
}

group storm {
       perm {
               task {
                      uid = 500;
                      gid = 500;
               }
               admin {
                      uid = 500;
                      gid = 500;
               }
       }
       cpu {
       }
}
```

cgconfig.confファイルの形式と設定の詳細については、次のサイトを参照してください:

https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/ch-Using_Control_Groups.html#The_cgconfig.conf_File

# Settings Related To CGroups in Storm

| Setting                       | Function                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| storm.cgroup.enable                | この設定は、cgroupを使用するかどうかを設定するために使用されます。cgroupの使用を有効にするには"true"を設定します。cgroupを使用しない場合は"false"を設定します。このconfigをfalseに設定すると、cgroupに関連する単体テストはスキップされます。デフォルトは"false"に設定されています
|
| storm.cgroup.hierarchy.dir   | Stormが使用するcgroup階層へのパス。デフォルトは"/cgroup/storm_resources"に設定されています
| storm.cgroup.resources       | CGroupによって制限されるサブシステムのリスト。デフォルトはcpuとメモリに設定されています。現在はCPUとメモリのみがサポートされています
|
| storm.supervisor.cgroup.rootdir     | スーパーバイザが使用するルートcgroup。cgroupへのパスは、\<storm.cgroup.hierarchy.dir>/\<storm.supervisor.cgroup.rootdir>です。 デフォルトは "storm"に設定されています                                                                                                                                                                                                                                                                                                                                                                           |
| storm.cgroup.cgexec.cmd            | cgroup内でワーカーを起動するために使用されるcgexecコマンドの絶対パス。デフォルトは"/bin/cgexec"に設定されています
|
| storm.worker.cgroup.memory.mb.limit | 各ワーカーのメモリ制限（MB）。スーパーバイザノードごとに設定できます。この設定は、cgroupのmemory.limit_in_bytesを設定するために使用されます。memory.limit_in_bytesの詳細については、https：//access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/sec-memory.htmlを参照してください。Resource Aware Schedulerを使用している場合、この設定はResource Aware Schedulerによって計算された値を上書きしてしまうので、この設定を使用しないでください。 |
| storm.worker.cgroup.cpu.limit       | 各ワーカーのCPUシェア。スーパーバイザノードごとに設定できます。この設定は、cgroupのcpu.shareを設定するために使用されます。cpu.shareの詳細については、https：//access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/sec-cpu.htmlを参照してください。Resource Aware Schedulerを使用している場合、この設定はResource Aware Schedulerによって計算された値を上書きしてしまうので、この設定を設定しないでください。                                       |

cpu.sharesはそのプロセスのCPU使用率に比例したCPU使用率を制限するため、スーパーバイザノード上のすべてのワーカープロセスのCPU使用量を制限するためには、supervisor.cpu.capacityを設定してください。1％増やすのは1コアに対する割合を表しているので、ユーザがsupervisor.cpu.capacity：200を設定した場合、ユーザは2コアの使用を指示していることになります。

## Integration with Resource Aware Scheduler

CGroupは、Resource Aware Schedulerと組み合わせて使用できます。CGroupsは、Resource Aware Schedulerによって割り当てられたワーカーのリソース使用を強制します。Resource Aware Schedulerでcgroupを使用するには、単にcgroupを有効にして、storm.worker.cgroup.memory.mb.limitとstorm.worker.cgroup.cpu.limit configsを設定しないでください。



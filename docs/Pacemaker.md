---
title: Pacemaker
layout: documentation
documentation: true
---


### Introduction
Pacemakerは、ワーカーからのハートビートを処理するように設計されたStormのデーモンです。Stormがスケールアップされると、ZooKeeperはワーカーからのハートビートがもたらす大量の書き込みによってボトルネックになります。ZooKeeperに一貫性を維持させようとすると、ディスクへの書き込みとネットワーク上のトラフィックが多くなりすぎてしまいます。

ハートビートは一時的な性質を持つため、ディスクに永続化する必要はなく、ノード間で同期させる必要もありません。インメモリに格納させることができます。これがPacemakerの役割です。Pacemakerは、ZooKeeperのようにディレクトリスタイルのキーとバイト配列の値を持つ、単純なインメモリのキーバリューストアとして機能します。

対応するPacemakerクライアントは、`ClusterState`インターフェースのプラグインである`org.apache.storm.pacemaker.pacemaker_state_factory`です。 ハートビートの呼び出しは`pacemaker_state_factory`によって生成された`ClusterState`によってPacemakerデーモンに流入し、その他のセット/ゲット操作はZooKeeperに転送されます。

------

### Configuration

 - `pacemaker.host` : Pacemakerデーモンが動作しているホスト
 - `pacemaker.port` : Pacemakerがリッスンするポート
 - `pacemaker.max.threads` : Pacemakerデーモンが要求を処理するために使用するスレッドの最大数。
 - `pacemaker.childopts` : Pacemakerが使用するべきJVMのパラメータ。（storm-deployプロジェクトで使用されます）
 - `pacemaker.auth.method` : 使用されている認証方法（詳細は後述）

#### Example

Pacemakerを起動して実行するには、すべてのノードのクラスタ設定で次のオプションを設定します:
```
storm.cluster.state.store: "org.apache.storm.pacemaker.pacemaker_state_factory"
```

Pacemakerホストもすべてのノードに設定する必要があります:
```
pacemaker.host: somehost.mycompany.com
```

そして、すべてのデーモンを起動します

(Pacemakerも含めて):
```
$ storm pacemaker
```

これで、Stormクラスタは、すべてのワーカーのハートビートををPacemakerにプッシュします。

### Security

現在、ダイジェスト（パスワードベース）とKerberosセキュリティがサポートされています。セキュリティは現在、読み取りのみについてのものであり、書き込みについてのものはありません。書き込みは誰でも実行できますが、読み取りは認証・認可されたユーザーのみが実行できます。これは、将来の開発される予定のものであり、クラスタをDoS攻撃に開放してはいますが、主要な目標である機密情報が不正に参照されることを防ぎます。

#### Digest
ダイジェスト認証を設定するには、NimbusとPacemakerをホストするノードのクラスタ設定で`pacemaker.auth.method: DIGEST`を設定します。
また、下記の構造を含むJAAS設定ファイルを指すように、`java.security.auth.login.config`を設定する必要があります:

```
PacemakerDigest {
    username="some username"
    password="some password";
};
```

これらの設定がなされているノードは、Pacemakerから読み取ることができます。
ワーカーノードはこれらの設定を行う必要はなく、また、`pacemaker.auth.method: NONE`としておけばPacemakerデーモンから読み取る必要がないため、そのままにしておくことができます。

#### Kerberos
Kerberos認証を設定するには、NimbusとPacemakerをホストするノードのクラスタ設定で`pacemaker.auth.method: KERBEROS`を設定します。
また、ノードはJAAS設定を指すように`java.security.auth.login.config`を設定する必要があります。

NimbusのJAAS設定は、次のようになります:

```
PacemakerClient {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    keyTab="/etc/keytabs/nimbus.keytab"
    storeKey=true
    useTicketCache=false
    serviceName="pacemaker"
    principal="nimbus@MY.COMPANY.COM";
};
```

PacemakerのJAAS設定は、次のようになります:

```
PacemakerServer {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="/etc/keytabs/pacemaker.keytab"
   storeKey=true
   useTicketCache=false
   principal="pacemaker@MY.COMPANY.COM";
};
```

 - Nimbusホストの`PacemakerClient`セクションにおけるクライアントのユーザプリンシパルは、Stormクラスタの`nimbus.daemon.user`の設定値と一致しなければなりません。
 - クライアントの`serviceName`値は、Pacemakerホストの`PacemakerServer`セクションのサーバのユーザプリンシパルと一致しなければなりません。


### Fault Tolerance

Pacemakerは単一のデーモンインスタンスとして実行されるため、潜在的に単一障害点となります。

クラッシュまたはネットワークの分断によってPacemakerがNimbusから到達不能になった場合、ワーカーは実行を継続し、Nimbusは繰り返し再接続を試みます。 Nimbusの機能は中断されますが、トポロジ自体は引き続き実行されます。
NimbusとPacemakerが分断されたクラスタにおける同じパーティションにある場合、別のパーティションにあるワーカーはハートビートできなくなり、Nimbusはタスクを他の場所で再スケジュールします。どちらの場合であっても望ましい状態になるはずです。


### ZooKeeper Comparison
PacemakerとZooKeeperを比較すると、ノード間の一貫性を保つことによるオーバーヘッドがないため、少ないCPU、少ないメモリ、ディスクは使わずに同じ負荷に対応できます。
ギガビットのネットワーク環境では、約6000ノードという理論上の制限があります。しかし、実際の制限は2000〜3000程度のノードの可能性が高いです。これらの制限は、まだテストされていません。
トポロジーを目一杯スケジュールした270のスーパーバイザクラスタにおいて、4つの`Intel(R) Xeon(R) CPU E5530 @ 2.40GHz`およびRAM 24GiB搭載のマシンで、Pacemakerは1コアを70％、RAMを1Gバイト近く使用していました。


PacemakerをHAにする簡単な方法があります。ZooKeeperとは異なり、Pacemakerはオーバーヘッドなしで水平方向に拡大縮小できるはずです。対照的に、ZooKeeperでは、ZKノードを追加する際のリターンは小さくなります。

要するに、単一のPacemakerノードはZooKeeperクラスタが受けることができる負荷の何倍も処理できるはずで、今後のHA化の取り組みによって水平スケーリングの可能性をさらに増やすことになります。

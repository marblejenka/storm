---
title: Running Apache Storm Securely
layout: documentation
documentation: true
---

# Running Apache Storm Securely

Apache Stormは、クラスタをセキュアにするためのさまざまな設定オプションを提供します
デフォルトでは、すべての認証と認可は無効になっていますが、
必要に応じて有効にすることができます。

## Firewall/OS level Security

正式な認証と認可を有効にすることなく、安全なStormのクラスタを作成できます。
しかし、そうするには、通常、
実行可能な操作を制限するようにオペレーティングシステムを設定する必要があります。
認証と認可が有効なクラスタを運用する予定がある場合でも、これは一般的には良い考えです。

これらの予防措置を設定する方法の正確な詳細は大きく異なり、
このドキュメントのスコープを超えています。

ファイアウォールを有効にして、ネットワークのインバウンドをクラスタそのものや信頼できるホストやサービスだけに制限するのは、
一般的には良い考えです。
Stormが使用するportの完全なリストは以下の通りです。

クラスタが処理しているデータがsensitiveな場合は、
クラスタ内のホスト間で送信されているすべてのトラフィックを暗号化するようにIPsecを設定するのがもっとも望ましいです。

### Ports

| Default Port | Storm Config | Client Hosts/Processes | Server |
|--------------|--------------|------------------------|--------|
| 2181 | `storm.zookeeper.port` | Nimbus, Supervisors, ならびに Worker processes | Zookeeper |
| 6627 | `nimbus.thrift.port` | Storm clients, Supervisors, ならびに UI | Nimbus |
| 8080 | `ui.port` | Client Web Browsers | UI |
| 8000 | `logviewer.port` | Client Web Browsers | Logviewer |
| 3772 | `drpc.port` | External DRPC Clients | DRPC |
| 3773 | `drpc.invocations.port` | Worker Processes | DRPC |
| 3774 | `drpc.http.port` | External HTTP DRPC Clients | DRPC |
| 670{0,1,2,3} | `supervisor.slots.ports` | Worker Processes | Worker Processes |


### UI/Logviewer

UIとlogviewerプロセスは、クラスタが何をしているかを見るだけでなく、
実行中のトポロジを操作する方法を提供します。
通常、これらのプロセスをクラスタのユーザー以外は公開しないでください。

通常、javaサーブレットフィルタとあわせて、いくつかの形式の認証が要求されます。

```yaml
ui.filter: "filter.class"
ui.filter.params: "param1":"value1"
```
あるいは、UIとlogviewerのポートをローカルホストからの接続のみを受け入れるように制限し、受信した接続を認証/許可してStormのプロセスにプロキシできる別のWebサーバー(Apacheのhttpdなど)をフロントにすることもできます。これが機能するようにするには、uiプロセスはstorm.yamlでlogviewer.portをプロキシのポートに設定する必要がありますが、一方で、logviewersはバインドしようとしている実際のポートに設定する必要があります。

サーブレットフィルタを使用することによって、
トポロジごとに関連するページにアクセスすることを許可されている/いないユーザーを指定できるため、
サーブレットフィルタを使用するのが望ましいです。

Storm UIは、hadoop-authからAuthenticationFilterを使用するように設定できます。

```yaml
ui.filter: "org.apache.hadoop.security.authentication.server.AuthenticationFilter"
ui.filter.params:
   "type": "kerberos"
   "kerberos.principal": "HTTP/nimbus.witzend.com"
   "kerberos.keytab": "/vagrant/keytabs/http.keytab"
   "kerberos.name.rules": "RULE:[2:$1@$0]([jt]t@.*EXAMPLE.COM)s/.*/$MAPRED_USER/ RULE:[2:$1@$0]([nd]n@.*EXAMPLE.COM)s/.*/$HDFS_USER/DEFAULT"
```

プリンシパル'HTTP/{hostname}'を作成していることを確認してください(ここでは、ホスト名はUIデーモンが動作する場所にする必要があります)

いったん設定すると、ユーザーはUIにアクセスする前にkinitを行う必要があります。
Once configured users needs to do kinit before accessing UI.
例:
curl  -i --negotiate -u:anyUser  -b ~/cookiejar.txt -c ~/cookiejar.txt  http://storm-ui-hostname:8080/api/v1/cluster/summary

1. Firefox: about:configにいって、network.negotiate-auth.trusted-urisを検索し、ダブルクリックして "http://storm-ui-hostname:8080" を追加してください
2. Google-chrome:  コマンドラインから: google-chrome --auth-server-whitelist="*storm-ui-hostname" --auth-negotiate-delegate-whitelist="*storm-ui-hostname"   
3. IE:  "storm-ui-hostname"を含むように信頼できるWebサイトを設定し、そのWebサイトとのネゴシエーションを許可してください

**注意**: AD MIT Keberosの設定では、キーのサイズがデフォルトのUI jettyサーバーの要求ヘッダーサイズより大きくなっています。storm.yamlでui.header.buffer.bytesを65536に設定してください。詳細は[STORM-633](https://issues.apache.org/jira/browse/STORM-633)を参照してください。


## UI / DRPC SSL 

UIとDRPCの両方で、ユーザーはsslを設定できます。

### UI

UIユーザーは、storm.yamlに次のconfigを設定する必要があります。この手順を実行する前に、適切なキーと証明書を持つキーストアを生成する必要があります。

1. ui.https.port 
2. ui.https.keystore.type (example "jks")
3. ui.https.keystore.path (example "/etc/ssl/storm_keystore.jks")
4. ui.https.keystore.password (keystore password)
5. ui.https.key.password (private key password)

オプション設定

6. ui.https.truststore.path (example "/etc/ssl/storm_truststore.jks")
7. ui.https.truststore.password (truststore password)
8. ui.https.truststore.type (example "jks")

双方向の認証を要求する場合、

9. ui.https.want.client.auth (これをtrueにすると、サーバーがクライアント証明書認証を要求し、認証が提供されていない場合でも接続を維持する)
10. ui.https.need.client.auth (これをtrueにすると、サーバーがクライアントに認証を要求する)




### DRPC
UIと同様に、ユーザーはDRPCのために以下の設定を行う必要があります

1. drpc.https.port 
2. drpc.https.keystore.type (example "jks")
3. drpc.https.keystore.path (example "/etc/ssl/storm_keystore.jks")
4. drpc.https.keystore.password (keystore password)
5. drpc.https.key.password (private key password)

オプション設定

6. drpc.https.truststore.path (example "/etc/ssl/storm_truststore.jks")
7. drpc.https.truststore.password (truststore password)
8. drpc.https.truststore.type (example "jks")

双方向の認証を要求する場合、

9. drpc.https.want.client.auth (これをtrueにすると、サーバーがクライアント証明書認証を要求し、認証が提供されていない場合でも接続を維持する)
10. drpc.https.need.client.auth (これをtrueにすると、サーバーがクライアントに認証を要求する)





## Authentication (Kerberos)

Stormは、thriftとSASLによるプラグイン可能な認証サポートを提供します。
この例は、大部分の大規模なデータプロジェクトの一般的な設定であるため、
Kerberosについてだけ扱います。

各ノードでKDCを設定し、kerberosを設定することは、
このドキュメントのスコープを超えており、すでに行っていることを前提としています。

### Create Headless Principals and keytabs

各Zookeeper Server、Nimbus、およびDRPCサーバーにはサービスプリンシパルが必要です。これには、規約として、実行するホストのFQDNが含まれます。zookeeperのユーザーはzookeeperで*なければならない*ことに注意してください。
スーパーバイザーとUIには、実行するためのプリンシパルが必要ですが、アウトバウンドであるためサービスプリンシパルである必要はありません。
以下は、Kerberosのプリンシパルを設定する方法の例ですが、
詳細はKDCおよびOSによって異なる場合があります。

```bash
# Zookeeper (Will need one of these for each box in teh Zk ensamble)
sudo kadmin.local -q 'addprinc zookeeper/zk1.example.com@STORM.EXAMPLE.COM'
sudo kadmin.local -q "ktadd -k /tmp/zk.keytab  zookeeper/zk1.example.com@STORM.EXAMPLE.COM"
# Nimbus and DRPC
sudo kadmin.local -q 'addprinc storm/storm.example.com@STORM.EXAMPLE.COM'
sudo kadmin.local -q "ktadd -k /tmp/storm.keytab storm/storm.example.com@STORM.EXAMPLE.COM"
# All UI logviewer and Supervisors
sudo kadmin.local -q 'addprinc storm@STORM.EXAMPLE.COM'
sudo kadmin.local -q "ktadd -k /tmp/storm.keytab storm@STORM.EXAMPLE.COM"
```

適切なサーバーにkeytabを配布し、ZKまたはStormを実行しているヘッドレスユーザーのみがアクセスできるようにファイルシステムにおけるアクセス権限を設定してください。

#### Storm Kerberos Configuration

StormとZookeeperはjaas設定ファイルを使用してユーザーをログインさせます。
各jaasファイルには、使用されているインタフェースに応じた複数のセクションがあります。

StormでKerberos認証を有効にするには、次の設定をstorm.yamlにセットする必要があります

```yaml
storm.thrift.transport: "org.apache.storm.security.auth.kerberos.KerberosSaslTransportPlugin"
java.security.auth.login.config: "/path/to/jaas.conf"
```

NimbusとスーパーバイザプロセスもZooKeeper(ZK)に接続し、ZKでの認証にKerberosを使用するように設定したいと考えています。これ行うには、nimbus, ui, and supervisorのchildoptsに以下を追加します。

```
-Djava.security.auth.login.config=/path/to/jaas.conf
```

執筆時点でのデフォルトのchildoptsに設定が与えられた例を次に示します:

```yaml
nimbus.childopts: "-Xmx1024m -Djava.security.auth.login.config=/path/to/jaas.conf"
ui.childopts: "-Xmx768m -Djava.security.auth.login.config=/path/to/jaas.conf"
supervisor.childopts: "-Xmx256m -Djava.security.auth.login.config=/path/to/jaas.conf"
```

Stormのノードについては、jaas.confファイルは次のようになります。
StormServerセクションは、nimbusとDRPCノードで使用されます。supervisorのノードに含める必要はありません。
StormClientセクションは、ui、logviewer、およびsupervisorを含む、nimbusと通信するすべてのStormのクライアントによって使用されます。このセクションはゲートウェイでも使用しますが、その構造は少し異なります。
Clientセクションは、zookeeperと通信するプロセスで使用され、実際にはnimbusとsupervisorに含める必要があります。
Serverセクションは、zookeeperのサーバーによって使用されます。
jaasに未使用セクションがあっても問題になりません。

```
StormServer {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="$keytab"
   storeKey=true
   useTicketCache=false
   principal="$principal";
};
StormClient {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="$keytab"
   storeKey=true
   useTicketCache=false
   serviceName="$nimbus_user"
   principal="$principal";
};
Client {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="$keytab"
   storeKey=true
   useTicketCache=false
   serviceName="zookeeper"
   principal="$principal";
};
Server {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="$keytab"
   storeKey=true
   useTicketCache=false
   principal="$principal";
};
```

以下は、生成されたkeytabに基づく例です

```
StormServer {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="/keytabs/storm.keytab"
   storeKey=true
   useTicketCache=false
   principal="storm/storm.example.com@STORM.EXAMPLE.COM";
};
StormClient {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="/keytabs/storm.keytab"
   storeKey=true
   useTicketCache=false
   serviceName="storm"
   principal="storm@STORM.EXAMPLE.COM";
};
Client {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="/keytabs/storm.keytab"
   storeKey=true
   useTicketCache=false
   serviceName="zookeeper"
   principal="storm@STORM.EXAMPLE.COM";
};
Server {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="/keytabs/zk.keytab"
   storeKey=true
   useTicketCache=false
   serviceName="zookeeper"
   principal="zookeeper/zk1.example.com@STORM.EXAMPLE.COM";
};
```

Nimbusは、プリンシパルをローカルユーザー名に変換して、他のサービスがこの名前を使用できるようにします。Kerberosの認証セットに対してこれを設定するには、

```
storm.principal.tolocal: "org.apache.storm.security.auth.KerberosPrincipalToLocal"
```

これはnimbuでのみ行う必要があり、他のどのノード影響はありません。
また、ZooKeeperの観点からトポロジ、supervisorデーモンとnimbusデーモンが、動作していることを通知する必要があります。

```
storm.zookeeper.superACL: "sasl:${nimbus-user}"
```

ここで、*nimbus-user*は、nimbusがZooKeeperで認証するために使用するKerberosユーザーです。ZooKeeeperがホストとrealmを削除している場合は、このユーザーでもホストとrealmも削除する必要があります。

#### ZooKeeper Ensemble

セキュアなZKを設定する方法の詳細は、このドキュメントの範囲を超えています。しかし、一般的には、各サーバでSASL認証を有効にし、必要に応じてホストとrealmを取り除くはずです。

```
authProvider.1 = org.apache.zookeeper.server.auth.SASLAuthenticationProvider
kerberos.removeHostFromPrincipal = true
kerberos.removeRealmFromPrincipal = true
```

また、サーバーを起動するときにコマンドラインにjaas.confを含めて、keytabが見つかるようにすることもできます。

```
-Djava.security.auth.login.config=/jaas/zk_jaas.conf
```

#### Gateways

理想的には、エンドユーザーはstormと対話する前にkinitを実行する必要だけあります。これをシームレスに行うために、ゲートウェイ上のデフォルトのjaas.confを次のようにする必要があります

```
StormClient {
   com.sun.security.auth.module.Krb5LoginModule required
   doNotPrompt=false
   useTicketCache=true
   serviceName="$nimbus_user";
};
```

エンドユーザーは、keytabを持つヘッドレスユーザーがいる場合、これを上書きすることができます。

### Authorization Setup

*認証*はユーザーが何者なのかを確認する作業ですが、各ユーザーができることを制御するために*認可*も必要です。

nimbusの認可についてのデフォルトのプラグインは*SimpleACLAuthorizer*。です*SimpleACLAuthorizer*を使用するには、次のように設定します:

```yaml
nimbus.authorizer: "org.apache.storm.security.auth.authorizer.SimpleACLAuthorizer"
```

DRPCには別の認可設定があります。DRPCにSimpleACLAuthorizerを使用しないでください。

*SimpleACLAuthorizer*プラグインは、supervisorのユーザが誰であるかを知る必要があり、また、uiデーモンを実行しているユーザを含むすべての管理者ユーザについて知る必要があります。

これらはそれぞれ*nimbus.supervisor.users*と**nimbus.admins*によって設定されます。それぞれは、完全なKerberosプリンシパル名、またはホストとrealmが削除されたユーザーの名前のいずれかです。

ログサーバーには独自の認証設定があります。これらは*logs.users*と*logs.groups*によって設定されます。 れらは、クラスタ内のすべてのノードの管理ユーザーまたはグループに設定する必要があります。

トポロジがsubmitされると、送信側ユーザーはこのリスト内のユーザーも指定できます。指定されたユーザーおよびグループ(ならびにクラスタ全体の設定のユーザーに加えて)には、ログビューアで送信されたトポロジのワーカーログへのアクセスが許可されます。
指定されたユーザーおよびグループ（クラスタ全体の設定のユーザーに加えて）には、submitされたトポロジのワーカーログへのログビューアにおけるアクセス権が与えられます。

### Supervisors headless User and group Setup

マルチテナントでユーザーを分離するためには、supervisorノードでの実行にsupervisorとヘッドレスのユーザとグループを一意してに実行する必要があります。これを有効にするには以下の手順に従ってください。

1. すべてのスーパーバイザホストにヘッドレスユーザを追加する。
2. 一意なグループを作成し、supervisorノード上のヘッドレスユーザのプライマリグループにします。
3. これらのスーパバイザノードのストームに関するプロパティを設定します。

### Multi-tenant Scheduler

マルチテナントをより良くサポートするために、新しいスケジューラを作成しました。このスケジューラーセットを有効にするには、

```yaml
storm.scheduler: "org.apache.storm.scheduler.multitenant.MultitenantScheduler"
```
このスケジューラの機能の多くは、Stormの認証に依存していることに注意してください。それらがなければ、スケジューラーはユーザーが何であるかを知ることができませんし、トポロジを適切に分離できません。

マルチテナントスケジューラの目標は、トポロジそれぞれをを分離する方法を提供することですが、個々のユーザーがクラスタ内に持つことができるリソースも制限することもそうです。

スケジューラーは現在、=storm.yaml=または=storm.yaml=と同じディレクトリーに置かれる=multitenant-scheduler.yaml=という別の設定ファイルを通じて設定できる1つの構成を持っています。nimbusを再起動しなくても更新できるので、=multitenant-scheduler.yaml=を使用することをお勧めします。

=multitenant-scheduler.yaml=における唯一の設定である=multitenant.scheduler.user.pools=は、ユーザー名と、ユーザーがトポロジで使用できることが保証されているノードの最大数の対応付けです。

例えば:

```yaml
multitenant.scheduler.user.pools: 
    "evans": 10
    "derek": 10
```

### Run worker processes as user who submitted the topology
デフォルトでは、Stormはスーパーバイザを実行しているユーザでワーカーを実行します。これはセキュリティにとって理想的ではありません。Stormにトポロジを起動したユーザがでトポロジを実行するようにするには、以下を設定します。

```yaml
supervisor.run.worker.as.user: true
```

上記と合わせて、Stormをセキュアにするために適切に設定する必要があるファイルがいくつかあります。

worker-launcherの実行ファイルは、スーパーバイザが異なるユーザーでワーカーを起動できるようにする特別なプログラムです。これが動作するようにするには、そのファイルがrootによって所有されている必要がありますが、そのグループは、スーパーバイザーのヘッドレスユーザーだけが属するグループに設定されています。
また、6550のパーミッションが必要です。
worker-launcher.cfgファイルもあります。通常は/etc/配下にあり、以下のようになります

```
storm.worker-launcher.group=$(worker_launcher_group)
min.user.id=$(min_user_id)
```
ここで、worker_launcher_groupはスーパーバイザーが属するグループと同じであり、min.user.idはシステム上の最初の実ユーザーIDに設定されています。
この設定ファイルは、rootが所有し、ワールドまたはグループの書き込み権限を持たないようにする必要があります。

### Impersonating a user
Stormのクライアントは、別のユーザに代わってリクエストをsumbitすることがあります。例えば、`userX`がoozieワークフローをsubmitし、ワークフロー実行の一部としてユーザ`oozie`が `userX`のためにトポロジをsubmitしたい場合などです。
偽装機能を利用することでこれが実現できます。他のユーザーとしてトポロジを送信するには、`StormSubmitter.submitTopologyAs`APIを使用できます。
あるいは、`NimbusClient.getConfiguredClientAs`を使用して、他のユーザとしてnimbusクライアントを取得し、このクライアントを使用して任意のニンバスアクション(kill/rebalance/activate/deactivate)を実行できます。

偽装についての認可は、デフォルトでは無効になっています。つまり、すべてのユーザーが偽装を実行できます。許可されたユーザーだけが偽装を実行できるようにするには、 `nimbus.impersonation.authorizer`を`org.apache.storm.security.auth.authorizer.ImpersonationAuthorizer`に設定してnimbusを起動する必要があります。
`ImpersonationAuthorizer`はaclとして`nimbus.impersonation.acl`を使用してユーザーを認証します。以下に、偽装をサポートするサンプルのnimbus設定を示します:

```yaml
nimbus.impersonation.authorizer: org.apache.storm.security.auth.authorizer.ImpersonationAuthorizer
nimbus.impersonation.acl:
    impersonating_user1:
        hosts:
            [comma separated list of hosts from which impersonating_user1 is allowed to impersonate other users]
        groups:
            [comma separated list of groups whose users impersonating_user1 is allowed to impersonate]
    impersonating_user2:
        hosts:
            [comma separated list of hosts from which impersonating_user2 is allowed to impersonate other users]
        groups:
            [comma separated list of groups whose users impersonating_user2 is allowed to impersonate]
```

oozieユースケースをサポートするには、設定を指定する必要があります:

```yaml
nimbus.impersonation.acl:
    oozie:
        hosts:
            [oozie-host1, oozie-host2, 127.0.0.1]
        groups:
            [some-group-that-userX-is-part-of]
```

### Automatic Credentials Push and Renewal
個々のトポロジには、資格情報(チケットとトークン)を安全なサービスにアクセスできるようにプッシュする機能があります。これをすべてのユーザーに公開することは、ユーザーにとって不利益になる可能性があります。
これを一般的な状況で隠すために、プラグインを使用して資格証明を入力し、結果をJava Subjectに展開し、必要に応じてNimbusが資格情報を更新できるようにします。
これらは以下の設定で制御されます。topology.auto-credentialsはjavaプラグインのリストであり、すべてがIAutoCredentialsインターフェースを実装しなければならず、それらはゲートウェイ上のcredentialを生成してワーカー側で解凍します。
Kerberosによるセキュアなクラスタでは、デフォルトでorg.apache.storm.security.auth.kerberos.AutoTGTを指すように設定する必要があります。
nimbus.credential.renewers.classesもこの値に設定して、ユーザーの代わりにnimbusが定期的にTGTを更新できるようにする必要があります。

nimbus.credential.renewers.freq.secsは、何か更新する必要があるかどうかを確認するために更新プログラムがポーリングする頻度を制御しますが、デフォルトでも十分機能します。

さらに、Nimbus自体を使用して、ユーザーがトポロジを送信する代わりに資格情報を取得することもできます。 これは、完全に修飾されたクラス名のリストであるnimbus.autocredential.plugins.classesを使用して設定することができます。これらはすべてINimbusCredentialPluginを実装する必要があります。 Nimbusは、構成されたすべての実装のpopulateCredentialsメソッドをトポロジの送信の一部として呼び出します。 この設定をtopology.auto-credentialsとnimbus.credential.renewers.classesとともに使用すると、資格情報をワーカー側に取り込むことができ、nimbusは自動的にそれらを更新できます。 現時点では、AutoHDFSとAutoHBaseの2つの例があります。これは、hdfsとhbaseの委譲トークンをトポロジサブミッターに自動的に取り込み、すべてのワーカーホストにkeytabを配布する必要がないようにします。

さらに、Nimbus自体を使用して、ユーザーがトポロジをsubmitする代わりにcredentialを取得することもできます。
これは、完全に修飾されたクラス名のリストであるnimbus.autocredential.plugins.classesを使用して設定することができます。これらはすべてINimbusCredentialPluginを実装する必要があります。
Nimbusは、構成されたすべての実装のpopulateCredentialsメソッドをトポロジのsubmitする際に呼び出します。 
この設定をtopology.auto-credentialsとnimbus.credential.renewers.classesとともに使用すると、credentialをワーカー側で取り込むことができ、nimbusは自動的にそれらを更新できます。
現時点では、AutoHDFSとAutoHBaseの2つの例があります。これらは、hdfsとhbaseの委譲トークンをトポロジのsubmitterに自動的に取り込み、すべてのワーカーホストにkeytabを配布する必要がないようにします。

### Limits
デフォルトでは、ストームはあらゆるサイズのトポロジをsumbitすることを許しています。しかし、ZKや他のコンポーネントには、トポロジーが実際にどれだけ大きなものになるかについての限界があります。次の設定では、トポロジの最大サイズを制限できます。

| YAML Setting | Description |
|------------|----------------------|
| nimbus.slots.perTopology | トポロジが使用できるスロット/ワーカーの最大数。 |
| nimbus.executors.perTopology | トポロジが使用できるエグゼキュータ/スレッドの最大数。 |

### Log Cleanup
Logviewerデーモンは、死んだトポロジが出力した古いログファイルをクリーンアップするようになりました。

| YAML Setting | Description |
|--------------|-------------------------------------|
| logviewer.cleanup.age.mins | (最終更新時刻から起算して)そのログがクリーンアップされる前にワーカーのログがどのくらい古くなっていなければならないか(生きているワーカーのログは決してログビューアによってクリーンアップされません。ログはlogbackによってロールされます)。 |
| logviewer.cleanup.interval.secs | ログビューアがワーカーログをクリーンアップする時間間隔(秒単位)。 |


### Allowing specific users or groups to access storm

 SimpleACLAuthorizerを使用すると、kerberosの有効なチケットを持つユーザーは、トポロジをデプロイ、有効化、無効化したり、クラスタ情報にアクセスするなどの操作を行うことができます。
 nimbus.usersまたはnimbus.groupsを指定することで、このアクセスを制限することができます。nimbus.usersが設定されている場合、リスト内のユーザーのみがトポロジをデプロイしたりクラスタにアクセスできます。
 同様にnimbus.groupsは、それらのグループに属するユーザへのStormクラスタへのアクセスを制限します。
 
 設定するにはstorm.yamlに以下の設定を指定してください

```yaml
nimbus.users: 
   - "testuser"
```

あるいは、

```yaml
nimbus.groups: 
   - "storm"
```
 

### DRPC
未記述



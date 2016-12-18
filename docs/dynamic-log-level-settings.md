---
title: Dynamic Log Level Settings
layout: documentation
documentation: true
---

Storm UIとStorm CLIを使用して、実行中のトポロジのログレベル設定を行う機能を追加しています。

ログレベルの設定は、log4jに対するのと同様の方法で適用でき、log4jにログレベルを設定するように指示するだけです。親ロガーのログ・レベルを設定すると、子ロガーはそのレベルを使用します（子がすでにより限定的なレベルになっていない限り）。ワーカーがログレベルを自動的にリセットする必要がある場合、オプションでタイムアウトを指定することができます（DEBUGモードを除く、UIではオプションではなく必須）。

この復帰アクションは、ポーリング（30秒ごとですが、これは設定可能です）を使用してトリガーされます。タイムアウトは、指定した値と0から設定値の間の任意の値になるはずです。

Using the Storm UI
-------------

レベルを設定するには、実行中のトポロジをクリックし、Topology Actionsセクションの「Change Log Level」をクリックします。

![Change Log Level dialog](images/dynamic_log_level_settings_1.png "Change Log Level dialog")

次に、ロガーのを指定し、期待するレベル（たとえばWARN）を選択し、タイムアウトを秒単位で指定します（必要でない場合は0）。次に、「Add」をクリックします。

![After adding a log level setting](images/dynamic_log_level_settings_2.png "After adding a log level setting")

ログレベルを消去するには、「Clear」ボタンをクリックします。これにより、ログレベルが設定を追加する前の状態に戻ります。ログレベルの行はUIから消えます。

ログレベルを元に戻すには遅延がありますが、最初にログレベルを設定するのは即時（メッセージがUI/CLIからnimbusとzookeeper経由でワーカーに伝わる程度の速さ）です。

Using the CLI
-------------

CLIを使用して、次のコマンドを発行します。

`./bin/storm set_log_level [topology name] -l [logger name]=[LEVEL]:[TIMEOUT]`

例えば:

`./bin/storm set_log_level my_topology -l ROOT=DEBUG:30`

ROOTロガーをDEBUGに30秒間設定しています。

`./bin/storm set_log_level my_topology -r ROOT`

ROOTロガーの動的ログレベルをクリアし、元の値にリセットしています。


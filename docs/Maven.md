---
title: Maven
layout: documentation
documentation: true
---
トポロジを開発するには、クラスパス上にStorm jarが必要です。プロジェクトのクラスパスに解凍されたjarを含めるか、Mavenを使用して開発に必要な依存関係としてStormを取り込む必要があります。StormはMaven Centralにホストされています。依存関係としてStormをプロジェクトに組み込むには、pom.xmlに次の行を追加します:


```xml
<dependency>
  <groupId>org.apache.storm</groupId>
  <artifactId>storm-core</artifactId>
  <version>{{page.version}}</version>
  <scope>provided</scope>
</dependency>
```

[これが]({{page.git-blob-base}}/examples/storm-starter/pom.xml)Stormプロジェクトのpom.xmlの例です。

### Developing Storm

詳細については、[DEVELOPER.md]({{page.git-blob-base}}/DEVELOPER.md) を参照してください。

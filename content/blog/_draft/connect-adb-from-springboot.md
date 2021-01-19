+++
title = "Spring Boot ApplicationからAutonomous Transaction Processing（ATP）に接続する"
description = ""
author = "Shuhei, Kawamura"
date = "2020-10-05"
tags = ["Spring Boot", "nomous Transaction Processing Database", "Oracle Cloud Infrastructure", "Doma2"]
categories = ["tech"]
draft = "true"
[[images]]
  src = "img/2020/1004/main.png"
  alt = "main"
  stretch = "stretchH"
+++

- [始めに](#始めに)
- [環境](#環境)
- [手順](#手順)
  - [ATP を作成する](#atp-を作成する)
  - [ウォレットをダウンロードする](#ウォレットをダウンロードする)
  - [アプリケーションで DB の接続設定をする](#アプリケーションで-db-の接続設定をする)
    - [ひな型作成](#ひな型作成)
    - [Doma2 を依存関係に追加](#doma2-を依存関係に追加)
      - [Eclipse 用の設定を追加](#eclipse-用の設定を追加)
    - [DB の接続設定を追加](#db-の接続設定を追加)
  - [起動の確認を行う](#起動の確認を行う)
  - [API の作成](#api-の作成)
- [終わりに](#終わりに)
- [参考](#参考)

# 始めに

Spring Boot アプリケーションから [Autonomous Transaction Processing](https://docs.oracle.com/cd/E83857_01/paas/atp-cloud/index.html)（以下、ATP） に接続する手順を記載します。また、動作確認用に簡単な API も作成したいと思います。OR Mapper として Doma2 を使用していますが、何でも構いません。

# 環境

- Windows 10
- Java 15
  - Spring Boot 2.3.4.RELEASE
  - Doma 2.43.0
- Autonomous Transaction Processing

# 手順

## ATP を作成する

まずは、接続先の DB がないと話にならないので以下のように入力し ATP を作成します。

![img01](https://shukawam.github.io/blog/img/2020/1004/img01.png)

![img02](https://shukawam.github.io/blog/img/2020/1004/img02.png)

- 表示名：My Work（任意の名前で結構です）
- データベース名：MyWork（任意の名前で結構です）
- ワークロード・タイプの選択：トランザクション処理
- デプロイメント・タイプの選択：共有インフラストラクチャ
- データベースの構成：Always Free にチェックを入れておくと良いです
- 管理者資格証明の作成：パスワードポリシーを満たすようなパスワードを入力します
- ネットワーク・アクセスの選択：すべての場所からのセキュア・アクセスを許可
- ライセンスタイプの選択：ライセンス込み

しばらくすると、プロビジョニングが完了し作成した ATP が使用可能になります。

![img03](https://shukawam.github.io/blog/img/2020/1004/img03.png)

## ウォレットをダウンロードする

作成した ATP を選択し、「DB 接続」ボタンを押下すると、ウォレット（資格証明）がダウンロードできるのでダウロードし、適当なディレクトリに展開しておきます。

![img04](https://shukawam.github.io/blog/img/2020/1004/img04.png)

## アプリケーションで DB の接続設定をする

### ひな型作成

[Spring Initializr](https://start.spring.io/)から zip をダウンロードし、任意の IDE に取り込みます。

参考までに今回のアプリケーションのひな型を作成するのに使用した[設定の URL](https://start.spring.io/#!type=gradle-project&language=java&platformVersion=2.3.4.RELEASE&packaging=jar&jvmVersion=15&groupId=com.example&artifactId=springboot-atp-connection-sample&name=springboot-atp-connection-sample&description=Demo%20project%20for%20Spring%20Boot&packageName=com.example.springboot-atp-connection-sample&dependencies=lombok,configuration-processor,devtools,web,jdbc,oracle)を貼っておきます。

### Doma2 を依存関係に追加

`build.gradle`に以下を追加します。

```gradle
plugins {
  // ... 省略
  id 'com.diffplug.eclipse.apt' version '3.22.0' // ... 1
}

// ... 省略

dependencies {
  // ... 省略
  implementation 'org.seasar.doma:doma-core:2.43.0'
  implementation 'org.seasar.doma.boot:doma-spring-boot-starter:1.4.0'
  annotationProcessor 'org.seasar.doma:doma-processor:2.43.0'
}

// ... 省略
```

#### Eclipse 用の設定を追加

本項目は、IDE として Eclipse を使用している場合のみ、必要な項目となります。注釈処理周りの設定を実施してくれるプラグインのため、Eclipse × Doma の組み合わせで開発している方は是非入れましょう。

### DB の接続設定を追加

`src/main/resources/application.properties`に以下のプロパティを追加します。

```properties
# Database connection settings
spring.datasource.driver-class-name=oracle.jdbc.OracleDriver
spring.datasource.url=jdbc:oracle:thin:@<DatabaseName>_medium?TNS_ADMIN=<YourPath>
spring.datasource.username=<Username>
spring.datasource.password=<Password>

# Doma2
doma.dialect=oracle
```

- DatabaseName：先ほど作成した DB の名前を指定します（今回だと`MyWork`）
- YourPath：Wallet ファイルを展開したディレクトリを指定します
- Username：DB のユーザ名を指定します
- Password：指定したユーザのパスワードを指定します

## 起動の確認を行う

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.4.RELEASE)

2020-10-05 22:59:56.496  INFO 17712 --- [  restartedMain] SpringbootAtpConnectionSampleApplication : Starting SpringbootAtpConnectionSampleApplication on DESKTOP-DCQ3PA2 with PID 17712 (C:\Git\springboot-atp-connection-sample\bin\main started by kwmr1 in C:\Git\springboot-atp-connection-sample)
2020-10-05 22:59:56.499  INFO 17712 --- [  restartedMain] SpringbootAtpConnectionSampleApplication : No active profile set, falling back to default profiles: default
2020-10-05 22:59:56.544  INFO 17712 --- [  restartedMain] o.s.b.devtools.restart.ChangeableUrls    : The Class-Path manifest attribute in C:\Users\kwmr1\.gradle\caches\modules-2\files-2.1\com.oracle.database.jdbc\ojdbc8\19.3.0.0\967c0b1a2d5b1435324de34a9b8018d294f8f47b\ojdbc8-19.3.0.0.jar referenced one or more files that do not exist: file:/C:/Users/kwmr1/.gradle/caches/modules-2/files-2.1/com.oracle.database.jdbc/ojdbc8/19.3.0.0/967c0b1a2d5b1435324de34a9b8018d294f8f47b/oraclepki.jar
2020-10-05 22:59:56.544  INFO 17712 --- [  restartedMain] o.s.b.devtools.restart.ChangeableUrls    : The Class-Path manifest attribute in C:\Users\kwmr1\.gradle\caches\modules-2\files-2.1\com.oracle.database.security\oraclepki\19.3.0.0\e52a34f271c6c62ee1a73b71cc19da5459b709f\oraclepki-19.3.0.0.jar referenced one or more files that do not exist: file:/C:/Users/kwmr1/.gradle/caches/modules-2/files-2.1/com.oracle.database.security/oraclepki/19.3.0.0/e52a34f271c6c62ee1a73b71cc19da5459b709f/osdt_core.jar,file:/C:/Users/kwmr1/.gradle/caches/modules-2/files-2.1/com.oracle.database.security/oraclepki/19.3.0.0/e52a34f271c6c62ee1a73b71cc19da5459b709f/osdt_cert.jar,file:/C:/Users/kwmr1/.gradle/caches/modules-2/files-2.1/com.oracle.database.security/oraclepki/19.3.0.0/oracle.osdt/osdt_core.jar,file:/C:/Users/kwmr1/.gradle/caches/modules-2/files-2.1/com.oracle.database.security/oraclepki/19.3.0.0/oracle.osdt/osdt_cert.jar
2020-10-05 22:59:56.544  INFO 17712 --- [  restartedMain] .e.DevToolsPropertyDefaultsPostProcessor : Devtools property defaults active! Set 'spring.devtools.add-properties' to 'false' to disable
2020-10-05 22:59:56.544  INFO 17712 --- [  restartedMain] .e.DevToolsPropertyDefaultsPostProcessor : For additional web related logging consider setting the 'logging.level.web' property to 'DEBUG'
2020-10-05 22:59:57.493  INFO 17712 --- [  restartedMain] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-10-05 22:59:57.502  INFO 17712 --- [  restartedMain] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-10-05 22:59:57.502  INFO 17712 --- [  restartedMain] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.38]
2020-10-05 22:59:57.590  INFO 17712 --- [  restartedMain] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-10-05 22:59:57.590  INFO 17712 --- [  restartedMain] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 1046 ms
2020-10-05 22:59:57.795  INFO 17712 --- [  restartedMain] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-10-05 22:59:58.076  INFO 17712 --- [  restartedMain] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2020-10-05 23:00:29.179 ERROR 17712 --- [  restartedMain] oracle.simplefan.FanManager              : attempt to configure ONS in FanManager failed with oracle.ons.NoServersAvailable: Subscription time out
2020-10-05 23:00:29.204  INFO 17712 --- [  restartedMain] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2020-10-05 23:00:29.273  INFO 17712 --- [  restartedMain] o.s.b.d.a.OptionalLiveReloadServer       : LiveReload server is running on port 35729
2020-10-05 23:00:29.310  INFO 17712 --- [  restartedMain] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-10-05 23:00:29.326  INFO 17712 --- [  restartedMain] SpringbootAtpConnectionSampleApplication : Started SpringbootAtpConnectionSampleApplication in 33.109 seconds (JVM running for 33.881)
```

上記のように表示されれば、Spring Boot アプリケーションから ATP への接続は成功しています。（結構起動までに時間がかかります。大体 30 秒程度）

## API の作成

今回は API を作成することがメインではないので、中身を真似したい場合は、[リポジトリ](https://github.com/shukawam/springboot-atp-connection-sample)を参照してください。

`e2e`に格納している簡易テスト用のスクリプトを実行し、以下のような結果が返却されれば成功です。

```
HTTP/1.1 200
Content-Type: application/json
Transfer-Encoding: chunked
Date: Mon, 05 Oct 2020 14:50:37 GMT
Connection: close

[
  {
    "employeeId": "001",
    "employeeName": "Hillel Slovak"
  },
  {
    "employeeId": "002",
    "employeeName": "Jack Sherman"
  },
  {
    "employeeId": "003",
    "employeeName": "DeWayne McKnigh"
  },
  {
    "employeeId": "004",
    "employeeName": "John Frusciante"
  },
  {
    "employeeId": "005",
    "employeeName": "Arik Marshall"
  },
  {
    "employeeId": "006",
    "employeeName": "Jesse Tobias"
  },
  {
    "employeeId": "007",
    "employeeName": "Dave Navarro"
  },
  {
    "employeeId": "008",
    "employeeName": "Josh Klinghoffer"
  },
  {
    "employeeId": "009",
    "employeeName": "John Frusciante"
  }
]
```

こちらも初回リクエストは、15000ms 程度かかりますね。

余談ですが、返却された従業員のリストの名前は、Red Hot Chili Peppers の歴代のギタリストです！RHCP は最高です。

# 終わりに

公式のドキュメントで[nomous Transaction Processing データベース、SpringBoot および JDBC](https://docs.oracle.com/cd/E60665_01/tutorials_ja/atp_jdbc_springboot/index.html)に設定方法が記載されていますが、xml に設定を記載するのではなく`application.properties`に設定を寄せてみました。同じような設定をしたい方に参考になれば幸いです。

# 参考

- [nomous Transaction Processing データベース、SpringBoot および JDBC](https://docs.oracle.com/cd/E60665_01/tutorials_ja/atp_jdbc_springboot/index.html)

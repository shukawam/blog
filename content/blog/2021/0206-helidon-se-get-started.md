+++
title = "Helidon SE Get Started"
author = "Shuhei, Kawamura"
date = "2021-02-07"
tags = ["java", "helidon"]
categories = ["tech"]
draft = "true"
+++

# 始めに

Java の軽量フレームワークの一つである [Helidon](https://helidon.io/#/) を何回かに分けて全コンポーネントを触ってみたいと思います。尚、[Helidon](https://helidon.io/#/) には

- [Helidon SE](https://helidon.io/docs/v2/#/se/introduction/01_introduction): 軽量でフットプリント重視
- [Helidon MP](https://helidon.io/docs/v2/#/mp/introduction/01_introduction): Eclipse MicroProfile との互換性や機能性を重視

と、エディションが二つ存在しますが、一旦は SE の方に注目して触っていきたいと思います。(SE のコンポーネントを一通り試し終わったら、MP も同様に試していきます。) 今回は、アプリケーションの生成までをやっていきます。

# 環境

一応、私の環境情報を載せておきます。

- OS: Windows 10(WSL2 で Ubuntu 18.04 を使用しています)
- Java: OpenJDK 11

# 手順

## Helidon CLI を使う

アプリケーションのひな型生成や開発モード（ソースコードの変更を再度ビルドする必要なく即時反映してくれる仕組み）をサポートしている便利ツールです。2021/02/06 現在、Windows はまだ CLI の配布がされていないため、ローカルで CLI を使用したい場合は、WSL2 で Ubuntu を使うなどのひと工夫が必要です。

インストール自体は、非常に簡単でバイナリをダウンロードしてパスが通っているところにインストールするだけで大丈夫です。

```bash
$ curl -O https://helidon.io/cli/latest/linux/helidon
$ chmod +x ./helidon
$ sudo mv ./helidon /usr/local/bin/
```

とりあえず、どんなことができるかを見ておきます。

```bash
$ helidon --help

Usage:  helidon [OPTIONS] COMMAND

Helidon Project command line tool

Options:
  -D<name>=<value>    Define a system property
  --verbose           Produce verbose output
  --debug             Produce debug output
  --plain             Do not use color or styles in output

Commands:
  build      Build the application
  dev        Continuous application development
  info       Print project information
  init       Generate a new project
  version    Print version information

Run 'helidon COMMAND --help' for more information on a command.
```

細かいオプションを抜きにして以下の様な操作が可能となっています。

- build
  - アプリケーションのビルドができる
  - `--mode=PLAIN|NATIVE|JLINK`で実行可能 JAR, Native Image(要 GraalVM), jlink を使用したビルドが選択可能
- dev
  - Helidon のアプリケーションを開発モード(ソースコードの変更を即時反映してくれる仕組み)で起動することができる
- info
  - プロジェクトの情報を出力できる
- init
  - プロジェクトの生成ができる
  - 対話形式 or オプションで指定することで用途に合わせたひな型からプロジェクトを生成することができる
- version
  - バージョン情報を出力する

では、さっそくアプリケーションを作ってみたいと思います。まずは、ミニマルな構成で作成し、今後検証したコンポーネントを随時追加していきたいと思います。

```bash
$ helidon init --flavor SE --build MAVEN --archetype bare --groupid shukawam.examples --package shukawam.examples.helidon.se --name helidon-se-examples
Using Helidon version 2.2.0
Helidon flavor
  (1) SE
  (2) MP
Enter selection (Default: 1):
Select archetype
  (1) bare | Minimal Helidon SE project suitable to start from scratch
  (2) quickstart | Sample Helidon SE project that includes multiple REST operations
  (3) database | Helidon SE application that uses the dbclient API with an in-memory H2 database
Enter selection (Default: 1):
Project name (Default: helidon-se-examples):
Project groupId (Default: shukawam.examples):
Project artifactId (Default: helidon-se-examples):
Project version (Default: 1.0-SNAPSHOT):
Java package name (Default: shukawam.examples.helidon.se):
Switch directory to /home/kawamura/git/helidon-se-examples to use CLI

Start development loop? (Default: n):
```

Helidon CLI で生成されたアプリケーションを少し見てみると、以下の様になっています。

```bash
$ cd helidon-se-examples
$ tree
.
├── README.md
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── shukawam
    │   │       └── examples
    │   │           └── helidon
    │   │               └── se
    │   │                   ├── GreetService.java # 自動生成されたサービスクラス(ビジネスロジックを記載するクラス)
    │   │                   ├── Main.java # Mainクラス(サーバの起動やルーティングの設定を行うクラス)
    │   │                   └── package-info.java
    │   └── resources
    │       ├── META-INF
    │       │   └── native-image
    │       │       └── reflect-config.json # Native Image 化する際の設定ファイル(Reflection API 関連の設定ファイルです。詳細は、https://www.graalvm.org/reference-manual/native-image/Reflection/をどうぞ)
    │       ├── application.yaml # Application の設定ファイル
    │       └── logging.properties # ログの設定ファイル
    └── test
        └── java
            └── shukawam
                └── examples
                    └── helidon
                        └── se
                            └── MainTest.java # テストコード

16 directories, 9 files
```

開発モードで起動してみます。

```bash
$ helidon dev
helidon dev

| source file changed
| rebuilding (incremental)
| rebuild completed (0.3 seconds)
| helidon-se-examples starting

2021.02.06 23:10:47 INFO io.helidon.common.LogConfig Thread[main,5,main]: Logging at initialization configured using classpath: /logging.properties
2021.02.06 23:10:48 INFO io.helidon.common.HelidonFeatures Thread[features-thread,5,main]: Helidon SE 2.2.0 features: [Config, Health, Metrics, WebServer]
2021.02.06 23:10:48 INFO io.helidon.webserver.NettyWebServer Thread[nioEventLoopGroup-2-1,10,main]: Channel '@default' started: [id: 0x930f8a6a, L:/0:0:0:0:0:0:0:0:8080]
WEB server is up! http://localhost:8080 in 1017 milliseconds (since JVM startup).
```

JVM が起動してからアプリケーションが起動するまでの時間を計測したいので以下のログステートメントを追記しています。

```java
System.out.println(
    String.format(
      "WEB server is up! http://localhost:%s in %s milliseconds (since JVM startup).", // 追記
      ws.port(),
      ManagementFactory.getRuntimeMXBean().getUptime()
    )
);
```

折角なので、Native Image も作ってみます。今回は GraalVM の環境を作らずに Native Image を作成するため、以下の様な Dockerfile を用意します。

**Dockerfile.native**

```docker
# 1st stage, build the app
FROM helidon/jdk11-graalvm-maven:20.2.0 as build

WORKDIR /helidon

# Create a first layer to cache the "Maven World" in the local repository.
# Incremental docker builds will always resume after that, unless you update
# the pom
ADD pom.xml .
RUN mvn package -Pnative-image -Dnative.image.skip -DskipTests

# Do the Maven build!
# Incremental docker builds will resume here when you change sources
ADD src src
RUN mvn package -Pnative-image -Dnative.image.buildStatic -DskipTests

RUN echo "done!"

# 2nd stage, build the runtime image
FROM scratch
WORKDIR /helidon

# Copy the binary built in the 1st stage
COPY --from=build /helidon/target/helidon-se-examples .

ENTRYPOINT ["./helidon-se-examples"]

EXPOSE 8080
```

Native Image を生成します。(手元の環境に GraalVM のランタイムが入っている場合は、`Helidon build --mode NATIVE`で Native Image のビルドができます。)

```bash
$ docker build -t shukawam/helidon-se-native-image:v1.0 -f Dockerfile.native .
```

起動してみます。

```bash
$ docker run -p 8080:8080 shukawam/helidon-se-native-image:v1.0
02021.02.06 15:13:23 INFO io.helidon.common.LogConfig Thread[main,5,main]: Logging at runtime configured using classpath: /logging.properties
2021.02.06 15:13:23 INFO io.helidon.common.HelidonFeatures Thread[features-thread,5,main]: Helidon SE 2.2.0 features: [Config, Health, Metrics, WebServer]
2021.02.06 15:13:23 INFO io.helidon.webserver.NettyWebServer Thread[nioEventLoopGroup-2-1,10,main]: Channel '@default' started: [id: 0x4c520a0c, L:/0.0.0.0:8080]
WEB server is up! http://localhost:8080 in 18 milliseconds (since JVM startup).
```

18ms...さすがに、起動は桁違いに速いです。

ちなみに、自動生成された`pom.xml`内の依存関係は以下の様になっていました。

`pom.xml`

```xml
<dependencies>
    <dependency>
        <groupId>io.helidon.webserver</groupId>
        <artifactId>helidon-webserver</artifactId>
    </dependency>
    <dependency>
        <groupId>io.helidon.media</groupId>
        <artifactId>helidon-media-jsonp</artifactId>
    </dependency>
    <dependency>
        <groupId>io.helidon.config</groupId>
        <artifactId>helidon-config-yaml</artifactId>
    </dependency>
    <dependency>
        <groupId>io.helidon.health</groupId>
        <artifactId>helidon-health</artifactId>
    </dependency>
    <dependency>
        <groupId>io.helidon.health</groupId>
        <artifactId>helidon-health-checks</artifactId>
    </dependency>
    <dependency>
        <groupId>io.helidon.metrics</groupId>
        <artifactId>helidon-metrics</artifactId>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-api</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.helidon.webclient</groupId>
        <artifactId>helidon-webclient</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

マイクロサービスの実装に必要不可欠な Metrics, Health Check, Config といったライブラリは、既に含まれているので特に設定することなく、Metrics や Health Check のエンドポイントを利用可能になっています。

```bash
# Health Check
$ curl http://localhost:8080/health
{"outcome":"UP","status":"UP","checks":[{"name":"deadlock","state":"UP","status":"UP"},{"name":"diskSpace","state":"UP","status":"UP","data":{"free":"228.23 GB","freeBytes":245060718592,"percentFree":"90.93%","total":"250.98 GB","totalBytes":269490393088}},{"name":"heapMemory","state":"UP","status":"UP","data":{"free":"68.51 MB","freeBytes":71841744,"max":"1.54 GB","maxBytes":1648361472,"percentFree":"97.87%","total":"102.00 MB","totalBytes":106954752}}]}

# Metrics
$ curl http://localhost:8080/metrics --header 'Accept:application/json'
{"base":{"classloader.loadedClasses.count":3881,"classloader.loadedClasses.total":3884,"classloader.unloadedClasses.total":0,"cpu.availableProcessors":8,"cpu.systemLoadAverage":0.26,"gc.time;name=G1 Old Generation":0,"gc.time;name=G1 Young Generation":17,"gc.total;name=G1 Old Generation":0,"gc.total;name=G1 Young Generation":1,"jvm.uptime":118654,"memory.committedHeap":106954752,"memory.maxHeap":1648361472,"memory.usedHeap":38256688,"thread.count":20,"thread.daemon.count":15,"thread.max.count":20},"vendor":{"requests.count":2,"requests.meter":{"count":2,"meanRate":0.016974990693588664,"oneMinRate":0.006394042634681005,"fiveMinRate":0.002751971906724164,"fifteenMinRate":0.0010423449272394232}}}
```

## Maven を使う

以下、Maven, Gradle を使う場合についても参考程度に載せておきます。

プロジェクトを生成します。

```bash
$ mvn -U archetype:generate -DinteractiveMode=false \
    -DarchetypeGroupId=io.helidon.archetypes \
    -DarchetypeArtifactId=helidon-quickstart-se \
    -DarchetypeVersion=2.2.0 \
    -DgroupId=io.helidon.examples \
    -DartifactId=helidon-quickstart-se \
    -Dpackage=io.helidon.examples.quickstart.se
```

中身を覗いてみると以下のようになっています。

```bash
$ tree
.
├── Dockerfile # 実行可能JARを生成するDockerfile
├── Dockerfile.jlink # jlinkを使用してビルドするDockerfile
├── Dockerfile.native # Native Imageを生成するDockerfile
├── README.md
├── app.yaml # Kubernetesのマニフェストファイル
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── io
    │   │       └── helidon
    │   │           └── examples
    │   │               └── quickstart
    │   │                   └── se
    │   │                       ├── GreetService.java
    │   │                       ├── Main.java
    │   │                       └── package-info.java
    │   └── resources
    │       ├── META-INF
    │       │   └── native-image
    │       │       └── reflect-config.json
    │       ├── application.yaml
    │       └── logging.properties
    └── test
        └── java
            └── io
                └── helidon
                    └── examples
                        └── quickstart
                            └── se
                                └── MainTest.java

18 directories, 13 files
```

CLI から生成した時には、Kubernetes のマニフェストファイルや各種 Dockerfile は生成されなかったですが、これは便利そうですね。残りは、全て CLI から生成した場合と同じでした。

## Gradle を使う

Helidon のサンプルはほとんどが Maven を使用していますが、Gradle も同様に使用することができます。(推奨は、Gradle 6+)Gradle を使用してビルドしたい場合は、[Helidon SE の QuickStart](https://github.com/oracle/helidon/tree/2.2.0/examples/quickstarts/helidon-quickstart-se)に含まれる`build.gradle`を参考にするとよいです。

`build.gradle`

```gradle
/*
 * Copyright (c) 2018, 2020 Oracle and/or its affiliates. All rights reserved.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

apply plugin: 'java'

group = 'io.helidon.examples' // 自分のアプリケーションに合わせて修正する
version = '1.0-SNAPSHOT'

description = "helidon-quickstart-se" // 自分のアプリケーションに合わせて修正する

sourceCompatibility = 11
targetCompatibility = 11
tasks.withType(JavaCompile) {
    options.encoding = 'UTF-8'
}

ext {
    helidonversion = '2.2.0'
}

test {
    useJUnitPlatform()
}

repositories {
    mavenCentral()
    mavenLocal()
}

dependencies {
    // import Helidon BOM
    implementation enforcedPlatform("io.helidon:helidon-dependencies:${project.helidonversion}")
    implementation 'io.helidon.webserver:helidon-webserver'
    implementation 'io.helidon.media:helidon-media-jsonp'
    implementation 'io.helidon.config:helidon-config-yaml'
    implementation 'io.helidon.health:helidon-health'
    implementation 'io.helidon.health:helidon-health-checks'
    implementation 'io.helidon.metrics:helidon-metrics'

    testImplementation 'org.junit.jupiter:junit-jupiter-api'
    testImplementation 'io.helidon.webclient:helidon-webclient'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine'
}

// define a custom task to copy all dependencies in the runtime classpath
// into build/libs/libs
// uses built-in Copy
task copyLibs(type: Copy) {
  from configurations.runtimeClasspath
  into 'build/libs/libs'
}

// add it as a dependency of built-in task 'assemble'
copyLibs.dependsOn jar
assemble.dependsOn copyLibs

// default jar configuration
// set the main classpath
// add each jar under build/libs/libs into the classpath
jar {
  archiveFileName = "${project.name}.jar"
  manifest {
    attributes ('Main-Class': 'io.helidon.examples.quickstart.se.Main', // 自分のアプリケーションの構造に合わせて修正してください
                'Class-Path': configurations.runtimeClasspath.files.collect { "libs/$it.name" }.join(' ')
               )
  }
}

```

ビルドはおなじみのコマンドで。

```bash
$ gradle build
```

# 終わりに

今回は、プロジェクトの生成までを一通りやってみました。次回以降[公式ドキュメント](https://helidon.io/docs/v2/#/se/introduction/01_introduction)に記載されているコンポーネントを順番に試していきたいと思います。

今回作成したソースコードはこちらのリポジトリに格納しています。

# 参考

- [Helidon 公式ドキュメント](https://helidon.io/#/)

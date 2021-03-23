+++
title = "Spring Native Getting Started"
author = "Shuhei, Kawamura"
date = "2021-03-22"
tags = ["Java", "Spring", "GraalVM"]
categories = ["tech"]
draft = "false"
[[images]]
  src = "img/2021/0323/OG-Spring.png"
  alt = ""
  stretch = "stretchH"
+++

# 始めに

[Spring Native](https://docs.spring.io/spring-native/docs/current/reference/htmlsingle/)を使って、Spring Boot アプリケーションを Native Image 化してみます。

# 手順

[Spring Initializr](https://start.spring.io/) を使用してひな形を作成します。今回は、Helath Check のエンドポイントを一つ有するアプリケーションを作成します。

```bash
curl -G https://start.spring.io/starter.zip -o spring-boot-native-image-sample.zip -d javaVersion=11 -d dependencies=web,actuator,native
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 57164  100 57164    0     0   106k      0 --:--:-- --:--:-- --:--:--  106k
```

解凍します。

```bash
unzip spring-boot-native-image-sample.zip -d spring-boot-native-image
Archive:  spring-boot-native-image-sample.zip
  inflating: spring-boot-native-image/.gitignore
  inflating: spring-boot-native-image/HELP.md
  inflating: spring-boot-native-image/mvnw
   creating: spring-boot-native-image/src/
   creating: spring-boot-native-image/src/main/
   creating: spring-boot-native-image/src/main/java/
   creating: spring-boot-native-image/src/main/java/com/
   creating: spring-boot-native-image/src/main/java/com/example/
   creating: spring-boot-native-image/src/main/java/com/example/demo/
  inflating: spring-boot-native-image/src/main/java/com/example/demo/DemoApplication.java
   creating: spring-boot-native-image/src/main/resources/
   creating: spring-boot-native-image/src/main/resources/templates/
   creating: spring-boot-native-image/src/main/resources/static/
  inflating: spring-boot-native-image/src/main/resources/application.properties
   creating: spring-boot-native-image/src/test/
   creating: spring-boot-native-image/src/test/java/
   creating: spring-boot-native-image/src/test/java/com/
   creating: spring-boot-native-image/src/test/java/com/example/
   creating: spring-boot-native-image/src/test/java/com/example/demo/
  inflating: spring-boot-native-image/src/test/java/com/example/demo/DemoApplicationTests.java
   creating: spring-boot-native-image/.mvn/
   creating: spring-boot-native-image/.mvn/wrapper/
  inflating: spring-boot-native-image/.mvn/wrapper/maven-wrapper.jar
  inflating: spring-boot-native-image/.mvn/wrapper/MavenWrapperDownloader.java
  inflating: spring-boot-native-image/.mvn/wrapper/maven-wrapper.properties
  inflating: spring-boot-native-image/pom.xml
  inflating: spring-boot-native-image/mvnw.cmd
```

Native Image を生成します。

```bash
cd spring-boot-native-image/
mvn spring-boot:build-image
```

しばらく待つと、Docker Image が生成される。

```bash
docker images | grep demo
demo                            0.0.1-SNAPSHOT      4c417a6969fa   41 years ago        90MB
```

ちなみに、CREATED が 41 years ago になるのは仕様らしいです。(ちょっと気持ち悪い)

ref: [https://stackoverflow.com/questions/62865594/spring-boot-2-3-0-buildpack-builds-image-with-creation-date-40-years-ago](https://stackoverflow.com/questions/62865594/spring-boot-2-3-0-buildpack-builds-image-with-creation-date-40-years-ago)

> This is expected. In order to create reproducible builds (i.e. so that layers can be reused) the buildpack must create layers with a fixed time stamp. Otherwise, you wouldn’t be able to reuse the layers you created in previous builds because they would have different time stamps.

また、比較のために Fat JAR の Docker Image も生成しておきます。

```bash
docker build -t shukawam/spring-boot-fat-jar:v1.0 -f Dockerfile ./
docker images | grep shukawam/spring-boot-fat-jar
shukawam/spring-boot-fat-jar    v1.0                ccf63974666c   31 seconds ago      239MB
```

両方とも起動の時間を確認してみます。まずは、Fat JAR から。

```bash
docker run --rm -p 8080:8080 shukawam/spring-boot-fat-jar:v1.0
2021-03-23 09:46:02.644  INFO 1 --- [           main] o.s.nativex.NativeListener               : This application is bootstrapped with code generated with Spring AOT

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.4.4)

2021-03-23 09:46:02.737  INFO 1 --- [           main] com.example.demo.DemoApplication         : Starting DemoApplication v0.0.1-SNAPSHOT using Java 11.0.10 on 2e42d385fb33 with PID 1 (/spring-boot/demo-0.0.1-SNAPSHOT.jar started by root in /spring-boot)
2021-03-23 09:46:02.738  INFO 1 --- [           main] com.example.demo.DemoApplication         : No active profile set, falling back to default profiles: default
2021-03-23 09:46:03.933  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2021-03-23 09:46:03.945  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2021-03-23 09:46:03.945  INFO 1 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.44]
2021-03-23 09:46:04.002  INFO 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2021-03-23 09:46:04.002  INFO 1 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 1204 ms
2021-03-23 09:46:04.578  INFO 1 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2021-03-23 09:46:04.819  INFO 1 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 2 endpoint(s) beneath base path '/actuator'
2021-03-23 09:46:04.865  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2021-03-23 09:46:04.879  INFO 1 --- [           main] com.example.demo.DemoApplication         : Started DemoApplication in 2.605 seconds (JVM running for 3.05)
2021-03-23 09:46:10.908  INFO 1 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2021-03-23 09:46:10.909  INFO 1 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2021-03-23 09:46:10.910  INFO 1 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 1 ms
```

JVM が起動してから大体 2.6 秒です。お次に、Native Image 版。

```bash
docker run -p 8080:8080 docker.io/library/demo:0.0.1-SNAPSHOT
2021-03-23 09:47:57.657  INFO 1 --- [           main] o.s.nativex.NativeListener               : This application is bootstrapped with code generated with Spring AOT

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.4.4)

2021-03-23 09:47:57.658  INFO 1 --- [           main] com.example.demo.DemoApplication         : Starting DemoApplication using Java 11.0.10 on 26edf77d8c85 with PID 1 (/workspace/com.example.demo.DemoApplication started by cnb in /workspace)
2021-03-23 09:47:57.658  INFO 1 --- [           main] com.example.demo.DemoApplication         : No active profile set, falling back to default profiles: default
2021-03-23 09:47:57.697  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
Mar 23, 2021 9:47:57 AM org.apache.coyote.AbstractProtocol init
INFO: Initializing ProtocolHandler ["http-nio-8080"]
Mar 23, 2021 9:47:57 AM org.apache.catalina.core.StandardService startInternal
INFO: Starting service [Tomcat]
Mar 23, 2021 9:47:57 AM org.apache.catalina.core.StandardEngine startInternal
INFO: Starting Servlet engine: [Apache Tomcat/9.0.44]
Mar 23, 2021 9:47:57 AM org.apache.catalina.core.ApplicationContext log
INFO: Initializing Spring embedded WebApplicationContext
2021-03-23 09:47:57.699  INFO 1 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 41 ms
2021-03-23 09:47:57.701  WARN 1 --- [           main] i.m.c.i.binder.jvm.JvmGcMetrics          : GC notifications will not be available because MemoryPoolMXBeans are not provided by the JVM
2021-03-23 09:47:57.714  INFO 1 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2021-03-23 09:47:57.726  INFO 1 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 2 endpoint(s) beneath base path '/actuator'
Mar 23, 2021 9:47:57 AM org.apache.coyote.AbstractProtocol start
INFO: Starting ProtocolHandler ["http-nio-8080"]
2021-03-23 09:47:57.728  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2021-03-23 09:47:57.729  INFO 1 --- [           main] com.example.demo.DemoApplication         : Started DemoApplication in 0.076 seconds (JVM running for 0.077)
Mar 23, 2021 9:48:01 AM org.apache.catalina.core.ApplicationContext log
INFO: Initializing Spring DispatcherServlet 'dispatcherServlet'
2021-03-23 09:48:01.198  INFO 1 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2021-03-23 09:48:01.198  INFO 1 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
```

ログには、`Started DemoApplication in 0.076 seconds (JVM running for 0.077)`と書いてありますが、内部では JVM の起動を介さずに直接 Machine Code が実行されているので、少し誤解を生んでしまうログ出力かもしれません。細かいことは置いておいて、起動に要した時間は大体 0.07 秒といった所です。Fat JAR に比べると 37 倍も速いですね。また、イメージの容量に関しても 1/2 以下となっています。

## おまけ

Fat JAR 生成に使用した Dockerfile は以下

```Dockerfile
# 1st stage, build the app
FROM maven:3.6-jdk-11 as build

WORKDIR /spring-boot

# Create a first layer to cache the "Maven World" in the local repository.
# Incremental docker builds will always resume after that, unless you update
# the pom
ADD pom.xml .
ADD src src
RUN mvn package

# 2nd stage, build the runtime image
FROM openjdk:11-jre-slim
WORKDIR /spring-boot

# Copy the binary built in the 1st stage
COPY --from=build /spring-boot/target/demo-0.0.1-SNAPSHOT.jar ./

CMD ["java", "-jar", "demo-0.0.1-SNAPSHOT.jar"]

EXPOSE 8080
```

# 終わりに

FaaS 上で Spring アプリケーションが動く日もそんな遠い話じゃないかもなと感じました。(プラットフォームが GraalVM をサポートしていればの話ですが)
ちなみに、サポートについては[Spring Native - 3. Support](https://docs.spring.io/spring-native/docs/current/reference/htmlsingle/#support)を参照すると、どのコンポーネントが対応／非対応かが記されています。

# 参考

- [Spring Native documentation](https://docs.spring.io/spring-native/docs/current/reference/htmlsingle/)

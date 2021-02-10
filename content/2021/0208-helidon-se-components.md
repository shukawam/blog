+++
title = "Helidon SE Component - Config, DB Client, OpenAPI"
author = "Shuhei, Kawamura"
date = "2021-02-08"
tags = ["java", "helidon", "helidon se"]
categories = ["tech"]
draft = "true"
[[images]]
  src = "img/2021/0206/helidon.jpg"
  alt = "helidon.jpg"
  stretch = "stretchH"
+++

# 始めに

Java の軽量フレームワークの一つである[Helidon](https://helidon.io/#/)の全コンポーネントを触っていくエントリーの一回目です。本日は、

- Config
- DB Client
- OpenAPI

について触っていこうと思います。

# Components

## [Config](https://helidon.io/docs/v2/#/se/config/01_introduction)

- 様々なソース(`.properties`, `.yaml`, `.json`)から Config オブジェクトに設定プロパティをロードして処理するための Java API を提供します
- 各種設定情報をソースコードから分離することでコード自体の保守性を向上させたりする目的があります

早速、試していきます。CLI を使用してひな形を生成した場合は、自動的に Config 関連の依存関係が含まれていますが、含まれていない場合は以下を`pom.xml`に追加します。

`pom.xml`

```xml
<dependency>
    <groupId>io.helidon.config</groupId>
    <artifactId>helidon-config-yaml</artifactId>
</dependency>
```

今回は、`yaml`形式で記載したいと思います。`.json`でも`.properties`でも結果は同じですが個人的に`yaml`が好きだからです。（一番無駄がないし、視認性もよい）

`resources/application.yaml`に Config オブジェクトにマッピングさせる情報を記載していきます。

`application.yaml`

```yaml
app:
  greeting: "Hello"
  config: "config service works!!" # 追記

server:
  port: 8080
  host: 0.0.0.0
```

`application.yaml`に記載した項目を参照するには以下のようにします。

```java
var config = Config.create();
var value = config.get("app.config").asString().orElse("config service NOT work");
```

## [DB Client](https://helidon.io/docs/v2/#/se/dbclient/01_introduction)

- ノンブロッキングでデータベースを操作するためのリアクティブ API を提供している
- TODO: なんか書く

それでは、試していきます。まずは`pom.xml`に以下を追記します。

`pom.xml`

```xml
<!-- Helidon の DB Client -->
<dependency>
    <groupId>io.helidon.dbclient</groupId>
    <artifactId>helidon-dbclient</artifactId>
</dependency>
<!-- JDBC -->
<dependency>
    <groupId>io.helidon.dbclient</groupId>
    <artifactId>helidon-dbclient-jdbc</artifactId>
</dependency>
<!-- MongoDB -->
<!-- <dependency>
    <groupId>io.helidon.dbclient</groupId>
    <artifactId>helidon-dbclient-mongodb</artifactId>
</dependency> -->
<!-- JDBCを使用する場合は製品に合わせたドライバを依存関係に含める -->
<dependency>
  <groupId>com.oracle.database.jdbc</groupId>
  <artifactId>ojdbc8</artifactId>
  <version>21.1.0.0</version>
</dependency>
```

今回は、以下のデータベースに対して接続するサンプルコードを実装しようと思います。

- Autonomous Transaction Processing(ATP)
- MongoDB

### ATP

`application.yaml`に以下の接続情報を追記します。

```yaml
db:
  source: "jdbc" # source: jdbc or mongodb
  connection:
    url: "jdbc:oracle:thin@sandbox_high?TNS_ADMIN=<wallet-file-dir>"
    username: "<username>"
    password: "<password>"
    statements:
      ping: "DO 0"
      select-all-items: "SELECT * FROM ITEM"
```

### MongoDB

## [OpenAPI](https://helidon.io/docs/v2/#/se/openapi/01_openapi)

# 終わりに

# 参考

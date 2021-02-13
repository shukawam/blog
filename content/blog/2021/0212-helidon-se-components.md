+++
title = "Helidon SE Component - Config, OpenAPI"
author = "Shuhei, Kawamura"
date = "2021-02-08"
tags = ["java", "helidon", "helidon se"]
categories = ["tech"]
draft = "false"
[[images]]
  src = "img/2021/0206/helidon.jpg"
  alt = "helidon.jpg"
  stretch = "stretchH"
+++

# 始めに

Java の軽量フレームワークの一つである[Helidon](https://helidon.io/#/)の全コンポーネントを触っていくエントリーの一回目です。本日は、

- Config
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

今回は、`yaml`形式で記載したいと思います。`json`でも`properties`でも結果は同じですが個人的に`yaml`が好きだからです。（一番無駄がないし、視認性もよい）

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

## [OpenAPI](https://helidon.io/docs/v2/#/se/openapi/01_openapi)

- OpenAPI 仕様のドキュメントを生成するエンドポイントを容易に生成することができる
- Eclipse MicroProfile の OpenAPI 仕様となっているので、実装などで困った際は Helidon のドキュメントではなく、MicroProfile の OpenAPI 仕様のドキュメントを参考にするとよい

早速、試していきます。まずは、OpenAPI 関連の依存関係を`pom.xml`に追加します。

`pom.xml`

```xml
<dependency>
    <groupId>io.helidon.openapi</groupId>
    <artifactId>helidon-openapi</artifactId>
    <version>2.2.1</version>
</dependency>
```

`OpenAPISupport`をサーバの設定に追加します。

```java
return Routing.builder()
    // ...
    // OpenAPI support
    .register(OpenAPISupport.create(config))
    .build();
```

OpenAPI ドキュメントを構築するためにいくつか手段があります。

1. OpenAPI の定義ファイルを作成する

   - META-INF/openapi.yml, META-INF/openapi.yaml, META-INF/openapi.json に定義を追加する

2. `org.eclipse.microprofile.openapi.OASModelReader`インタフェースを実装した Java クラスに定義を記述する

### 1. OpenAPI の定義ファイルを作成する

今回は、以下のようなサンプルファイルを生成しました。

`openapi.yaml`

```yaml
openapi: 3.0.0
info:
  title: Helidon SE OpenAPI Example.
  description: helidon se openapi example.
  version: 1.0.0

servers:
  - url: http://localhost:8080
    description: Local server.

components:
  schemas:
    GreetingMessage:
      properties:
        message:
          type: string
    ConfigMessage:
      properties:
        message:
          type: string

paths:
  /greet:
    get:
      summary: Returns a generic greeting
      description: Greets the user generically
      responses:
        "200":
          description: OK
          content:
            application/json:
              examples:
                Greet:
                  value:
                    message: Hello World!
  /config:
    get:
      summary: Returns a config properties.
      description: return a config properties.
      responses:
        "200":
          description: OK
          content:
            application/json:
              examples:
                Config:
                  value:
                    message: config service works!!
```

API 仕様のエンドポイントは`/openapi`となっているため、特にヘッダー情報を追加せずにリクエストを送ると`YAML`形式のデータ(上記の定義ファイルと同じ内容)を返却します。

```yaml
servers:
  - description: Local server.
    url: http://localhost:8080
components:
  schemas:
    GreetingMessage:
      properties:
        message:
          type: string
    ConfigMessage:
      properties:
        message:
          type: string
info:
  description: helidon se openapi example.
  title: Helidon SE OpenAPI Example.
  version: 1.0.0
openapi: 3.0.0
paths:
  /greet:
    get:
      description: Greets the user generically
      responses:
        "200":
          content:
            application/json:
              examples:
                Greet:
                  value:
                    message: Hello World!
          description: OK
      summary: Returns a generic greeting
  /config:
    get:
      description: return a config properties.
      responses:
        "200":
          content:
            application/json:
              examples:
                Config:
                  value:
                    message: config service works!!
          description: OK
      summary: Returns a config properties.
```

ちなみに、ドキュメントを取得するパスを変更したい場合は以下のように設定を変更します。

`application.yaml`

```yaml
openapi:
  web-context: /myopenapi # 任意のパスに変更可能
```

### 2. `org.eclipse.microprofile.openapi.OASModelReader`インタフェースを実装した Java クラスに定義を記述する

まずは、設定ファイルに`OASModelReader`を使用することを記述します。具体的には以下のように。

`application.yaml`

```yaml
# OpenAPI configuration.
openapi:
  model:
    reader: shukawam.examples.helidon.se.openapi.OpenAPIModelReader
```

さて、実装クラスですが今回は以下のように実装してみました。

```java
public class OpenAPIModelReader implements OASModelReader { // OASModelReaderの実装クラスを作成する
    private static final String MODEL_READER_PATH = "/openapi/test";
    private static final String DESCRIPTION = "A sample endpoint from OASModelReader.";

    @Override
    public OpenAPI buildModel() {
        PathItem pathItem = OASFactory.createPathItem()
                .GET(OASFactory.createOperation()
                        .operationId("test")
                        .description(DESCRIPTION)
                );
        OpenAPI openAPI = OASFactory.createOpenAPI();
        Paths paths = OASFactory.createPaths()
                .addPathItem(MODEL_READER_PATH, pathItem);
        openAPI.paths(paths);
        return openAPI;
    }
}
```

そのうえで先ほどのエンドポイントに対してリクエストを送ってみると、

```yaml
servers:
  - description: Local server.
    url: http://localhost:8080
components:
  schemas:
    GreetingMessage:
      properties:
        message:
          type: string
    ConfigMessage:
      properties:
        message:
          type: string
info:
  description: helidon se openapi example.
  title: Helidon SE OpenAPI Example.
  version: 1.0.0
openapi: 3.0.0
paths:
  /openapi/test: # OASModelReaderの実装クラスで追加した情報が返却される
    get:
      description: A sample endpoint from OASModelReader.
      operationId: test
  /greet:
    get:
      description: Greets the user generically
      responses:
        "200":
          content:
            application/json:
              examples:
                Greet:
                  value:
                    message: Hello World!
          description: OK
      summary: Returns a generic greeting
  /config:
    get:
      description: return a config properties.
      responses:
        "200":
          content:
            application/json:
              examples:
                Config:
                  value:
                    message: config service works!!
          description: OK
      summary: Returns a config properties.
```

これに加え、`OASFilter`を実装したクラスを作成し、適切な処理を実装すれば OpenAPI が内部モデルを構築する前に任意の処理を挟みこむことができます。

# 終わりに

今回は、Config, OpenAPI のコンポーネントについて色々試してみました。次回は、[DB Client](https://helidon.io/docs/v2/#/se/dbclient/01_introduction)でも試してみようかと思います。

# 参考

- [Helidon SE Component - Config](https://helidon.io/docs/v2/#/se/config/01_introduction)

- [Helidon SE Component - OpenAPI](https://helidon.io/docs/v2/#/se/openapi/01_openapi)

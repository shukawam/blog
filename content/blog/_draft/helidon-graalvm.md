+++
title = "Javaのマイクロサービスフレームワーク\"Helidon\"を色々試した"
description = ""
author = "Shuhei, Kawamura"
date = "2021-01-24"
tags = ["Helidon", "Java", "GraalVM", "Microservices"]
categories = ["tech"]
draft = "true"
[[images]]
  src = "img/2021/0124/helidon.jpg"
  alt = "helidon"
  stretch = "stretchH"
+++

# 始めに

転職、引っ越しで忙しかったのもあって、前に更新した時から随分と日が経ってしまいました。
さて今回は、Java のマイクロサービスフレームワーク Helidon を色々試してみたいと思います。

また、検証に使用したコードはこちらの[リポジトリ](https://github.com/shukawam/helidon-graalvm-demo)に格納してあります。

# Helidon とは ?

Oracle 主導で開発している Java の軽量アプリケーション・フレームワーク(OSS)で、マイクロサービスを実装するための Java ライブラリの集合体となっています。
規模や要件に応じて選択可能な二つのエディションを提供しています。

- Helidon SE: 軽量でフットプリント重視
- Helidon MP: 機能性や互換性を重視

各エディションの有しているコンポーネントの詳細は[公式ドキュメント](https://helidon.io/docs/v2/#/about/01_overview)をご参照ください。

# 調査メモ

Health Check のエンドポイントを持つアプリケーションを作成します。Helidon の場合は、デフォルトで組み込まれていますが、Spring Boot の場合は Spring Boot Actuator を依存関係に追加すると自動的に有効になります。
それぞれリクエストを発行すると以下のようなレスポンスを取得することができます。

**Helidon SE/MP**  
リクエスト；

```
$ curl --request GET \
>   --url http://localhost:8080/health
```

レスポンス；

```json
{
  "outcome": "UP",
  "status": "UP",
  "checks": [
    {
      "name": "deadlock",
      "state": "UP",
      "status": "UP"
    },
    {
      "name": "diskSpace",
      "state": "UP",
      "status": "UP",
      "data": {
        "free": "20.45 GB",
        "freeBytes": 21957705728,
        "percentFree": "45.47%",
        "total": "44.97 GB",
        "totalBytes": 48287940608
      }
    },
    {
      "name": "heapMemory",
      "state": "UP",
      "status": "UP",
      "data": {
        "free": "483.44 MB",
        "freeBytes": 506924304,
        "max": "7.78 GB",
        "maxBytes": 8352956416,
        "percentFree": "99.82%",
        "total": "498.00 MB",
        "totalBytes": 522190848
      }
    }
  ]
}
```

**Spring Boot**

リクエスト；

```
$ curl --request GET \
>   --url http://localhost:8080/actuator/health
```

レスポンス；

```json
{
  "status": "UP"
}
```

今回はこのようなアプリケーションに対して、以下のような環境を作成し比較してみたいと思います。

- Helidon(SE/MP) Uber JAR on OpenJDK 11
- Spring Boot Uber JAR on OpenJDK 11
- Helidon(SE/MP) Native Image

## アーカイブのサイズ比較

### 普通にビルド

Maven を使用して普通にビルドしたアーカイブファイルの容量はそれぞれ以下のよう。

| Helidon SE | Helidon MP | Spring Boot     |
| ---------- | ---------- | --------------- |
| 8,082 byte | 6,653 byte | 18,958,042 byte |

流石に軽量フレームワークだけあって、アーカイブファイルの容量は全然違います。SE より MP の方が微妙に小さいのは少し意外でした。

### Docker Image

こちらはほとんど差がなく意外でした。

| Helidon SE | Helidon MP | Spring Boot |
| ---------- | ---------- | ----------- |
| 210MB      | 220MB      | 224MB       |

## 起動時間比較(OpenJDK 11)

## 起動時間比較(GraalVM EE 20.3.0)

## 起動時間比較(GraalVM Native Image)

# 終わりに

# 参考

- [Helidon 公式ドキュメント](https://helidon.io/docs/v2/#/about/01_overview)
- [一体何モノなの？GraalVM 入門編](https://speakerdeck.com/hhiroshell/graalvm-for-beginners)

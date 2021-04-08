+++
title = "Oracle FunctionsのCold Start対策"
author = "Shuhei, Kawamura"
date = "2021-04-08"
tags = ["OCI", "Functions", "Java"]
categories = ["tech"]
draft = "false"
[[images]]
  src = "img/2021/0408/89783130b95d3802.jpg"
  alt = "89783130b95d3802.jpg"
  stretch = "stretchH"
+++

# 始めに

OCI の FaaS サービスである Functions の Cold Start の対策をメモしておく。将来的に事前暖機の機能等がリリースされた場合、本記事はアンチパターンとなる可能性があることをご了承の上参照してください。(あくまで、2021/04/08 現在取りえる手法として書きます)

# Functions の定期実行

この手のサービスではよくとられてきた手法ですが、Functions に適当なエンドポイントを用意して定期的に実行するというものです。コンテナが落ちるまでの具体的な時間は不明ですが、感覚的に 15 ~ 20 分といった所なので 10 分に一回叩けば問題ないと思います。現在、Functions 自体に定期実行するような機能は存在しないので外部から実行してあげる必要がありますが、具体的に取りえる方法としては以下の通り。(他にもあるかもしれないですが...)

- 適当な Compute Instance から CLI や SDK、署名付きの HTTP エンドポイントを定期的に叩く
- OCI Monitoring の Health Check を使用する

一つずつ簡単に紹介します。

## 適当な Compute Instance から定期実行する

こちらは、特に解説不要かな、、cron でも書いて適当に叩いてください。

## OCI Monitoring の Health Check 機能を使用する

### Health Check のエンドポイントを持つ Functions を作成する

```bash
fn init --runtime java11 fn-hello
```

一つの Functions に対して、API Gateway でエンドポイントを複数作成し、そのパスで分岐処理するパターンです。  
今回は、

- `/cold/hello`: Hello, world という文字列を返却するエンドポイント
- `/cold/health`: 定期実行用の常に`{"status": "UP"}`を返却するエンドポイント

といった形で作ってみました。

```java
package com.example.fn;

import com.fnproject.fn.api.httpgateway.HTTPGatewayContext;

public class HelloFunction {
    private static final String HEALTH_CHECK_URL = "/cold/health";

    public static class HealthCheckResult {
        public HealthStatus status;

        public HealthCheckResult() {
            // default constructor.
        }

        public HealthCheckResult(HealthStatus status) {
            this.status = status;
        }
    }

    public enum HealthStatus {
        UP, DOWN
    }

    public Object handleRequest(HTTPGatewayContext httpGatewayContext) {
        return HEALTH_CHECK_URL.equals(httpGatewayContext.getRequestURL())
                ? healthCheck()
                : hello();
    }

    private String hello() {
        return "Hello, world!!";
    }

    private HealthCheckResult healthCheck() {
        // always return "UP"
        return new HealthCheckResult(HealthStatus.UP);
    }
}
```

作った Functions をデプロイします。

```bash
fn --verbose deploy --app <your-app-name>
```

### API Gateway

こちらは、同一 Functions に対し、以下のようにルーティングを二つ設定します。

- ルーティング 1
  - パス: `/hello`
  - メソッド: GET
  - タイプ: Oracle Functions
  - アプリケーション、機能名: 自分で作成したアプリケーションの名前と関数名を指定する
- ルーティング 2
  - パス: `/health`
  - メソッド: GET
  - タイプ: Oracle Functions
  - アプリケーション、機能名: ルーティング 1 で指定したものと同じアプリケーション、関数名を指定する

### Monitoring の設定

こちらは、OCI Monitoring の持つ Health Check 機能を使用して、API Gateway 経由で公開されている Functions のエンドポイントを定期的に実行するというものです。  
OCI コンソールから **モニタリング** > **ヘルス・チェック**と選択する。

![image01](https://shukawam.github.io/blog/img/2021/0408/image01.png)

**ヘルス・チェックの作成**を押します。

![image02](https://shukawam.github.io/blog/img/2021/0408/image02.png)

以下のように入力し、ヘルス・チェックを作成します。

![image03](https://shukawam.github.io/blog/img/2021/0408/image03.png)

- ヘルス・チェック名: functions-health-check
- ターゲット: API Gateway のホスト名
- バンテージ・ポイント: AWS Central EU1, Azure North Europe, Google West US
- プロトコル: HTTPS
- ポート: 443
- パス: `/cold/health`(API Gateway に設定した Functions の Health Check 用のエンドポイント)
- メソッド: GET
- タイムアウト: 30 秒
- 間隔: 10 分

ヘルス・チェックの履歴を見た際に可用性が使用可能となっていれば OK です。

![image04](https://shukawam.github.io/blog/img/2021/0408/image04.png)

# 終わりに

いかがでしたでしょうか。今回紹介した Functions 内に複数機能を持つような設計は複数機能での Cold Start を避けるという意味では非常に理にかなった設計となります。だからと言って、全ての機能を単一の関数内に閉じ込めると今度は、コンテナイメージが大きくなりすぎてしまい、イメージのプルに時間がかかってしまうなどの問題もあるのでバランスを見ながら設計していく必要があります。

# 参考

- [Using OCI Monitoring Healthchecks to Schedule execution of Serverless Functions on Oracle Cloud Infrastructure](https://technology.amis.nl/oracle-cloud/using-oci-monitoring-healthchecks-to-schedule-execution-of-serverless-functions-on-oracle-cloud-infrastructure/)

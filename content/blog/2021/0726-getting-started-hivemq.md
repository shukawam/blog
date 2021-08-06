+++
title = "HiveMQ 入門"
author = "Shuhei, Kawamura"
date = "2021-07-26"
tags = ["MQTT", "HiveMQ"]
categories = ["tech"]
draft = "false"
+++

# 始めに

業務で MQTT を扱う必要が出てきたので、その素振りがてら MQTT Broker の一つである [HiveMQ](https://www.hivemq.com/) に入門し、ざっくりと以下の環境を作ることをゴールにしてみます。

![architecture](http://localhost:1313/blog/img/2021/0726/architecture.png)

# About HiveMQ

HiveMQ は、IoT デバイスとの間で MQTT ベースでメッセージングを可能にするプラットフォームです。標準的な MQTT の機能は完全にサポートされてあり、HA 構成や既存システムへの統合等様々な拡張機能を提供しています。エディションは商用版と OSS 版の 2 種類が存在しますが、今回は OSS 版を用います。また、HiveMQ の実行方法はいくつか選択肢があります。

- [フルマネージドな HiveMQ Cloud Service を使用する](https://www.hivemq.com/docs/hivemq/4.6/user-guide/getting-started.html#hivemq-cloud)
- [HiveMQ のパッケージをダウンロードして使う](https://www.hivemq.com/docs/hivemq/4.6/user-guide/getting-started.html#download)
- [Docke 上で実行する](https://www.hivemq.com/docs/hivemq/4.6/user-guide/getting-started.html#docker)
- [AWS 上で実行する](https://www.hivemq.com/docs/hivemq/4.6/user-guide/getting-started.html#aws)(HiveMQ がプリインストールされた AMI を使用する)
- [Azure 上で実行する](https://www.hivemq.com/docs/hivemq/4.6/user-guide/getting-started.html#azure)(Azure Resource Manager を使用して、HiveMQ 環境を作成する)

今回は、最も手軽に試せそうだったので Docker 上で動作させる方法を採用したいと思います。

# 作業手順

まずは、HiveMQ を起動します。

```bash
docker run --rm -d -p 8080:8080 -p 1883:1883 hivemq/hivemq4
```

これで準備は完了です。超簡単！  
次に、作成した MQTT Broker に接続する MQTT Client を実装します。今回は、[Helidon](https://helidon.io/#/) で作成したアプリケーションを MQTT Client(Device n) と見立てて進めていきます。つまり、先ほどの図をもう少し詳細に書くと以下のようになります。

![architecture-detail](http://localhost:1313/blog/img/2021/0726/architecture-detail.png)

では、早速作っていきます。

## Device ~ MQTT Broker

まずは、図で言うとこの辺りから。

![image02](http://localhost:1313/blog/img/2021/0726/image02.png)

Helidon アプリケーションのひな形を Helidon CLI を用いて作ります。

```bash
helidon init --flavor MP --architecture bare --groupid me.shukawam --package me.shukawam.mqtt mqtt-client
```

次に、MQTT Client として動作させるために以下の依存関係を追加します。

```xml
<!-- MQTT Client -->
<dependency>
    <groupId>com.hivemq</groupId>
    <artifactId>hivemq-mqtt-client</artifactId>
    <version>1.2.2</version>
</dependency>
<!-- MicroProfile - Scheduling -->
<dependency>
    <groupId>io.helidon.microprofile.scheduling</groupId>
    <artifactId>helidon-microprofile-scheduling</artifactId>
</dependency>
```

TODO

## MQTT Source Connector

次はこの辺りです。

![image03](http://localhost:1313/blog/img/2021/0726/image03.png)

今回は、[Confluent](https://www.confluent.io/) から提供されている Kafka Connect を使用して MQTT のデータストリームを Kafka へ接続したいと思います。
まずは、MQTT 用の Kafka Connector を使うための Confluent Platform をインストールします。まずは、ZIP のダウンロード & 展開をします。

```bash
# Download Confluent Platform
curl -O http://packages.confluent.io/archive/6.2/confluent-6.2.0.zip
# Unzip
unzip confluent-6.2.0.zip
```

後は、展開したディレクトリ内に含まれる `bin` を環境変数の `PATH` に追加すれば良いです。インストールできたかどうかの確認は以下のコマンドで行えます。

```bash
confluent-hub
usage: confluent-hub <command> [ <args> ]

Commands are:
    help      Display help information
    install   install a component from either Confluent Hub or from a local file

See 'confluent-hub help <command>' for more information on a specific command.
```

次に、 [MQTT Connector](https://www.confluent.io/hub/confluentinc/kafka-connect-mqtt) をインストールします。対話形式でインストールが始まるので、全部 `y` を入力しインストールします。

```bash
confluent-hub install confluentinc/kafka-connect-mqtt:1.4.1
```

# 終わりに

# 参考

- [Getting Started with HiveMQ](https://www.hivemq.com/docs/hivemq/4.6/user-guide/getting-started.html#get-started)
- [HiveMQ MQTT Client - MQTT Client Library Encyclopedia](https://www.hivemq.com/blog/mqtt-client-library-enyclopedia-hivemq-mqtt-client/)

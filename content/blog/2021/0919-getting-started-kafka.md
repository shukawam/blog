+++
title = "Oracle Linux にApache Kakfaをインストールする"
author = "Shuhei, Kawamura"
date = "2021-09-19"
tags = ["kafka", "oracle linux"]
categories = ["tech"]
draft = "false"
+++

# 始めに

積んでおいた[Kafka のオライリー本](https://www.oreilly.co.jp/books/9784873118499/)を読み始め、これから本格的に Kafka を勉強しようと思っています。それに際して、手軽に触れる環境があったほうがいいな、ということでさくっと Oracle Linux 上に構築します。

# 手順

まずは、アーカイブをダウンロードします。ダウンロードサイトは[こちら](https://www.apache.org/dyn/closer.cgi?path=/kafka/2.8.0/kafka_2.13-2.8.0.tgz)にあります。

```bash
wget https://ftp.tsukuba.wide.ad.jp/software/apache/kafka/2.8.0/kafka_2.13-2.8.0.tgz
```

展開します。

```bash
tar -xzf kafka_2.13-2.8.0.tgz
```

環境変数を設定しておきます。`$HOME/.bashrc`に以下を追記

```bash
# Kafka settings
export KAFKA_HOME=$HOME/bin/kafka_2.13-2.8.0
```

Zookeeper, Kafka を起動します。

```bash
# Zookeeper
$KAFKA_HOME/bin/zookeeper-server-start.sh -daemon $KAFKA_HOME/config/zookeeper.properties

# Kafka
$KAFKA_HOME/bin/kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties
```

検証用に Topic の作成、メッセージの publish/subscribe を行います。

```bash
# Create "test" topic
$KAFKA_HOME/bin/kafka-topics.sh --create --zookeeper localhost:2181 --topic test --partitions 5 --replication-factor 1
Created topic test.
```

```bash
# Publish message
$KAFKA_HOME/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test
>The first message.
>The second message.
```

```bash
# Consume message
$KAFKA_HOME/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test
The first message.
The second message.
```

以上。

# 参考

- [APACHE KAFKA QUICKSTART](https://kafka.apache.org/quickstart)

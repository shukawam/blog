+++
title = "HiveMQ を Kubernetes 上に構築する"
author = "Shuhei, Kawamura"
date = "2021-08-06"
tags = ["kubernetes", "mqtt", "hivemq"]
categories = ["tech"]
draft = "false"
+++

# 始めに

業務で MQTT を扱う必要性が出てきたので、自分で色々遊べる MQTT Broker を Kubernetes 上に構築します。

# About HiveMQ

HiveMQ は、IoT デバイスとの間で MQTT ベースでメッセージングを可能にするプラットフォームです。標準的な MQTT の機能は完全にサポートされてあり、HA 構成や既存システムへの統合等様々な拡張機能を提供しています。エディションは商用版と OSS 版の 2 種類が存在しますが、今回は OSS 版を用います。また、HiveMQ の実行方法はいくつか選択肢があります。

- [フルマネージドな HiveMQ Cloud Service を使用する](https://www.hivemq.com/docs/hivemq/4.6/user-guide/getting-started.html#hivemq-cloud)
- [HiveMQ のパッケージをダウンロードして使う](https://www.hivemq.com/docs/hivemq/4.6/user-guide/getting-started.html#download)
- [Docke 上で実行する](https://www.hivemq.com/docs/hivemq/4.6/user-guide/getting-started.html#docker)
- [AWS 上で実行する](https://www.hivemq.com/docs/hivemq/4.6/user-guide/getting-started.html#aws)(HiveMQ がプリインストールされた AMI を使用する)
- [Azure 上で実行する](https://www.hivemq.com/docs/hivemq/4.6/user-guide/getting-started.html#azure)(Azure Resource Manager を使用して、HiveMQ 環境を作成する)

今回は、可用性なども考慮して Kubernetes(Oracle Container Engine for Kubernetes)上に HiveMQ のクラスターを構築します。

# 構築手順

非常に簡単です。まずは、HiveMQ Cluster 用を以下のように書きます。

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: hivemq-replica
spec:
  replicas: 3
  selector:
    app: hivemq-cluster1
  template:
    metadata:
      name: hivemq-cluster1
      labels:
        app: hivemq-cluster1
    spec:
      containers:
        - name: hivemq-pods
          image: hivemq/hivemq3:dns-latest
          ports:
            - containerPort: 8080
              protocol: TCP
              name: web-ui
            - containerPort: 1883
              protocol: TCP
              name: mqtt
          env:
            - name: HIVEMQ_DNS_DISCOVERY_ADDRESS
              value: "hivemq-discovery.default.svc.cluster.local."
            - name: HIVEMQ_DNS_DISCOVERY_TIMEOUT
              value: "20"
            - name: HIVEMQ_DNS_DISCOVERY_INTERVAL
              value: "21"
          readinessProbe:
            tcpSocket:
              port: 1883
            initialDelaySeconds: 30
            periodSeconds: 60
            failureThreshold: 60
          livenessProbe:
            tcpSocket:
              port: 1883
            initialDelaySeconds: 30
            periodSeconds: 60
            failureThreshold: 60
---
kind: Service
apiVersion: v1
metadata:
  name: hivemq-discovery
  annotations:
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
spec:
  selector:
    app: hivemq-cluster1
  ports:
    - protocol: TCP
      port: 1883
      targetPort: 1883
  clusterIP: None
```

次に MQTT Broker を外部に公開する場合は以下の Manifest を書きます。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: hivemq-mqtt
  annotations:
    service.spec.externalTrafficPolicy: Local
    # if you use other managed kubernetes, see specific document
    service.beta.kubernetes.io/oci-load-balancer-shape: "flexible"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-min: "10"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-max: "50"
spec:
  selector:
    app: hivemq-cluster
  ports:
    - name: mqtt
      protocol: TCP
      port: 1883
      targetPort: 1883
  type: LoadBalancer
```

また、GUI を使いたい場合は、以下の Manifest も作成してください。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: hivemq-web-ui
  annotations:
    service.beta.kubernetes.io/oci-load-balancer-shape: "flexible"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-min: "10"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-max: "10"
spec:
  selector:
    app: hivemq-cluster1
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: LoadBalancer
```

後は、apply するだけです。

```bash
kubectl apply -f .
```

作成されたリソースを確認してみます。

```bash
kubectl get pods,services
NAME                       READY   STATUS    RESTARTS   AGE
pod/hivemq-replica-c85vk   1/1     Running   0          6h43m
pod/hivemq-replica-js4h7   1/1     Running   0          6h43m
pod/hivemq-replica-pfv5p   1/1     Running   0          6h43m
pod/hivemq-replica-wh5p8   1/1     Running   0          6h43m

NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
service/hivemq-discovery   ClusterIP      None            <none>           1883/TCP         6h43m
service/hivemq-mqtt        LoadBalancer   10.96.217.166   140.83.39.93     1883:31530/TCP   6h43m
service/hivemq-web-ui      LoadBalancer   10.96.142.154   158.101.91.198   8080:32424/TCP   8m56s
```

簡易的にテストしてみます。今回は、VSCode の拡張機能に MQTT Client ([VSMqtt](https://marketplace.visualstudio.com/items?itemName=rpdswtk.vsmqtt))があったのでそちらを使います。

以下のように接続の定義をします。(Host 部は自分の環境に合わせて修正してください)

```json
{
  "vsmqtt.brokerProfiles": [
    {
      "name": "shukawam",
      "host": "140.83.39.93",
      "port": 1883,
      "clientId": "vsmqtt_client"
    }
  ]
}
```

実際に、pub/subしてみると以下のようになります。

![image01](https://shukawam.github.io/blog/img/2021/0806/image01.png)

# 終わりに

[Kubernetes Operator](https://www.hivemq.com/blog/hivemq-kubernetes-operator-eap/) もあるので、そちらから構築してもいいですね。

# 参考

- [How to run a HiveMQ cluster with Docker and Kubernetes](https://www.hivemq.com/blog/hivemq-cluster-docker-kubernetes/)
- [HiveMQ Kubernetes Operator Early Access Preview: Simply Automate Your HiveMQ Deployments Today](https://www.hivemq.com/blog/hivemq-kubernetes-operator-eap/)

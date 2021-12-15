+++
title = "Chaos Mesh を OKE で試す"
author = "Shuhei, Kawamura"
date = "2021-12-15"
tags = ["chaos mesh", "kubernetes", "chaos engineering"]
categories = ["tech"]
draft = "false"
+++

# 始めに

Kubernetes 上に構築したシステムにカオスエンジニアリングを導入する際の（おそらく）最有力候補である [Chaos Mesh](https://chaos-mesh.org/) を色々触って試してみます。とりあえず、今回は構築まで。

# 手順

インストール用のスクリプトが提供されているので、それを使います。

```bash
curl -sSL https://mirrors.chaos-mesh.org/v2.1.1/install.sh | bash
```

しばらくすると、`chaos-testing` という namespace にいくつかリソースが作成されます。

```bash
kubectl get pods,service -n chaos-testing
```

こんな感じです。

```bash
NAME                                           READY   STATUS    RESTARTS   AGE
pod/chaos-controller-manager-87f7677bf-jw27d   1/1     Running   0          11m
pod/chaos-controller-manager-87f7677bf-wc4zx   1/1     Running   0          11m
pod/chaos-controller-manager-87f7677bf-z4xpm   1/1     Running   0          11m
pod/chaos-daemon-8nr8f                         1/1     Running   0          11m
pod/chaos-daemon-jrl4q                         1/1     Running   0          11m
pod/chaos-dashboard-85db5f48d8-vzxtf           1/1     Running   0          11m

NAME                                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                 AGE
service/chaos-daemon                    ClusterIP   None            <none>        31767/TCP,31766/TCP                     11m
service/chaos-dashboard                 NodePort    10.96.199.100   <none>        2333:32490/TCP                          11m
service/chaos-mesh-controller-manager   ClusterIP   10.96.208.200   <none>        443/TCP,10081/TCP,10082/TCP,10080/TCP   11m
```

それぞれ、簡単に補足すると以下のようになります。

- Chaos Dashboard: カオスを仕込んだり、その結果を観察するための Web インタフェースを提供
- Chaos Controller Manager: カオスを仕込むためのスケジューリングや管理等を行うコンポーネント
- Chaos Daemon: 実際に実行するコンポーネント

実際にカオスを仕込む際は、YAML ファイル(Manifest)を記述するか、ダッシュボードから操作するらしいのですが、初心者なので画面から触ってみます。ということで、ダッシュボードは Load Balancer 経由で公開します。

```bash
kubectl patch service chaos-dashboard \
> --patch '{"spec": {"type": "LoadBalancer"}, "metadata": {"annotations": {"service.beta.kubernetes.io/oci-load-balancer-shape": "flexible", "service.beta.kubernetes.io/oci-load-balancer-shape-flex-min": "10", "service.beta.kubernetes.io/oci-load-balancer-shape-flex-max": "20"}}}' \
> --namespace chaos-testing
```

※Load Balancer の仕様に関しては、各クラウドプロバイダーの仕様を参照してください。本例は、OKE[^1] でのみ実行可能です。

[^1]: Oracle Container Engine for Kubernetes

実際に、パブリック IP が割り当てられたことを確認します。

```bash
kubectl get service -n chaos-testing
```

以下のようにパブリック IP が割り当てられていれば OK です！

```bash
NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                 AGE
chaos-daemon                    ClusterIP      None            <none>           31767/TCP,31766/TCP                     25m
chaos-dashboard                 LoadBalancer   10.96.199.100   150.230.100.86   2333:32490/TCP                          25m
chaos-mesh-controller-manager   ClusterIP      10.96.208.200   <none>           443/TCP,10081/TCP,10082/TCP,10080/TCP   25m
```

実際に画面にアクセスしてみましょう。ブラウザで、`http://<public-ip>:2333`でアクセスできます。

![image01](https://shukawam.github.io/blog/img/2021/1215/image01.png)

# 終わりに

思っていたよりも簡単に導入できました。（本当に大変なのは、ここからどういうシナリオを考えるか...という事だと思いますが）
次回以降で、サンプルアプリケーションに対していくつかシナリオを試していきたいと思います。

# 参考

- [Chaos Mesh](https://chaos-mesh.org/docs/)

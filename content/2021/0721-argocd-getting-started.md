+++
title = "Argo CD - Getting Started でちょっとハマったこと"
author = "Shuhei, Kawamura"
date = "2021-07-21"
tags = ["argocd"]
categories = ["tech"]
draft = "true"
+++

# 始めに

タイトルの通りです。[Argo CD の Getting Started](https://argoproj.github.io/argo-cd/getting_started/) を実施した際のハマったことを書き起こしておきます。

# 何が起きたのか？

新しく作った Kubernetes クラスタに Argo CD(v2.0.4) を導入しようと思ったところ、ドキュメントの Getting Started 通りには行かなかった。Argo CD の初期パスワードは、 v1.9 以降 Kubernetes の `secret` に格納されているので確認するためには、kubectl で確認すればよい。

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

が、確認したところ、`argocd-initial-admin-secret` なんて存在しないとエラーメッセージが出力された。(原因までは調査してないです。)

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
Error from server (NotFound): secrets "argocd-initial-admin-secret" not found
```

関連 Issue あったのでリンク張っておきます。(2021/07/25 追記);  
[https://github.com/argoproj/argo-cd/issues/6787](https://github.com/argoproj/argo-cd/issues/6787)

# 解決策

修正されるまでは、admin ユーザーのパスワードが Kubernetes の`secret`で管理されているのでそれを単純に変更すればよい。

```bash
kubectl -n argocd patch secret argocd-secret \
> -p '{"stringData": {"admin.password": "$2a$10$qo3Q/Zt7UaYZgcapXCZ8jOdcnU1Z/PXe8Y4EG.JHiLZUCoBVQ9sFq", "admin.passwordMtime": "'$(date +%FT%T%Z)'"}}'
secret/argocd-secret patched
```

なお、パスワードは bcrypt でハッシュ化する必要があるので、[この辺り](https://www.browserling.com/tools/bcrypt)のサイトを活用すると便利です。

# 参考

- [Argo CD FAQ - I forgot the admin password, how do I reset it?](https://argoproj.github.io/argo-cd/faq/#i-forgot-the-admin-password-how-do-i-reset-it)

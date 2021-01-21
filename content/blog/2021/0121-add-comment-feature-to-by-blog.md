+++
title = "Staticmanを使用してブログにコメント機能を追加してみた"
description = ""
author = "Shuhei, Kawamura"
date = "2021-01-21"
tags = ["hugo", "staticman"]
categories = ["tech"]
draft = "true"
[[images]]
  src = "img/2020/0909/hugo.png"
  alt = "hugo.png"
  stretch = "stretchH"
+++

# 始めに

最近、引っ越しやら転職やらでバタバタしていたので久方ぶりの更新です。今回は、Hugo で作成しているこのブログにコメント機能をつけてみたいと思います。

# Staticman

[Staticman](https://staticman.net/) は GitHub 上にコメントファイルをホスティングすることで、静的サイトのコメント機能を実現しています。GitHub のコラボレータに[@staticmanlab](https://github.com/staticmanlab)を追加すると、コメント投稿時の POST リクエストを Staticman の API を通して、リポジトリにコメントファイルを push してくれます。

# 手順

## GitHub のコラボレータに[@staticmanlab](https://github.com/staticmanlab)を追加する

GitHub Pages を設定しているリポジトリ（私の場合だと、[shukawam/blog](https://github.com/shukawam/blog/)）のコラボレータに[@staticmanlab](https://github.com/staticmanlab)を追加します。

Settings > Manage Account > Invite a collaborator に`staticmanlab`と入力しコラボレータを追加します。

![img01](https://shukawam.github.io/blog/img/2021/0121/image01.png)

以下のリクエストを発行し、レスポンスが返ってくればコラボレータの追加作業は完了です。（コラボレータ承認用の API をコールしています。）

```
$ curl https://staticman3.herokuapp.com/v3/connect/github/<username>/<repo-name>
OK!
```

## config.toml を修正する

```toml
[params.staticman]
  enabled             = true
  api                 = "https://staticman3.herokuapp.com"  # No Trailing Slash
  gitProvider         = "github"
  username            = "<username>"
  repo                = "<repo-name>"
  branch              = "master"
```

# 終わりに

非常に簡単にコメント機能を追加することができました。

# 参考

- [hugo-future-imperfect-slim#adding-staticman](hhttps://github.com/pacollins/hugo-future-imperfect-slim/wiki/staticman.yml#adding-staticman)

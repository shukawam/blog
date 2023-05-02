+++
title = "Hugo + GitHub Pagesでブログを始めた"
author = "Shuhei, Kawamura"
date = "2020-09-08"
tags = ["hugo", "github", "github pages", "github actions"]
categories = ["tech"]
draft = "true"
+++

# 始めに

つい先日、Qiita からはてなブログに移行したばかりだったのですが、個人的に何点か気に食わない点があったので自分で作って運用してみることにしました。
この記事は実際にブログを作ってみた時のお話です。

# このブログについて

## 環境

- 静的サイトジェネレーター：[Hugo Static Site Generator](https://gohugo.io/) v0.74.3
- 運用（ホスティング）：GitHub Pages

という環境で、作成し運用されています。

## 選定理由

いくつか背景があるので、順にお話します。

### Qiita, はてなブログじゃダメだった理由

Markdown で記載できるブログあるあるだと思うのですが、**とにかく画像のアップロードがめんどくさい**。他にも、デザインが気に食わなかったりなど細かい理由はありますが、理由の 9 割はこれでした。自分のメモ、ついでに外部発信くらいと考えてる人にとってはなるべく省エネで執筆したかったのです。

### 静的サイトジェネレーター

このブログは、Golang 謹製の[Hugo](https://gohugo.io/)を使用して作成しています。Hugo 以外の有名どころだと、Gatsby, Hexo, Jekyll etc ... 辺りがありますが、正直何でも良かったです。環境構築が非常に楽という噂を聞きつけたので、Hugo を採用しました。（他のものは試してすらないです）実際、私の PC は Windows 10 なのですが、環境構築～このブログを作成するまで 1 時間程度でできました。

### ホスティング先

ホスティング先には、いくつか条件がありました。

- 無料で運用できること
- インフラ（サーバ）の面倒を見なくてもいいこと

これらを加味した結果、ホスティング先は GitHub Pages or GitLab Pages に絞られました。最初は、Hugo によって生成されるコンテンツを管理したくなかったので、GitLab Pages にしようと思っていましたが、個人的に GitLab から GitHub に移行したばかりだったので少し微妙だなと思ったことをツイートしたのですが、こんな回答をいただけました。

{{< tweet 1302572704789270529 >}}

どうやら、Hugo で生成されるファイル群をバージョン管理する方法ではなく、GitHub Actinos で静的ファイルを任意のブランチに生成し、そのブランチを GitHub Pages へデプロイするという方法があるようです。

# ブログの作り方

前置きが長くなってしまいましたが、当ブログの作り方を簡単に説明したいと思います。

- 省エネ、基本的に無料でブログを書きたい人
- 自分でデザインしないと満足できない人

には、お勧めできる方法だと思います。尚、テーマのカスタマイズや自作方法については当記事では触れないので悪しからず。

## 環境構築

Windows 10 を使用しています。Chocolatey, Scoop といったパッケージマネージャーを使用する方法と zip を展開してパスを通す方法がありますが、今回紹介するのは後者の方法です。

[https://github.com/gohugoio/hugo/releases](https://github.com/gohugoio/hugo/releases)から最新版の zip ファイルをダウンロードし適用なディレクトリに解凍します。

同じバージョンでも通常版（hugo）と Extended 版（hugo_extended）と 2 種類ありますが、後ほど紹介する公開されているテーマを使用する場合は**Extended 版**の方を選択してください。（Extended 版の方は、SASS/SCSS が使用できます。公開されているテーマで SASS/SCSS が使用されている場合、通常の hugo ではエラーが発生し静的ファイルの生成ができません。）

次に、パスを通します。システム環境変数から`Path`を選択し、先ほど解凍したディレクトリを指定します。

```bash
hugo version
Hugo Static Site Generator v0.74.3/extended windows/amd64 BuildDate: unknown
```

この様に表示されれば、環境構築は完了です。

## ブログを生成する

適当なディレクトリで以下のように入力する。

```bash
hugo new site blog
Congratulations! Your new Hugo site is created in C:\Git\test\blog.

Just a few more steps and you're ready to go:

1. Download a theme into the same-named folder.
   Choose a theme from https://themes.gohugo.io/ or
   create your own with the "hugo new theme <THEMENAME>" command.
2. Perhaps you want to add some content. You can add single files
   with "hugo new <SECTIONNAME>\<FILENAME>.<FORMAT>".
3. Start the built-in live server via "hugo server".

Visit https://gohugo.io/ for quickstart guide and full documentation.
```

### テーマを選定する

[https://themes.gohugo.io/](https://themes.gohugo.io/)にいい感じのテーマが沢山あるので好きなものを選びます。（ここでは仮に、[Hugo Future Imperfect Slim](https://themes.gohugo.io/hugo-future-imperfect-slim/)を選択したとする）Download ボタンを押すと、[GitHub のリポジトリトップ](https://github.com/pacollins/hugo-future-imperfect-slim)に遷移するので、README を読みながら設定を進めます。

### テーマのインストール

テーマは Git の Submodule として管理するのが主流みたいです。

```bash
cd blog/themes
git submodule add https://github.com/pacollins/hugo-future-imperfect-slim.git
git submodule update --remote --merge
```

### 設定ファイルをコピーし修正する

サンプルサイトの設定ファイル（`config.toml, staticman.yml`）を自サイトのルートにコピーします。

```bash
cp ./hugo-future-imperfect-slim/exampleSite/config.toml ../
cp ./hugo-future-imperfect-slim/exampleSite/staticman.yml ../
```

この手の設定は、自分で動かしながら変えていくのが手っ取り早いかなと思いました。（分からない所だけドキュメントを参照する）

## GitHub Pages で公開する

`.github/workflows/gh-pages.yml`を作成する。詳細な説明は、[リポジトリ](https://github.com/peaceiris/actions-hugo)を参照してください。

```yaml
name: github pages

on:
  push:
    branches:
      - master # Set a branch to deploy

jobs:
  deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0 # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: "0.74.2"
          extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./docs
```

後は、master に push すれば`https://<github username>.github.io/<repository name>`として公開されます！

# 終わりに

細かい個所などは紹介できていないので、詳細が気になる方は[リポジトリ](https://github.com/shukawam/blog)を参照してください。

# 参考

- [https://gohugo.io/](https://gohugo.io/)
- [https://github.com/peaceiris/actions-hugo](https://github.com/peaceiris/actions-hugo)
- [https://github.com/shukawam/blog](https://github.com/shukawam/blog)

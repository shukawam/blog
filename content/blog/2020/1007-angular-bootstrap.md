+++
title = "Angular9系以上でBootstrap4を使う"
description = ""
author = "Shuhei, Kawamura"
date = "2020-10-07"
tags = ["Angular", "Bootstrap"]
categories = ["tech"]
draft = "false"
[[images]]
  src = "img/2020/1007/main.png"
  alt = "main"
  stretch = "stretchH"
+++

# 始めに

※こちらの記事は以前 Qiita で執筆した[Angular9 で Bootstrap4 を使う](https://qiita.com/kawash/items/1134147d5ac61789987d)の移行記事です。執筆当時とバージョンなどを微修正しています。

- Angular 9 系に Bootstrap4 (ng-bootstrap) を適用する手順です。
- 通常の Bootstrap（jQuery, popper.js 依存）を使用してもよいですが、余計なライブラリに依存することになる事になるため、おすすめしません。（Angular の思想にも反します。）
  - ng-bootstrap は、Bootstrap が依存している jQuery, popper.js の実装を Angular の component に差し替えています。
- メジャーバージョンはしっかりと確認する。
  - 特に Angular5 と Angular6+では CLI の設定ファイル周りが大きく変更となっています。
- 公式の英語ドキュメント読むのめんどいって方向け。

# 環境

タイトルにもある通り、今回は Angular10 系に Bootstrap4 系を適用します。

```
$ ng --version

     _                      _                 ____ _     ___
    / \   _ __   __ _ _   _| | __ _ _ __     / ___| |   |_ _|
   / △ \ | '_ \ / _` | | | | |/ _` | '__|   | |   | |    | |
  / ___ \| | | | (_| | |_| | | (_| | |      | |___| |___ | |
 /_/   \_\_| |_|\__, |\__,_|_|\__,_|_|       \____|_____|___|
                |___/


Angular CLI: 10.1.2
Node: 12.18.4
OS: win32 x64

Angular:
...
Ivy Workspace:

Package                      Version
------------------------------------------------------
@angular-devkit/architect    0.1001.2 (cli-only)
@angular-devkit/core         10.1.2 (cli-only)
@angular-devkit/schematics   10.1.2 (cli-only)
@schematics/angular          10.1.2 (cli-only)
@schematics/update           0.1001.2 (cli-only)
```

# 手順

`ng-bootstrap`をインストールする。[公式ドキュメント](https://ng-bootstrap.github.io/#/getting-started)にも記載がありますが、Angular CLI を使用してセットアップすることが強く推奨されています。（依存している Bootstrap のインストール、`angular.json`への設定反映、`app.module.ts`へのインポートをコマンド一発で実行できます。）

```
$ ng add @ng-bootstrap/ng-bootstrap
```

また、Angular 9.0.0 以上かつ ng-bootstrap 6.0.0 の場合は、`@angular/localize`を polyfills に追加する必要があります。

```
$ ng add @angular/localize
```

# 動作確認

`app.component.html`

```html
<button type="button" class="btn btn-primary">BUTTON</button>
```

アプリケーションを起動後、[http://localhost:4200](http://localhost:4200)にアクセスし、以下の画面が見えれば OK です。

![img01](https://shukawam.github.io/blog/img/2020/1007/img01.png)

# 参考

- [ng-bootstrap#getting-started](https://ng-bootstrap.github.io/#/getting-started)
- [setting-up-localization-with-the-angular-cli](https://angular.io/guide/i18n#setting-up-localization-with-the-angular-cli)

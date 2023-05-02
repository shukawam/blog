+++
title = "FIDO2 Tips & Considering"
author = "Shuhei, Kawamura"
date = "2021-04-06"
tags = ["webauthn", "fido2", "ctap"]
categories = ["tech"]
draft = "true"
+++

# 始めに

FIDO2 をアプリケーションに組み込むときのちょっとした Tips や考慮しないといけないことを思いつくままにまとめてみようと思います。思いつく＆何か新しい発見があるたびに記事は随時更新していきます。

# Tips

## [Virtual Authenticators Tab](https://github.com/google/virtual-authenticators-tab)

まずは、Chrome の拡張機能の紹介です。組織のポリシーなどで会社支給のパソコンから`internal`の認証器が呼び出せないパターンもあるでしょう。例えば、デモやちょっとした検証をやりたいのにわざわざそのために USB セキュリティキーを買うのもちょっと馬鹿らしいですよね。そんな時に非常に役に立ちます。  
使い方自体は非常に簡単で、まずは [chrome ウェブストア - Virtual Authenticators Tab](https://chrome.google.com/webstore/detail/virtual-authenticators-ta/gafbpmlmeiikmhkhiapjlfjgdioafmja?hl=JA)で当該機能を有効化する。

![image01](https://shukawam.github.io/blog/img/2021/0406-webauthn-tips/image01.png)

**Virtual Authenticators Tab を追加しますか？** というポップアップが出てくるので、**拡張機能を追加** を押して追加する。

![image02](https://shukawam.github.io/blog/img/2021/0406-webauthn-tips/image02.png)

確認には、おなじみの[webauthn.io](https://webauthn.io/)が良いと思います。  
まずは、DevTool を開き追加されている**Virtual Authenticators**内にある**Enable Virtual Authenticator Environment**のチェックボックスにチェックを入れます。

![image03](https://shukawam.github.io/blog/img/2021/0406-webauthn-tips/image03.png)

後は、お好きな仮想認証器を生成してください。一応、設定できるパラメータを解説すると

- Protocol
  - ctap2
    - CTAP1 を FIDO2 用に拡張したもの
  - u2f
    - CTAP1 のことでクライアントと外部認証器との通信プロトコルのこと
    - 選択肢では、`internal`が選択できそうですが、作成の際にエラーメッセージが出力されます
- Transport
  - usb
    - USB セキュリティキーのこと(e.g. Yubikey)
  - nfc
    - Near Field Communication(e.g. Suica, Pasmo)
  - ble
    - Bluetooth Smart/Bluetooth Low Energy Technology
  - internal
    - プラットフォームの認証器のこと(e.g. Windows Hello, Apple Touch ID, iPhone Face ID)
- Supports Resident Keys
  - 認証器内の保存領域にサービスの情報＆ユーザーの情報の保存をサポートするかどうか
  - CTAP2 から追加された仕様でユーザビリティ向上のための仕様
- Supports User Verification
  - 認証器の持つ本人確認機能を有効にするかどうか

## FIDO2 のデモサイト

※2021/05/28 追記

Auth0 が作成しているデモサイトが**かなり**良かったのでその紹介です。([https://webauthn.me/](https://webauthn.me/))  
FIDO2 の一連のフローが視覚的に分かりやすく整理されています。下手なデモを作成するよりはフローを紹介した後、このサイトに誘導する方が理解度は高まるかもしれない。

![https://shukawam.github.io/blog/img/2021/0406-webauthn-tips/image04.png](https://shukawam.github.io/blog/img/2021/0406-webauthn-tips/image04.png)

# Considering

## ユーザーの負担

Web 標準の認証規格となった WebAuthn だが、どれだけ認証時の利便性が高くなろうとも結局ユーザーが認証器(で生成される公開鍵)をサービス(Relying Party)に登録しなければならない事には変わりはない。現在自分が管理しているパスワードの数がどれだけになるかは分からない(おそらく 100 以上)が、その全てのワークフローにおいてパスワードから認証器に置き換えていく運用が行われていくことはあまり想像できない。従って、一般的には OIDC(OpenId Connect)等と組み合わせた ID 連携の仕組みを補完的に使用することが重要である。  
まとめると、以下のようになる。(全て個人の意見なので参考にする際はよく検討してください)

- 別のシステムとの連携などは一切考える必要がなくそれ一つで完結するシステム
  - Relying Party の実装方法
    - OSS のライブラリを使用して実装する
    - FIDO2 の Relying Party を実装している Identity Provider を使用する(おすすめはこっち)
- 別のシステムと連携が必要なシステム
  - Relying Party の実装方法
    - FIDO2 の Relying Party を実装している Identity Provider を使用する

ID 連携の必要の有無で選択肢が微妙に変わります。必要ないのであれば、極端なことを言えば好きに作ればよいと思いますし、連携する可能性があるのであれば、素直に IdP を使った方がよいと思います。ただし、IdP を採用する場合は、Relying Party の実装が良くも悪くも完全に IdP に依存してしまうというのは少し考慮が必要な点かもしれません。

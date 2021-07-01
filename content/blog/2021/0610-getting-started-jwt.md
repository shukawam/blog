+++
title = "JWT ことはじめ"
author = "Shuhei, Kawamura"
date = "2021-06-10"
tags = ["jwt"]
categories = ["tech"]
draft = "true"
[[images]]
  src = ""
  alt = ""
  stretch = "stretchH"
+++

# 始めに

OAuth のアクセストークン, OIDC の ID トークンなどで使われる JSON Web Token(JWT) ですが、多くの人は

> ヘッダー、ペイロード、署名の 3 部構成(.で連結)になっている。ヘッダーとペイロードは Base64Url エンコードされてて署名はヘッダーで指定された署名のアルゴリズムで決められた作り方をしている。

くらいの認識しか持っていないのではないでしょうか？私もちょっと前までそんな認識でしたが、ちゃんと周辺仕様まで調べてみると認識がかなり甘かったと分かったので自戒のために書き記します。

# JWT とは？

まずは、[RFC7519 - Abstract](https://datatracker.ietf.org/doc/html/rfc7519) から見てみます。

> JSON Web Token (JWT) は 2 者間でやりとりされるコンパクトで URL-safe なクレームの表現方法である. JWT に含まれるクレームは JavaScript Object Notation (JSON) オブジェクトとしてエンコードされ, JSON Web Signature (JWS) のペイロードや JSON Web Encryption (JWE) の平文として利用される. JWS や JWE とともに用いることで, クレームに対してデジタル署名や MAC を付与と暗号化の両方を行うことが可能となる.

とのことです... ここを見るだけでも関連仕様がいくつか出てきますが JWT の関連仕様としては以下が上げられます。

- JWS: JSON をベースとしたデータ構造を用いてデジタル署名や MACs によって保護されたコンテンツを表す
- JWE: JSON ベースのデータ構造を用いて暗号化されたコンテンツを表す
- JWK: 暗号鍵を表現するための JSON データ構造
- JWA: JWS, JWE で使用される暗号アルゴリズム

# 終わりに

# 参考

- [JSON Web Token (JWT) - OpenID Foundation Japan - 翻訳](https://openid-foundation-japan.github.io/draft-ietf-oauth-json-web-token-11.ja.html)

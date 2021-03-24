+++
title = "WebAuthn DeepDive #1 - navigator.credentials.create()"
author = "Shuhei, Kawamura"
date = "2021-03-24"
tags = ["WenAuthn", "CTAP"]
categories = ["tech"]
draft = "false"
[[images]]
  src = "img/2021/0323/webauthn1-960x480.png"
  alt = "webauthn"
  stretch = "stretchH"
+++

# 始めに

[WebAuthn](https://webauthn.io/) について学習したときのメモ。今回は、[WebAuthn](https://webauthn.io/)よりも CTAP の話が多めかも。

# Deep Dive

この辺の仕様を読んでいく。

![regist](https://shukawam.github.io/blog/img/2021/0323/WebAuthn_Registration_r4.png)

※図は[https://developer.mozilla.org/ja/docs/Web/API/Web_Authentication_API](https://developer.mozilla.org/ja/docs/Web/API/Web_Authentication_API)から参照

全体の流れとしては、

2. `navigator.credentials.create()`が実行される
   - `authenticatorMakeCredential()`を実行するためのパラメータを組み立てる
3. 認証器の`authenticatorMakeCredential()`が呼び出される
4. `authenticatorMakeCredential()`を実行した結果がブラウザに返される

となっています。WebAuthn が実行され、認証器の`authenticatorMakeCredential()`が呼び出される所から順番に見ていきます。

## 2. `navigator.credentials.create()`の実行

`navigator.credentials.create()`が呼び出されると、認証器の`authenticatorMakeCredential()`を実行するためのパラメータを組み立てて実行します。その際、`authenticatorMakeCredential()`実行のために必要なパラメータの仕様は、[こちら](https://fidoalliance.org/specs/fido-v2.0-id-20180227/fido-client-to-authenticator-protocol-v2.0-id-20180227.html#authenticatorMakeCredential)に公開されています。一部抜粋して読んでいきたいと思います。(ひとまず今回は、**Required** のものだけ確認していきます)

| パラメータ名     | 型                            | 概要                                                                                        |
| ---------------- | ----------------------------- | ------------------------------------------------------------------------------------------- |
| clientDataHash   | Byte Array                    | clientData のハッシュ                                                                       |
| rp               | PublicKeyCredentialRpEntity   | Relying Party の情報                                                                        |
| user             | PublicKeyCredentialUserEntity | 新しく生成する Credential に紐づくユーザーの情報                                            |
| pubKeyCredParams | CBOR Array                    | 生成する Credential のパラメータ(type: `public-key`で固定, alg: 公開鍵の暗号化アルゴリズム) |

### clientDataHash

> Hash of the ClientData contextual binding specified by host. See [WebAuthN].

clientData のハッシュ...これだけ言われても、正直意味が分からないですが、[WebAuthN](https://fidoalliance.org/specs/fido-v2.0-id-20180227/fido-client-to-authenticator-protocol-v2.0-id-20180227.html#biblio-webauthn)を見ろとの事なので見てみます。

> This attribute, inherited from AuthenticatorResponse, contains the JSON-compatible serialization of client data (see § 6.5 Attestation) passed to the authenticator by the client in order to generate this credential. The exact JSON serialization MUST be preserved, as the hash of the serialized client data has been computed over it.

Credential(パスワードや生体情報などユーザ認証に必要な情報)を生成するのに必要なクライアントデータが JSON 互換のシリアライズされたものだそう。[JSON-compatible serialization of client data](https://www.w3.org/TR/webauthn/#collectedclientdata-json-compatible-serialization-of-client-data)をもう少し見てみましょう。

> **JSON-compatible serialization of client data**
> This is the result of performing the JSON-compatible serialization algorithm on the CollectedClientData dictionary.

[CollectedClientData](https://www.w3.org/TR/webauthn/#dictdef-collectedclientdata)がシリアライズされたものだという事が分かりました。[CollectedClientData](https://www.w3.org/TR/webauthn/#dictdef-collectedclientdata) は以下のようになっています。

```txt
dictionary CollectedClientData {
    required DOMString           type;
    required DOMString           challenge;
    required DOMString           origin;
    boolean                      crossOrigin;
    TokenBinding                 tokenBinding;
};

dictionary TokenBinding {
    required DOMString status;
    DOMString id;
};

enum TokenBindingStatus { "present", "supported" };
```

ここでも Required のプロパティのみ見ていきます。

| パラメータ名 | 型        | 概要                                                                                                                 |
| ------------ | --------- | -------------------------------------------------------------------------------------------------------------------- |
| type         | DOMString | Credential を生成するときは`webauthn.create`、取得するときには`webauthn.get`が固定値で設定される                     |
| challenge    | DOMString | Relying Party で生成された challenge(16 バイト以上のランダムバッファー)が base64url エンコードされたものが設定される |
| origin       | DOMString | クライアントの origin(e.g. <http://localhost:4200>)が設定される                                                      |

ということで、`authenticatorMakeCredential()`が実行されるときには以下のようなデータがシリアライズされて渡されるみたいです。

```txt
{
  "type": "webauthn.create",
  "challenge": "xTzphZPuJyfW12TAT…",
  "origin": "https://example.com"
}
```

ちなみに、シリアライズするときの手順はこちら([5.8.1.1. Serialization](https://www.w3.org/TR/webauthn/#clientdatajson-serialization))に記されております。

### rp

> This PublicKeyCredentialRpEntity data structure describes a Relying Party with which the new public key credential will be associated. It contains the Relying party identifier, (optionally) a human-friendly RP name, and (optionally) a URL referencing a RP icon image. The RP name is to be used by the authenticator when displaying the credential to the user for selection and usage authorization.

とのことで、Relying Party の情報が含まれているデータのようです。最終的には、以下のようなデータが渡されます。

```txt
{
  // Required
  "id": "example.com", // Relying Partyを識別するID
  // Optionally
  "name": "Relying Party Example", // Relying Partyの名前
  "icon": "https://example.com/..." // アイコンイメージが格納されている場所をURLで指定する
}
```

`navigator.credentials.create()`の実行時に渡したパラメータがそのまま渡されるようです。

### user

> This PublicKeyCredentialUserEntity data structure describes the user account to which the new public key credential will be associated at the RP. It contains an RP-specific user account identifier, (optionally) a user name, (optionally) a user display name, and (optionally) a URL referencing a user icon image (of a user avatar, for example). The authenticator associates the created public key credential with the account identifier, and MAY also associate any or all of the user name, user display name, and image data (pointed to by the URL, if any).

とのことで、新しい公開鍵 Credential が紐づけられるユーザーの情報が含まれます。最終的には、以下のようなデータが渡されます。

```txt
{
  // Required
  "id": BufferSource, // ユーザーを一意に識別するID
  // Optionally
  "displayName": "John P. Smith", // 表示名
  "name": "john.p.smith@example.com" // ユーザー名
}
```

`navigator.credentials.create()`の実行時に渡したパラメータがそのまま渡されるようです。

### pubKeyCredParams

> A sequence of CBOR maps consisting of pairs of PublicKeyCredentialType (a string) and cryptographic algorithm (a positive or negative integer), where algorithm identifiers are values that SHOULD be registered in the IANA COSE Algorithms registry [IANA-COSE-ALGS-REG]. This sequence is ordered from most preferred (by the RP) to least preferred.

Credential のタイプと署名のアルゴリズムを指定する項目となっています。最終的には、以下のようなデータが渡されます。

```txt
[
  {
    "alg": -7,
    "type": "public-key"
  }
  //...
]
```

`navigator.credentials.create()`の実行時に渡したパラメータがそのまま渡されるようです。

## 4. `authenticatorMakeCredential()`を実行した結果

`authenticatorMakeCredential()`を実行すると以下のようなデータがブラウザ(JavaScript Application)に返却されます。

```txt
{
  "id": "xTzphZPuJyfW12TAT…",
  "rawId": ArrayBuffer(64) {},
  "response": {
    "attestationObject": ArrayBuffer(1024),
    "clientDataJSON": ArrayBuffer(116) {}
  },
  "type": "public-key"
}
```

それぞれのプロパティを見ていきましょう。

| パラメータ名      | 概要                                                                                                                       |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------- |
| id                | rawId を base64url エンコードしたもの                                                                                      |
| rawId             | Credential 毎に一意に定められている                                                                                        |
| attestationObject | 認証用の公開鍵や CredentialID、署名などが含まれている                                                                      |
| clientDataJSON    | challenge, oritin, type などが含まれている clientData を JSON シリアライズしたもの(ref: [clientDataHash](#clientDataHash)) |
| type              | `public-key`固定                                                                                                           |

このうち、`attestationObject`, `clientDataJSON`を Relying Party へ送信し検証が成功すれば登録処理は完了です。

# 参考

- [Web Authentication API](https://developer.mozilla.org/ja/docs/Web/API/Web_Authentication_API)
- [Client to Authenticator Protocol (CTAP)](https://fidoalliance.org/specs/fido-v2.0-id-20180227/fido-client-to-authenticator-protocol-v2.0-id-20180227.html)

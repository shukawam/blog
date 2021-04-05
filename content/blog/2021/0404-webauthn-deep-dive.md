+++
title = "WebAuthn DeepDive #2 - Attestation"
author = "Shuhei, Kawamura"
date = "2021-04-04"
tags = ["WebAuthn", "CTAP"]
categories = ["tech"]
draft = "false"
[[images]]
  src = "img/2021/0323/webauthn1-960x480.png"
  alt = "webauthn"
  stretch = "stretchH"
+++

# 始めに

[WebAuthn](https://webauthn.io/) について学習したときのメモ。今回は、[Attestation](https://www.w3.org/TR/webauthn-1/#sctn-attestation) についてです。

# About Attestation

雑に言うと、FIDO2 のユーザー登録フローにおいて認証器で非対称の鍵ペア(秘密鍵、公開鍵)が生成されるが、その公開鍵がきちんと FIDO2 認証ベンダーの認証器から生成されたものか？どうかを検証する仕組みのこと。認証器は出荷時に FIDO2 認定ベンダーより認証器内のセキュア領域にベンダー固有の秘密鍵を埋め込まれるが、その秘密鍵でユーザー登録時に生成した公開鍵に対して署名を行う。最終的に RP(Relying Party)では、その署名をベンダー固有の公開鍵(ルート証明書)を用いる事で検証し、送信されてきた公開鍵の妥当性を検証することでユーザーの登録可否を決定する。

# Deep Dive

まずは、原文を読んでみましょう。

> Authenticators MUST also provide some form of attestation. The basic requirement is that the authenticator can produce, for each credential public key, an attestation statement verifiable by the WebAuthn Relying Party. Typically, this attestation statement contains a signature by an attestation private key over the attested credential public key and a challenge, as well as a certificate or similar data providing provenance information for the attestation public key, enabling the Relying Party to make a trust decision. However, if an attestation key pair is not available, then the authenticator MUST perform self attestation of the credential public key with the corresponding credential private key. All this information is returned by authenticators any time a new public key credential is generated, in the overall form of an attestation object. The relationship of the attestation object with authenticator data (containing attested credential data) and the attestation statement is illustrated in figure 5, below.

雑に訳すと、

- Authenticator(認証器)は、何らかの認証を提供する必要がある(生成した公開鍵の出所を証明すること)
- Authenticator(認証器)の満たすべき基本的な要件は以下
  - クレデンシャル公開鍵(登録時に生成される公開鍵)について、WebAuthn RP が検証可能な証明書(`attestation statement`)を作成できる事
  - `attestation statement`には以下の情報を含む(RP はこれらの情報を用いて、公開鍵の出所の妥当性を検証を行う)
    - FIDO2 認証ベンダーから認証器内のセキュア領域に埋め込まれた秘密鍵によって生成された署名
    - RP から発行された challenge
    - 公開鍵の出所情報を提供する証明書(もしくはそれに値する情報)

となる。何となく、FIDO2 における Attestation がどういうものだか分かってきたので、WebAuthn 実行後に認証器から返却されるデータを振り返ってみましょう。こんなデータが返ってきます。

```txt
{
  "id": "xTzphZPuJyfW12TAT…",
  "rawId": ArrayBuffer(64) {},
  "response": {
    "attestationObject": ArrayBuffer(1024),
    "clientDataJSON": ArrayBuffer(116) {}
  },
  "type": "public-key"
}
```

今回ちゃんと仕様を読んでいくのは、このうち`attestationObject`ということになります。

## Attestation Object

`attestationObject`は、CBOR[^1]で表現されており、W3C で仕様が公開されています。具体的には以下の通り。

![attestation-object](http://localhost:1313/blog/img/2021/0404/attestation-object.png)

もう少し、開発者に分かりやすく JSON 形式で表現すると以下のようになる。

```json
[
  {
    "fmt": "packed",
    "attStmt": {
      "alg": -7,
      "sig": {
        "type": "Buffer",
        "data": "..."
      },
      "x5c": [
        {
          "type": "Buffer",
          "data": "..."
        }
      ]
    },
    "authData": {
      "type": "Buffer",
      "data": "..."
    }
  }
]
```

各プロパティ内に含まれるバイナリデータまで見ていきましょう。

### fmt

[Defined Attestation Statement Formats](https://www.w3.org/TR/webauthn/#sctn-defined-attestation-formats)によると、以下の種類があるそうです。

| Format                                                                                  | Attestation Support Type | 説明                                                                                    |
| --------------------------------------------------------------------------------------- | ------------------------ | --------------------------------------------------------------------------------------- |
| [Packed](https://www.w3.org/TR/webauthn/#sctn-packed-attestation)                       | Basic, Self, AttCA       | WebAuthn に最適化されたフォーマット                                                     |
| [TPM](https://www.w3.org/TR/webauthn/#sctn-tpm-attestation)                             | AttCA                    | Trusted Platform Module を暗号エンジンとして使用する認証器で使用される                  |
| [Android Key](https://www.w3.org/TR/webauthn/#sctn-android-key-attestation)             | Basic                    | 対象となる認証器が Android N(Nougat) 移行のプラットフォーム認証器の場合に使用される     |
| [Android SafetyNet](https://www.w3.org/TR/webauthn/#sctn-android-safetynet-attestation) | Basic                    | 認証器が特定の Android プラットフォーム上の認証器の場合、SafetyNet API に基づいても良い |
| [FIDO U2F](https://www.w3.org/TR/webauthn/#sctn-fido-u2f-attestation)                   | Basic, AttCA             | FIDO U2F 認証器で使用される                                                             |
| [None](https://www.w3.org/TR/webauthn/#sctn-none-attestation)                           | None                     | Relying Party が認証情報を受け取ることを希望しない場合に使用する                        |
| [Apple Anonymous](https://www.w3.org/TR/webauthn/#sctn-apple-anonymous-attestation)     | Anonymization CA         | Apple が WebAuthn をサポートする特定の種類の Apple デバイスに対して独占的に使用される   |

上の Attestation Object の構造の所で先出ししてしまいましたが、今回は`Packed`の場合のみ取り扱います。

### Attedted Credential Data

下図の赤枠でかこっている箇所の話です。

![attested-credential-data](http://localhost:1313/blog/img/2021/0404/attested_credential_data.png)

`authData`内に含まれる Attested Credential Data は、以下のような構成になっています。

| プロパティ名        | バイト長 | 説明                                                                            |
| ------------------- | -------- | ------------------------------------------------------------------------------- |
| aaguid              | 16       | 認証器のタイプを示す 128bit(16byte)の識別子のこと                               |
| credentialIdLength  | 2        | クレデンシャル ID のバイト長を示す 16bit(2byte)の符号なしビッグエンディアン整数 |
| credentialId        | L        | クレデンシャルを一意に識別する ID                                               |
| credentialPublicKey | variable | COSE_Key 形式でエンコードされたクレデンシャル公開鍵(認証器で生成された公開鍵)   |

ここに COSE_Key 形式でエンコードされた以降の認証フローで使用するための公開鍵が含まれているので、検証終了時に Database で永続化しておきます。

### attStmt

下図の赤枠でかこっている箇所の話です。

![attStmt](http://localhost:1313/blog/img/2021/0404/att-stmt.png)

何やら、FIDO2 認証ベンダーの秘密鍵によって生成された署名や証明書が含まれていそうです。きちんとプロパティを見ていくと、以下のようになっています。(今回は Basic の場合)

| プロパティ名 | 説明                                                                                        |
| ------------ | ------------------------------------------------------------------------------------------- |
| alg          | COSE Algorithm Identifier で指定する sig(署名)の暗号化アルゴリズム                          |
| sig          | FIDO 認証ベンダー固有の秘密鍵によって生成された署名                                         |
| x5c          | attestnCert (とその証明書チェーン)が含まれていて、それぞれ X.509 形式でエンコードされている |

署名の検証手順については、[6.5.2. Attestation Statement Formats](https://www.w3.org/TR/webauthn/#sctn-attestation-formats)に記されています。(OSS のライブラリの開発などやフルスクラッチで実装しようという方以外は知らなくても大丈夫な部分かと思われます)

[^1]: Concise Binary Object Representation (CBOR)の略で、バイナリの JSON のようなもの

# 参考

- [W3C WebAuthn](https://www.w3.org/TR/webauthn/)

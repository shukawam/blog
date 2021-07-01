+++
title = "MicroProfile - JWT RBAC"
author = "Shuhei, Kawamura"
date = "2021-05-14"
tags = ["MicroProfile", "Microservices", "Authentication", "Authorization", "Java", "Helidon"]
categories = ["tech"]
draft = "true"
[[images]]
  src = ""
  alt = ""
  stretch = "stretchH"
+++

# 始めに

以前、[マイクロサービスの認証・認可](Microservicesの認証・認可)という記事を書きました。マイクロサービスの認証・認可の設計における代表的なパターンの紹介だったのですが、今回は Java で実現するにはどうすればいいのか？といった内容となっています。

# Eclipse MicroProfile

Java EE の仕様策定のプロセスが遅いといった事から、複数ベンダーと Java コミュニティーによって生まれた仕様のことで、Polyglot な Microservices 環境の中で協調して動く(=Microservices 各種仕様に準拠する、Microservices のデザインパターンを実装する)Java アプリケーションの開発に必要な機能をフレームワークとして規定しています。  
最新版は、4.0.1(2021/05/14 現在)で以下のコンポーネントを有しています。

![microprofile-component](http://localhost:1313/blog/img/2021/0514/microprofile-component.png)

※図は、[https://microprofile.io/](https://microprofile.io/)を引用しています。

Jakarta(Java) EE でおなじみの

- CDI
- JSON-P
- JAX-RS
- JSON-B

といった機能に加え、マイクロサービス実装のための

- Open Tracing
- Open API
- Rest Client
- Config
- Fault Tolerance
- Metrics
- JWT Propagation
- Helath

といった機能を有しています。この中でも **JWT Propagation** に焦点を当てて見ていきたいと思います。

# Eclipse MicroProfile Interoperable JWT RBAC

## Summary

- Eclipse MicroProfile では、Client Side Token を使用した仕様が定められている(セッションを共有するような方式ではない)
  - トークンは OpenID Connect(OIDC)ベースの JSON Web Token(JWT)を使用する
- マイクロサービスのエンドポイントに対して、Role Base Access Control(RBAC)を行う

## MP-JWT

Java アプリケーションに組み込むこと、他のマイクロサービスアプリケーションと協調して動作することのために必要な要件は以下のように定められている。

- 認証トークン(authentication token)として使用可能であること
- Java EE アプリケーションで認可トークン(authorization token)として使用可能であること
- [JSR375](https://www.jcp.org/en/jsr/detail?id=375)の identityStore にマッピング可能であること
- [IANA JWT Assignments](https://www.iana.org/assignments/jwt/jwt.xhtml)に記載されている追加の標準的なクレーム、非標準的なクレームをサポートできること

上記の要件を満たすために、MP-JWT では一般的な JWT に加え以下のクレームが追加されています。

| claim  | description                                                                                                                                                 |
| ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| upn    | トークンのサブジェクト、ユーザープリンシパルをトークンがアクセスする MicroProfile サービス全体で一意に識別するためのクレーム(human readable)                |
| groups | MP-JWT のプリンシパルに割り当てられるグループ名のリストで、通常はアプリケーション・コンテナ・レベルでのアプリケーション・ロールへのマッピングを必要とします |

## MP-JWT - Headers and Claims

### MP-JWT Headers

MP-JWT の header 部は一般的には以下の claim で構成されています。

| header | description                                                |
| ------ | ---------------------------------------------------------- |
| \*alg  | JWT のセキュリティに使用される暗号アルゴリズムを指定します |
| \*enc  | クレームやネストされた JWT を暗号化する場合に使用する      |
| typ    | トークンのタイプを指定する claim で`JWT`で固定             |
| kid    | JWT のセキュリティにどの鍵が使われたかを示すヒント         |

\*: 必須項目

### MMP-JWT Claims

MP-JWT の header 部は一般的には以下の claim で構成されています。

| claim  | description                                                                                                                |
| ------ | -------------------------------------------------------------------------------------------------------------------------- |
| \*iss  | JWT の発行者を示す項目                                                                                                     |
| \*iat  | JWT の発行された時刻を示す項目                                                                                             |
| \*exp  | JWT の有効期限を示す項目                                                                                                   |
| \*upn  | ユーザープリンシパル(JWT 内に含まれていない場合は、`preferred_username`か`sub`を参照するように JWT の検証元は設計するべき) |
| sub    | ユーザープリンシパル                                                                                                       |
| jti    | JWT の一意の識別子を表す                                                                                                   |
| aud    | JWT によりアクセスが可能なエンドポイントを表す                                                                             |
| groups | エンドポイントが認可を必要とする場合に持つことを推奨されるが、他の claim から groups にマッピングすることができる。        |

\*: 必須項目

MP-JWT の具体例は以下の通りです。

```json
{
     "typ": "JWT",
    "alg": "RS256",
    "kid": "abc-1234567890"
}
{
    "iss": "https://server.example.com",
    "jti": "a-123",
    "exp": 1311281970,
    "iat": 1311280970,
    "sub": "24400320",
    "upn": "jdoe@server.example.com",
    "groups": ["red-group", "green-group", "admin-group", "admin"],
}
```

MicroProfile アプリケーションで使用する場合には、`java.security.Principal`を拡張した`org.eclipse.microprofile.jwt.JsonWebToken`から`getXxx`が提供されているのでそこからアクセスします。  
JWT には、カスタムクレームを含める事ができるが追加したクレームは、`JsonWebToken#getOtherClaim(String)`を使って取得することができます。

MP-JWT の仕様の説明は以上となります。ここからは実際にアプリケーションに組み込む際にどのように実装していけばよいか見ていきましょう。

# Getting Started MP-JWT with Helidon MP

Java のアプリケーションフレームワークとして、Helidon を使いますが MicroProfile 準拠のフレームワークであれば何でも良いです。IDaaS まで含めた全体の構成は以下のようになります。

![mp-jwt-architecture](http://localhost:1313/blog/img/2021/0514/architecture.png)

- IDCS: Oracle の提供するアイデンティティ・アクセス管理基盤です
- Auth: OpenID Connect でログインするためのサービスです。今回は、Helidon の Security Provider を用いて実装します
- Employee: 従業員情報を取り扱うためのサービスです。図では、HTTP Method で制限を書けているように見えますが、大きく参照系(GET) -> 認証済みのユーザーであれば誰でも実行可能、更新系(POST/PUT/DELETE) -> 認証済みの admin ロールのユーザーのみ実行可能という形で制限をかけたいと思います

流石に、全量は紹介できないのでポイントを絞って解説します。記載されていない箇所を参照したい場合はリポジトリを参照してください。

# 終わりに

# 参考

- [Eclipse MicroProfile Interoperable JWT RBAC](https://download.eclipse.org/microprofile/microprofile-jwt-auth-1.2/microprofile-jwt-auth-spec-1.2.html)

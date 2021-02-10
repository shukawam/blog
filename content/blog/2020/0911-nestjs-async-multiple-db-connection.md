+++
title = "Nest.jsで接続先情報を環境変数から非同期で取得する"
author = "Shuhei, Kawamura"
date = "2020-09-11"
tags = ["node", "nest"]
categories = ["tech"]
draft = "false"
[[images]]
  src = "img/2020/0911/nestjs.png"
  alt = "Nest.js"
  stretch = "stretchH"
+++

# 始めに

※ブログを一か所にまとめるため、以前 Qiita に投稿した記事の移行しています。

Nest.js で環境ごとにデータベースの接続先を分けるために、接続情報を実行環境の環境変数から非同期で取得するサンプルを作成します。

# 環境

- Node.js v12.14.1
- Nest.js v6.7.2
- TypeORM v0.2.22
- PostgreSQL v11.6

# 実装手順

## 必要最小限の実装

参考）[Nest.js Document > TECHNIQUES >Database](https://docs.nestjs.com/techniques/database)

### ライブラリインストール

TypeORM, Database Driver (PostgreSQL)をインストールする。

```bash
$ npm install @nestjs/typeorm typeorm pg
```

### DB 接続情報を定義

**app.module.ts**にデータベースの接続情報を定義する。

```tsx
import { Module } from "@nestjs/common";
import { TypeOrmModule } from "@nestjs/typeorm";
import { ItemModule } from "./item/item.module";
import { Connection } from "typeorm";
import { join } from "path";

@Module({
  imports: [
    ItemModule,
    // DBの接続情報を定義
    TypeOrmModule.forRoot({
      type: "postgres",
      host: "localhost",
      port: 5432,
      username: "postgres",
      password: "postgres",
      database: "postgres",
      entities: [join(__dirname + "/**/*.entity{.ts,.js}")],
      synchronize: false,
    }),
  ],
})
export class AppModule {}
```

一番シンプルな書き方です。自分一人しか触らず、環境もこれだけ！ということであればこの書き方で良いでしょう。

しかし、実際の開発では個人の開発環境、テスト環境、ステージング環境、本番環境と複数の環境が存在し、上記のような実装では環境ごとに接続情報をハードコードし直し → ビルド → デプロイという手順を踏む必要がありナンセンスです。

そのため、通常は環境変数に定義し、接続情報はその環境変数を参照し作成します。

と、いうことで環境変数を参照するように**app.module.ts**を修正します。

## 【失敗】環境変数を参照するように実装を修正

**※この方法では実行時に依存関係が解決できずエラーとなります。**

参考）[Nest.js Document > TECHNIQUES > Configuration](https://docs.nestjs.com/techniques/configuration)

### ライブラリインストール

環境変数を参照するために必要なライブラリをインストールします。

```bash
$ npm install @nestjs/config
```

### ダミーの環境変数を用意

本来は、環境変数に定義するのですがサンプル実装なので環境変数ファイル（**.env**）をプロジェクトルートに作成します。

```env
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=postgres
DATABASE_NAME=postgres
```

### 環境変数を参照するように接続定義を修正

```tsx
import { Module } from "@nestjs/common";
import { TypeOrmModule } from "@nestjs/typeorm";
import { ConfigModule, ConfigService } from "@nestjs/config";
import { Connection } from "typeorm";
import { join } from "path";

@Module({
  imports: [
    ConfigModule.forRoot({
      envFilePath: ".env",
      isGlobal: true,
      // ignoreEnvFile: true, <- 環境変数から取得する場合はコメントアウトを外す。
    }),
    // 非同期で環境変数から値を取得し、接続情報を作成する。
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => ({
        type: "postgres" as "postgres",
        host: configService.get("DATABASE_HOST"),
        port: Number(configService.get("DATABASE_HOST")),
        username: configService.get("DATABASE_USERNAME"),
        password: configService.get("DATABASE_PASSWORD"),
        database: configService.get("DATABASE_NAME"),
        entities: [join(__dirname + "/**/*.entity{.ts,.js}")],
        synchronize: false,
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {
  constructor(private readonly connection: Connection) {}
}
```

起動後、以下のエラーが発生。

```bash
2:25:34 PM - Found 0 errors. Watching for file changes.
[Nest] 19111   - 01/13/2020, 2:25:35 PM   [NestFactory] Starting Nest application...
[Nest] 19111   - 01/13/2020, 2:25:35 PM   [InstanceLoader] TypeOrmModule dependencies initialized +24ms
[Nest] 19111   - 01/13/2020, 2:25:35 PM   [InstanceLoader] ConfigModule dependencies initialized +1ms
[Nest] 19111   - 01/13/2020, 2:25:35 PM   [ExceptionHandler] Nest can't resolve dependencies of the TypeOrmModuleOptions (?). Please make sure that the argument ConfigService at index [0] is available in the TypeOrmCoreModule context.

Potential solutions:
- If ConfigService is a provider, is it part of the current TypeOrmCoreModule?
- If ConfigService is exported from a separate @Module, is that module imported within TypeOrmCoreModule?
  @Module({
    imports: [ /* the Module containing ConfigService */ ]
  })
 +1ms
Error: Nest can't resolve dependencies of the TypeOrmModuleOptions (?). Please make sure that the argument ConfigService at index [0] is available in the TypeOrmCoreModule context.

Potential solutions:
- If ConfigService is a provider, is it part of the current TypeOrmCoreModule?
- If ConfigService is exported from a separate @Module, is that module imported within TypeOrmCoreModule?
  @Module({
    imports: [ /* the Module containing ConfigService */ ]
  })

    at Injector.lookupComponentInExports (/home/kawamura/docker/docker-services/sample-app/sample-back/node_modules/@nestjs/core/injector/injector.js:185:19)
    at processTicksAndRejections (internal/process/task_queues.js:94:5)
    at async Injector.resolveComponentInstance (/home/kawamura/docker/docker-services/sample-app/sample-back/node_modules/@nestjs/core/injector/injector.js:142:33)
    at async resolveParam (/home/kawamura/docker/docker-services/sample-app/sample-back/node_modules/@nestjs/core/injector/injector.js:96:38)
    at async Promise.all (index 0)
    at async Injector.resolveConstructorParams (/home/kawamura/docker/docker-services/sample-app/sample-back/node_modules/@nestjs/core/injector/injector.js:111:27)
    at async Injector.loadInstance (/home/kawamura/docker/docker-services/sample-app/sample-back/node_modules/@nestjs/core/injector/injector.js:78:9)
    at async Injector.loadProvider (/home/kawamura/docker/docker-services/sample-app/sample-back/node_modules/@nestjs/core/injector/injector.js:35:9)
    at async Promise.all (index 3)
    at async InstanceLoader.createInstancesOfProviders (/home/kawamura/docker/docker-services/sample-app/sample-back/node_modules/@nestjs/core/injector/instance-loader.js:41:9)
```

環境変数を参照するための`ConfigService`が`TypeOrmModuleOptions`内で依存関係が解決できないことが原因らしい。

同じような事象が Github の Issue にあがっていたので参考に載せておきます。

[Can't init TypeOrmModule using factory and forRootAsync](https://github.com/nestjs/nest/issues/1119#)

## 【成功】環境変数を参照するように実装を修正

**app.module.ts**を以下のように修正します。

```tsx
import { Module } from "@nestjs/common";
import { TypeOrmModule } from "@nestjs/typeorm";
import { ConfigModule } from "@nestjs/config";
import { TypeOrmConfigService } from "./common/database/type-orm-config.service";

@Module({
  imports: [
    ConfigModule.forRoot({
      envFilePath: ".env",
      isGlobal: true,
      // ignoreEnvFile: true, <- 環境変数から取得する場合はコメントアウトを外す。
    }),
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      // 接続情報を作成するServiceクラスを定義
      useClass: TypeOrmConfigService,
    }),
  ],
})
export class AppModule {}
```

接続情報を作成する Service クラスを生成します。

```tsx
import { TypeOrmOptionsFactory, TypeOrmModuleOptions } from "@nestjs/typeorm";
import { Injectable } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { join } from "path";

/**
 * DBの接続情報を作成するServiceクラスです。
 */
@Injectable()
export class TypeOrmConfigService implements TypeOrmOptionsFactory {
  /**
   * DBの接続設定を環境変数をもとに作成します。
   * 環境変数に設定されていない場合は、デフォルトの設定値を返却します。
   * @returns 接続情報
   */
  createTypeOrmOptions(): TypeOrmModuleOptions {
    const configService = new ConfigService();
    return {
      type: "postgres" as "postgres",
      host: configService.get("DATABASE_HOST", "localhost"),
      port: Number(configService.get("DATABASE_PORT", 5432)),
      username: configService.get("DATABASE_USERNAME", "postgres"),
      password: configService.get("DATABASE_PASSWORD", "postgres"),
      database: "postgres" as "postgres",
      entities: [join(__dirname + "../**/*.entity{.ts,.js}")],
      synchronize: false,
    };
  }
}
```

`ConfigService`を DI するのではなく、自分で new するのがポイントです。

# 最後に

今回のサンプル実装はこちらの[リポジトリ](https://github.com/shukawam/nestjs-async-multiple-connection-settings)で公開しています。

最近使い始めたのですが、素晴らしいフレームワークだとひしひしと感じております。
フレームワーク自体の良さは[こちら](https://qiita.com/kmatae/items/5aacc8375f71105ce0e4)の記事で紹介されています。

# 参考

- [NestJs 公式ドキュメント](https://docs.nestjs.com/)

- [Can't init TypeOrmModule using factory and forRootAsync](https://github.com/nestjs/nest/issues/1119#)

- [Nest.js は素晴らしい](https://qiita.com/kmatae/items/5aacc8375f71105ce0e4)

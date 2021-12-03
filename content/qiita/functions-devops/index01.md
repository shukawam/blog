+++
title = ""
author = "Shuhei, Kawamura"
date = ""
tags = []
categories = []
draft = "true"
+++

## はじめに

こちらの記事は、[Oracle Cloud Infrastructure Advent Calendar 2021](https://qiita.com/advent-calendar/2021/oci) Day 4 の記事として書かれています。カレンダー 2 の 3 日目に [OracleCloudでJenkinsCI/CDデプロイ環境作成](https://cloudii.jp/news/blog/oracle-cloud/oraclecloud%e3%81%a7jenkinsci-cd%e3%83%87%e3%83%97%e3%83%ad%e3%82%a4%e7%92%b0%e5%a2%83%e4%bd%9c%e6%88%90/)という記事がありましたが、私の方では OCI Native な CI/CD サービスについて書きたいと思います。
（内容はほとんど変わらないと思いますが）OKE の方はきっと誰かが書いてくれることを願って私の方では Oracle Functions に特化した内容で書きたいと思います。内容が結構ボリューミーになりそうなので全 2 回に分けたいと思います。

1. **事前準備編** ← 本記事はこれ
2. ビルド、デプロイメント・パイプライン作成編

本記事は、事前準備編です。

# 今回作る環境

全 2 回を通してこのような環境を作ってみます。

![image01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/1d4895a8-f8ec-79de-2768-39b9c03172f9.png)

ソースコード本体は、コミュニケーション機能（Issue, Wiki, etc.）が充実している GitHub, GitLab で管理し、OCI へデプロイするためにソースコードを Code Repository へミラーリングする実際の開発現場でよく取られそうな構成です。

# 実際に作ってみる

## 前提

以下の前提で書いていきたいと思います。

- DevOps サービスを使用する[前提条件](https://docs.oracle.com/ja-jp/iaas/Content/devops/using/getting_started.htm#prereq)を満たしていること
  - テナンシへのアクセス権を有していること、ポリシー関連の設定が完了していること
- GitHub/GitLab のアカウントを有していること
  - 本記事では、GitHub 前提で書きますが GitLab でも読み替えれば同じように実施可能です
- Fn CLI, Oracle Functions のセットアップが済んでいること
  - 完了していない場合は、[Fn Project ハンズオン](https://oracle-japan.github.io/ocitutorials/cloud-native/fn-for-beginners/), [Oracle Functions ハンズオン](https://oracle-japan.github.io/ocitutorials/cloud-native/functions-for-beginners/)を参考にセットアップを行ってください

## GitHub 側の設定

### Personal Access Token(PAT)を発行する

OCI DevOps から作成した外部リポジトリ（GitHub）へアクセスするために Personal Access Token(以下、PAT)を発行します。GitHub で、**Settings** > **Developer settings** > **Personal Access Token** とアクセスし、**Generate new token** をクリックします。

![image02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/d4e26483-5892-9d45-f259-03b8ea7b7b61.png)

以下のように設定し、PAT を発行します。

- Note: OCI DevOps
- Expiration: No expiration
  - 簡易的に無期限に設定していますが、気になる方は適当な期限に設定すると良いでしょう
- Select scopes: repo にチェック

![image03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/f5eac989-c542-71b8-8db4-8285044de391.png)

発行されると、トークンの文字列が表示されるのでメモ帳などに控えておきます。（再度表示されないので、コピーし忘れた方は再発行してください）

![image04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/0e2e8d4b-1c7b-eb80-97c3-963bd60c73e2.png)

これで、GitHub 側の設定は完了です！

## OCI 側の設定

### 認証トークンを発行する

OCIR へアクセスするための認証トークンを発行します。OCI コンソール右上の人型アイコンをクリックし、ユーザーの詳細を開きます。

![image05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/dd3d39cb-72d8-be93-e419-29bb58f4ba87.png)

リソース内の**認証トークン**をクリックします。

![image06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/0a6a227e-1dc4-0938-30e6-43ab3d7b8833.png)

**トークンの生成**をクリックし、以下のように入力して認証トークンを生成します。

- 説明: for OCIR

生成されたランダムな文字列は、再度表示されないのでメモ帳などに控えておいてください。

### OCI Vault に PAT と認証トークンを格納する

事前の手順までに生成した PAT と認証トークンを OCI Vault のシークレットに格納します。OCI コンソール左上のハンバーガーメニューから**アイデンティティとセキュリティ** > **ボールト**と選択します。

![image07.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/51ccc3bf-e019-e0ee-e5e0-7fad68e6e492.png)

**ボールトの作成**をクリックし、以下のように入力して、Vault を新規に作成します。

- 名前: devops-vault
  - 便宜上名前を定めているだけなので、自由に命名してください

![image08.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/da11820c-3337-5e75-46c0-67327e6478f3.png)

作成が完了したら、ボールトの詳細画面からマスター暗号キーを作成します。リソースから**マスター暗号化キー**を選択し、**キーの作成**をクリックします。

![image09.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/078b6585-2c90-ef94-6b93-dcbfb2df8b83.png)

以下のように入力し、マスター暗号化キーを作成します。

- 保護モード: HSM
- 名前: devops-master-key
- キーのシェイプ - アルゴリズム: AES
- キーのシェイプ - 長さ: 256 ビット

次に、ボールトの詳細画面からシークレットを作成します。リソースから**シークレット**を選択し、**シークレットの作成**をクリックします。

![image10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/1ee92384-5a95-bf29-8221-699bbe8c384b.png)

以下のように入力し、シークレット(ocir_secret, github_pat)を作成します。

ocir_secret:

- コンパートメント: 任意のコンパートメント（OCI DevOps や Oracle Functions のリソースが含まれているコンパートメント）
- 名前: ocir_secret
- 暗号化キー: 前述の手順で作成したマスター暗号化キー（devops-master-key）を選択する
- シークレット・タイプのテンプレート: プレーン・テキスト
- シークレット・コンテンツ: 発行済みの認証トークンを入力する

![image11.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/83bd0d8b-530e-dc2c-e357-97c553695f6f.png)

github_pat:

- コンパートメント: 任意のコンパートメント（OCI DevOps や Oracle Functions のリソースが含まれているコンパートメント）
- 名前: github_pat
- 暗号化キー: 前述の手順で作成したマスター暗号化キー（devops-master-key）を選択する
- シークレット・タイプのテンプレート: プレーン・テキスト
- シークレット・コンテンツ: 発行済みの PAT(Personal Access Token)を入力する

![image12.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/aa30c489-6875-0868-914d-1f1cbd2ae926.png)

これで、今回使用する Vault の設定は完了です。

### OCIR にリポジトリを作成する

ビルドした関数コードの Docker Image が格納されるリポジトリを OCIR に作成します。OCI コンソール左上のハンバーガーメニューから**開発者サービス** > **コンテナ・レジストリ**と選択します。

![image13.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/5ef8eb23-50e0-b17b-0d88-2da0894fc290.png)

**リポジトリの作成**をクリックし、以下のように入力し新規にリポジトリを作成します。

- コンパートメント: 任意のコンパートメント（OCI DevOps や Oracle Functions のリソースが含まれているコンパートメント）
- リポジトリ名: devops/fn-hello
- アクセス: プライベート

![image14.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/39aa5e70-0e64-5778-efe3-fcc687d890d3.png)

これで、OCIR の設定は完了です。

### Oracle Functions のアプリケーションを作成する

関数コードの管理単位であるアプリケーションを作成します。OCI コンソール左上のハンバーガーメニューから**開発者サービス** > **アプリケーション**と選択します。

![image15.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/6a4d5302-56f3-3047-3d37-ff8cc502d44f.png)

以下のように入力し、新規にアプリケーションを作成します。

- 名前: devops-app
- VCN: 作成済みの VCN
- サブネット: 作成済みのサブネット

![image16.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/560068/da828cdc-73b4-3518-b0c7-c05d2b2cf986.png)

これでアプリケーションの作成は完了です。

### Notifications の設定をする

OCI DevOps では、プロジェクト作成時に通知を行う、行わないに関わらず Notifications の Topic を設定する必要があります。作成方法は、[OCI Tutorials - モニタリング機能で OCI リソースを監視する](https://oracle-japan.github.io/ocitutorials/intermediates/monitoring-resources/#4-%E3%82%A2%E3%83%A9%E3%83%BC%E3%83%A0%E3%81%AE%E9%80%9A%E7%9F%A5%E5%85%88%E3%81%AE%E4%BD%9C%E6%88%90)に詳しく書いてあるので、そちらをご参照下さい。

## デプロイする関数コード

デプロイする関数コードを作成します。ここでは、単純に "Hello world!" の文字列を返す関数コードとします。

```bash
fn init --runtime java11 fn-hello
```

ローカルで動作確認を行います。

```bash
fn start -d
```

デプロイする。

```bash
fn deploy --app devops-app
```

実行する。

```bash
fn invoke devops-app fn-hello
```

"Hello, world!" という文字列が返ってくれば完了です。

### リポジトリを作成する

Oracle Functions にデプロイする関数コードを管理するリポジトリを作成します。名前は何でも良いですが、本記事では、便宜上`fn-examples`という名前で進めていきます。ここに先ほど作成した関数コードを含めてください。手順は省略しますが、こんなイメージです。

```bash
./fn-examples
└── fn-hello
    ├── build_spec.yaml // OCI DevOpsの設定ファイル（後述の手順で作成）
    ├── func.yaml // Oracle Functions(Fn Project)の設定ファイル
    ├── pom.xml
    └── src
```

# 終わりに

これで OCI DevOps を使用するための環境が整ったので、別エントリにてビルド、デプロイメントパイプラインを作成していきます。

# 参考

- [DevOps ドキュメント](https://docs.oracle.com/ja-jp/iaas/Content/devops/using/home.htm)
- [OCI Tutorials - モニタリング機能で OCI リソースを監視する](https://oracle-japan.github.io/ocitutorials/intermediates/monitoring-resources/)

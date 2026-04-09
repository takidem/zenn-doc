---
title: "[Terraform]Azure DevOps CI/CD 環境を整備する"
emoji: "🐈"
type: "tech"
topics:
  - "terraform"
published: false
---

# Azure DevOps CI/CD を使用した Terraform デプロイ

## この記事でやること

1. Azure DevOps Pipelines を使用した CI の整備
2. Azure DevOps Releases を使用した CD の整備
3. 1,2に必要な Azure 側の設定
4. Terraform を使用した deploy, destroy

## この記事でやらないこと

- Terraform 自体の解説

## 対象者

- パブリッククラウドにおける IaC の概念について理解している方
- Terraform の使用方法、基本的なコードが書ける乃至は読める、基本的なコマンドが分かる方
- 上記を満たさなくても調べて自己解決できる方

## 環境構成

### CI/CD 環境構成

- 開発者が編集した Terrform コードが main ブランチにマージされたことを契機として、以下のパイプラインが実行されて Azure 環境へデプロイされる。
- デプロイ・削除の際に承認者の承認が必要になるようにすることで事故を防止する作りこみを入れる。
- デプロイと削除はステージを分ける。

![](https://storage.googleapis.com/zenn-user-upload/72a91c6be688-20250405.png)

### Azure リソース環境構成

ざっと以下のようなシンプルな構成にします。
ほんとは NAT とかあるといいですが、今回は検証ということでパブリックサブネットに置きます。
tfstate ファイルを格納するためにバックエンドとなるストレージアカウントを手動でデプロイしておきます。
tfstate ファイルを管理するストレージも terraform で管理するのは NG です。

![](https://storage.googleapis.com/zenn-user-upload/9ccf8ab986b2-20250405.png)

## CI 環境構築

1. 
# Kubernetes The Hard Way on NIFCLOUD

Kubernetes クラスタを自力で組み上げる [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) を[ニフクラ](https://pfs.nifcloud.com/)上で実践するチュートリアルです。

Kubernetes クラスタのコンポーネントをセットアップしながら、理解を深めていきましょう！

## 対象の読者

> The target audience for this tutorial is someone planning to support a production Kubernetes cluster and wants to understand how everything fits together.

本番 Kubernetes クラスタを維持することを考えており、どのようにして全てがピッタリとはまっていくかを理解したいと思っている人がこのチュートリアルの対象の読者です。

## クラスタの詳細

* kubernetes v1.21.0
* containerd v1.4.4
* coredns v1.8.3
* cni v0.9.1
* etcd v3.4.15

## Labs

このチュートリアルではニフクラへのアクセスを持っていることを前提に進めますが、通常の Ubuntu 環境などに応用できることもあるかも知れません。

* [前提](docs/01-prerequisites.md)
* [クライアントツールのインストール](docs/02-client-tools.md)
* [コンピューティングリソースのプロビジョニング](docs/03-compute-resources.md)
* [認証局のプロビジョニングと TLS 証明書の生成](docs/04-certificate-authority.md)
* [認証のための Kubernetes の設定ファイルの生成](docs/05-kubernetes-configuration-files.md)
* [データ暗号化の設定ファイルと鍵の生成](docs/06-data-encryption-keys.md)
* [etcd クラスターの起動](docs/07-bootstrapping-etcd.md)
* [Kubernetes コントロールプレーンの起動](docs/08-bootstrapping-kubernetes-controllers.md)
* [Kubernetes ワーカーノードの起動](docs/09-bootstrapping-kubernetes-workers.md)
* [遠隔アクセスのための kubectl の設定](docs/10-configuring-kubectl.md)
* [Pod のネットワークルートのプロビジョニング](docs/11-pod-network-routes.md)
* [DNS クラスターアドオンのデプロイ](docs/12-dns-addon.md )
* [発煙試験](docs/13-smoke-test.md)
* [掃除](docs/14-cleanup.md)

ベースとなる Kubernetes The Hard Way バージョン: [79a3f79](https://github.com/kelseyhightower/kubernetes-the-hard-way/tree/79a3f79b27bd28f82f071bb877a266c2e62ee506)

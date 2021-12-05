# 前提

[本家 Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/01-prerequisites.md)

## nifcloud-cli の準備

ニフクラ公式の CLI ツール [nifcloud-cli](https://github.com/nifcloud/nifcloud-cli) を使用します。

```sh
pip install nifcloud-cli
```

```sh
## クレデンシャルとデフォルトで使用する任意のリージョンを指定します
$ export NIFCLOUD_ACCESS_KEY_ID=<Your NIFCLOUD Access Key ID>
$ export NIFCLOUD_SECRET_ACCESS_KEY=<Your NIFCLOUD Secret Access Key>
$ export NIFCLOUD_DEFAULT_REGION=jp-east-1
```

## tmux を使ってコマンドを並行で実行する

tmux の使用は任意ですが、使用することでコマンドを一度入力するだけで複数のインスタンスに対して同時に操作できるので使える状態にしておくことをお勧めします。

いかによく使う操作を示します。

```
## 水平分割
prefix + "

## 垂直分割
prefix + %

## ペイン間の移動
prefix + o

## 同時入力
prefix を押してから
:set-window-option synchronize-panes on と入力
```

次: [クライアントツールのインストール](02-client-tools.md)

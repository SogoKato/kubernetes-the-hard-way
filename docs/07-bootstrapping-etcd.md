# etcd クラスターの起動

[本家 Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/07-bootstrapping-etcd.md)

## Prerequisites

この Lab での作業は各 Controller インスタンス (controller0, controller1, controller2) 上で実行しなくてはいけません。

例えば以下のように SSH でログインします。

```sh
ssh controller0
```

### ホスト名を変えておく

SSH キーのパスフレーズ: `theveryhardway`

```sh
for instance in controller0 controller1 controller2; do
  ssh ${instance} sudo hostnamectl set-hostname --static ${instance}
done
```

### Running commands in parallel with tmux

再掲です。

```
## 水平分割
prefix + "

## 垂直分割
prefix + %

## ペイン間の移動
prefix + o

## スクロールモードに入る (q を押して抜ける)
prefix + [

## 同時入力
prefix を押してから
:set-window-option synchronize-panes on と入力 (やめるときは on を off にする)
```

## Bootstrapping an etcd Cluster Member

### Download and Install the etcd Binaries

```sh
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.15/etcd-v3.4.15-linux-amd64.tar.gz"
```

```sh
{
  tar -xvf etcd-v3.4.15-linux-amd64.tar.gz
  sudo mv etcd-v3.4.15-linux-amd64/etcd* /usr/local/bin/
}
```

### Configure the etcd Server

```sh
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo chmod 700 /var/lib/etcd
  sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
}
```

```sh
{
  INTERNAL_IP=$(ip a show ens224 | grep "inet " | cut -d ' ' -f6 | cut -d/ -f1)
  ETCD_NAME=$(hostname -s)
  echo ${INTERNAL_IP}
}
```

`${INTERNAL_IP}` が `10.240.0.1X` であることを確認します。

```sh
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller0=https://10.240.0.10:2380,controller1=https://10.240.0.11:2380,controller2=https://10.240.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the etcd Server

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

## Verification

```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

次: [Kubernetes コントロールプレーンの起動](08-bootstrapping-kubernetes-controllers.md)

# Pod のネットワークルートのプロビジョニング

[本家 Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/11-pod-network-routes.md)

## The Routing Table

内側の IP アドレスと Pod の CIDR レンジを確認します。

SSH キーのパスフレーズ: `theveryhardway`

```sh
for instance in worker0 worker1 worker2; do
  ssh ${instance} "ip a show ens224 | grep 'inet ' | cut -d ' ' -f6 | cut -d/ -f1; cat ~/POD_CIDR"
done
```

> 出力

```
Enter passphrase for key '/home/hoge/.ssh/K8sTHWKey.pem':
10.240.0.20
export POD_CIDR=10.200.0.0/24
Enter passphrase for key '/home/hoge/.ssh/K8sTHWKey.pem':
10.240.0.21
export POD_CIDR=10.200.1.0/24
Enter passphrase for key '/home/hoge/.ssh/K8sTHWKey.pem':
10.240.0.22
export POD_CIDR=10.200.2.0/24
```

## Routes

脳筋ですみません。

SSH キーのパスフレーズ: `theveryhardway`

```sh
for i in 1 2; do
  ssh worker0 ip r add 10.200.${i}.0/24 via 10.240.0.2${i} dev ens224
done
```

```sh
for i in 0 2; do
  ssh worker1 ip r add 10.200.${i}.0/24 via 10.240.0.2${i} dev ens224
done
```

```sh
for i in 0 1; do
  ssh worker2 ip r add 10.200.${i}.0/24 via 10.240.0.2${i} dev ens224
done
```

```sh
for instance in worker0 worker1 worker2; do
  ssh ${instance} ip route
done
```

> 出力

```
Enter passphrase for key '/home/hoge/.ssh/K8sTHWKey.pem':
...
10.200.1.0/24 via 10.240.0.21 dev ens224
10.200.2.0/24 via 10.240.0.22 dev ens224
...
Enter passphrase for key '/home/hoge/.ssh/K8sTHWKey.pem':
...
10.200.0.0/24 via 10.240.0.20 dev ens224
10.200.2.0/24 via 10.240.0.22 dev ens224
...
Enter passphrase for key '/home/hoge/.ssh/K8sTHWKey.pem':
...
10.200.0.0/24 via 10.240.0.20 dev ens224
10.200.1.0/24 via 10.240.0.21 dev ens224
...
```

次: [DNS クラスターアドオンのデプロイ](12-dns-addon.md)

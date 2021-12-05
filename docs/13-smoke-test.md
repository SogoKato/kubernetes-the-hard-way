# 発煙試験

[本家 Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/13-smoke-test.md)

## Data Encryption

```sh
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

SSH キーのパスフレーズ: `theveryhardway`

```sh
ssh controller0 sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C
```

> 出力

```
Enter passphrase for key '/home/hoge/.ssh/K8sTHWKey.pem':
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 08 87 81 5b 7c 51 39  |:v1:key1:...[|Q9|
00000050  62 2f 58 e6 9c 65 92 7a  09 06 b1 14 a8 8a cc b8  |b/X..e.z........|
00000060  99 bf 10 b8 60 4f 83 c5  97 ac 67 0c 35 2a 8a 66  |....`O....g.5*.f|
00000070  48 1c c3 51 7c 6f cd 84  06 84 96 7a e4 b3 04 82  |H..Q|o.....z....|
00000080  9b 1e f8 ed 3d 0b a8 f7  4b 6b 9f d8 7e d8 18 16  |....=...Kk..~...|
00000090  09 1c e6 48 76 16 5a cd  26 ab d6 5b ca 5f 02 24  |...Hv.Z.&..[._.$|
000000a0  8e 3c 90 80 1d a9 92 3a  ca b5 d2 ad 53 f9 92 67  |.<.....:....S..g|
000000b0  ee c8 62 3f 2b ad 94 df  bc 76 9c c4 51 c0 f2 ab  |..b?+....v..Q...|
000000c0  74 a2 4f 3e 4d 5b 3d af  23 1b a5 55 59 2f c3 93  |t.O>M[=.#..UY/..|
000000d0  c3 35 df a5 02 ed 24 0c  e0 3f 13 37 35 46 73 ad  |.5....$..?.75Fs.|
000000e0  ac d0 13 40 ed e5 95 12  5d 48 6f 15 4c 1e a8 ed  |...@....]Ho.L...|
000000f0  ce 67 15 81 71 34 1f b3  21 97 56 fc dc f4 20 64  |.g..q4..!.V... d|
00000100  25 b7 72 9a 4c 56 32 4e  62 13 47 41 c5 17 36 60  |%.r.LV2Nb.GA..6`|
00000110  86 9a 32 c4 4d 5c 33 3f  09 0d e0 5c 76 c7 12 a8  |..2.M\3?...\v...|
00000120  16 70 a9 d5 d1 23 40 8e  d5 00 63 11 69 53 1d 7d  |.p...#@...c.iS.}|
00000130  00 0f 37 1d d5 a7 5c 46  fe 44 b6 8d c9 34 c3 79  |..7...\F.D...4.y|
00000140  e9 72 17 22 f4 26 38 53  d6 b9 59 3a 19 b7 c3 4d  |.r.".&8S..Y:...M|
00000150  15 ff 6b 8e 20 d2 00 27  4b 0a                    |..k. ..'K.|
0000015a
```

## Deployments

```sh
kubectl create deployment nginx --image=nginx
```

```sh
kubectl get pods -l app=nginx
```

> 出力

```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6799fc88d8-99j8k   1/1     Running   0          13s
```

### Port Forwarding

```sh
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

ローカルマシンの `8080` ポートを `nginx` Pod の `80` に転送します。

```sh
kubectl port-forward $POD_NAME 8080:80
```

> 出力

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

新しいターミナルを開き、転送したアドレスを使って HTTP リクエストをします。

```sh
curl --head http://127.0.0.1:8080
```

> 出力

```
HTTP/1.1 200 OK
Server: nginx/1.21.4
Date: Sun, 28 Nov 2021 04:47:55 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 02 Nov 2021 14:49:22 GMT
Connection: keep-alive
ETag: "61814ff2-267"
Accept-Ranges: bytes
```

前のターミナルに戻って nginx pod へのポート転送を止める

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Logs

```sh
kubectl logs $POD_NAME
```

> 出力

```
...
127.0.0.1 - - [08/Dec/2021:09:00:00 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.68.0" "-"
```

### Exec

```sh
kubectl exec -ti $POD_NAME -- nginx -v
```

> 出力

```
nginx version: nginx/1.21.4
```

## Services

```sh
kubectl expose deployment nginx --port 80 --type NodePort
```

```sh
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

```sh
{
  PUBLIC_IP=$(curl http://ifconfig.me)
  nifcloud computing authorize-security-group-ingress \
    --group-name=K8sTHWInternal \
    --ip-permissions='[
      {"Description": "K8sTheHardWayAllowNginxService", "ListOfRequestIpRanges": [{"CidrIp": "'${PUBLIC_IP}'"}], "IpProtocol": "tcp" ,"FromPort": '${NODE_PORT}'}
    ]'
}
```

```sh
EXTERNAL_IP=$(nifcloud computing describe-instance-attribute \
  --instance-id=worker0 \
  --attribute=networkInterfaceSet \
  --output text \
  --query "NetworkInterfaceSet[0].Association.PublicIp")
```

```sh
curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

> 出力

```
HTTP/1.1 200 OK
Server: nginx/1.21.4
Date: Wed, 08 Dec 2021 09:00:00 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 02 Nov 2021 14:49:22 GMT
Connection: keep-alive
ETag: "61814ff2-267"
Accept-Ranges: bytes
```

お片付けします。

```sh
nifcloud computing revoke-security-group-ingress \
  --group-name=K8sTHWInternal \
  --ip-permissions='[
    {"ListOfRequestIpRanges": [{"CidrIp": "'${PUBLIC_IP}'"}], "IpProtocol": "tcp" ,"FromPort": '${NODE_PORT}'}
  ]'
```

次: [掃除](14-cleanup.md)

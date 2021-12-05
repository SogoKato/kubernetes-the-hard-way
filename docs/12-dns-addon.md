# DNS クラスターアドオンのデプロイ

[本家 Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/12-dns-addon.md)

## The DNS Cluster Add-on

```sh
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns-1.8.yaml
```

> 出力

```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
```

```sh
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> 出力

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-8494f9c688-h6n69   1/1     Running   0          4s
coredns-8494f9c688-prtdc   1/1     Running   0          4s
```

## Verification

```sh
kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
```

```sh
kubectl get pods -l run=busybox
```

> 出力

```
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          7s
```

```sh
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

```sh
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> 出力

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

次: [発煙試験](13-smoke-test.md)

# 遠隔アクセスのための kubectl の設定

[本家 Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/10-configuring-kubectl.md)

> この Lab は Admin client certificates を生成するのに使ったのと同じディレクトリで実行してください。

## The Admin Kubernetes Configuration File

```sh
{
  KUBERNETES_PUBLIC_ADDRESS=$(nifcloud computing nifty-describe-elastic-load-balancers \
    --elastic-load-balancers=ListOfRequestElasticLoadBalancerName=K8sTHWLB \
    --output text \
    --query 'NiftyDescribeElasticLoadBalancersResult.ElasticLoadBalancerDescriptions[0].NetworkInterfaces[0].IpAddress')

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}
```

## Verification

```sh
kubectl version
```

> 出力

```
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:25:06Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
```

```sh
kubectl get nodes
```

> 出力

```
NAME      STATUS   ROLES    AGE     VERSION
worker0   Ready    <none>   8m12s   v1.21.0
worker1   Ready    <none>   8m12s   v1.21.0
worker2   Ready    <none>   8m12s   v1.21.0
```

次: [Pod のネットワークルートのプロビジョニング](11-pod-network-routes.md)

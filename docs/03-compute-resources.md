# コンピューティングリソースのプロビジョニング

[本家 Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/03-compute-resources.md)

## Networking

### Virtual Private Cloud Network

プライベートLANを使います。

https://pfs.nifcloud.com/api/cli/nifty-create-privatelan.htm

```sh
nifcloud computing nifty-create-private-lan \
  --cidr=10.240.0.0/24 \
  --availability-zone=east-11 \
  --private-lan-name=kubernetes \
  --description=K8sTheHardWay
```

<details>
<summary>API レスポンス例</summary>

```
{
    "PrivateLan": {
        "AccountingType": "2",
        "AvailabilityZone": "east-11",
        "CidrBlock": "10.240.0.0/24",
        "CreatedTime": "2021-12-08T00:00:00+09:00",
        "Description": "K8sTheHardWay",
        "ElasticLoadBalancingSet": [],
        "InstancesSet": [],
        "NetworkId": "<truncated>",
        "NetworkInterfaceSet": [],
        "NextMonthAccountingType": "2",
        "PrivateLanName": "kubernetes",
        "RemoteAccessVpnGatewaySet": [],
        "RouterSet": [],
        "SharingStatus": "none",
        "State": "available",
        "TagSet": [],
        "VpnGatewaySet": []
    },
    "RequestId": "<truncated>"
}
```

</details>


### DHCP サーバーを設定

https://blog.pfs.nifcloud.com/privatelan_build \
https://blog.pfs.nifcloud.com/privatelan_build_dhcpserver \
https://blog.pfs.nifcloud.com/sshlogin_dnat \
https://pfs.nifcloud.com/api/rest/NiftyCreateRouter.htm

#### ルーター作成

```sh
nifcloud computing nifty-create-router \
  --router-name=router \
  --availability-zone=east-11 \
  --description=K8sTheHardWay \
  --network-interface='[{"NetworkName":"kubernetes","Dhcp":true,"IpAddress":"10.240.0.1"}]'
```

<details><summary>API レスポンス例</summary>

```
{
    "RequestId": "<truncated>",
    "Router": {
        "AccountingType": "2",
        "AvailabilityZone": "east-11",
        "BackupInformation": {
            "IsBackup": false
        },
        "Description": "K8sTheHardWay",
        "GroupSet": [],
        "NetworkInterfaceSet": [
            {
                "Dhcp": true,
                "IpAddress": "10.240.0.1",
                "NetworkId": "<truncated>",
                "NetworkName": "kubernetes"
            }
        ],
        "NextMonthAccountingType": "2",
        "RouterId": "<truncated>",
        "RouterName": "router",
        "State": "pending",
        "Type": "small",
        "VersionInformation": {
            "IsLatest": true,
            "Version": "v3.2"
        }
    }
}
```

</details>

### Firewall Rules

https://pfs.nifcloud.com/api/rest/CreateSecurityGroup.htm
https://pfs.nifcloud.com/api/rest/AuthorizeSecurityGroupIngress.htm

本家では IP アドレスを ANY (0.0.0.0) で開けていますが、セキュリティ上危ないので、SSH はクライアントの IP アドレスに絞ります。

```sh
PUBLIC_IP=$(curl http://ifconfig.me)
```

controller 用のファイアウォールグループ

```sh
nifcloud computing create-security-group \
  --group-name=K8sTHWExternal --placement=AvailabilityZone=east-11
```

```sh
nifcloud computing authorize-security-group-ingress \
  --group-name=K8sTHWExternal \
  --ip-permissions='[
    {"ListOfRequestIpRanges": [{"CidrIp": "'${PUBLIC_IP}'"}], "IpProtocol": "tcp", "FromPort": 22},
    {"ListOfRequestIpRanges": [{"CidrIp": "0.0.0.0/0"}], "IpProtocol": "tcp", "FromPort": 6443},
    {"ListOfRequestIpRanges": [{"CidrIp": "0.0.0.0/0"}], "IpProtocol": "icmp"}
  ]'
```

Worker 用のファイアウォールグループ

SSH (22) への通信および Controller インスタンスからの通信を許可しています。

```sh
nifcloud computing create-security-group \
  --group-name=K8sTHWInternal \
  --placement=AvailabilityZone=east-11
```

```sh
nifcloud computing authorize-security-group-ingress \
  --group-name=K8sTHWInternal \
  --ip-permissions='[
    {"ListOfRequestGroups": [{"GroupName": "K8sTHWExternal"}], "IpProtocol": "tcp", "FromPort": 0, "ToPort": 65535},
    {"ListOfRequestGroups": [{"GroupName": "K8sTHWExternal"}], "IpProtocol": "udp", "FromPort": 0, "ToPort": 65535},
    {"ListOfRequestGroups": [{"GroupName": "K8sTHWExternal"}], "IpProtocol": "icmp"},
    {"ListOfRequestIpRanges": [{"CidrIp": "'${PUBLIC_IP}'"}], "IpProtocol": "tcp", "FromPort": 22}
  ]'
```

### Kubernetes Public IP Address

ニフクラマルチLB に付替IPを振ることができないので、先にマルチロードバランサを作成します。

#### マルチロードバランサー作成

本家の[第8章 Provision a Network Load Balancer](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#provision-a-network-load-balancer)に該当します。

ニフクラのロードバランサ―では 6443 ポートに対して、ヘルスチェックパスを指定してヘルスチェックをすることができないので ICMP で対応します。

```sh
nifcloud computing nifty-create-elastic-load-balancer \
  --availability-zones=east-11 \
  --elastic-load-balancer-name=K8sTHWLB \
  --listeners='[{"Protocol": "TCP", "ElasticLoadBalancerPort": 6443, "Description": "K8sTheHardWay", "RequestHealthCheck": {"Target": "ICMP", "Interval":5, "UnhealthyThreshold": 1}}]' \
  --network-interface='[{"NetworkId": "net-COMMON_GLOBAL","IsVipNetwork": true}]'
```

<details><summary>API レスポンス例</summary>

```
{
    "DNSName": ""
}
```

</details>


## Compute Instances

### SSH キーの準備

https://pfs.nifcloud.com/api/rest/CreateKeyPair.htm

```
nifcloud computing create-key-pair \
  --key-name=K8sTHWKey --password=theveryhardway --description=K8sTheHardWay \
  | jq -r '.KeyMaterial' | base64 -d > K8sTHWKey.pem
```

API が返すレスポンスは以下のようになっていて、 `KeyMaterial` には Base64 エンコードされた RSA 鍵が入っているので、それをデコードして使います。

```
{
    "Description": "K8sTheHardWay",
    "KeyFingerprint": "<truncated>",
    "KeyMaterial": "<truncated>",
    "KeyName": "K8sTHWKey",
    "RequestId": "<truncated>"
}
```

```sh
nifcloud computing create-key-pair \
  --key-name=K8sTHWKey --password=theveryhardway --description=K8sTheHardWay \
  --output text \
  --query 'KeyMaterial' | base64 -d > ~/.ssh/K8sTHWKey.pem
```

```sh
chmod 600 ~/.ssh/K8sTHWKey.pem
```

### Kubernetes Controllers

https://pfs.nifcloud.com/api/rest/RunInstances.htm

```sh
for i in 0 1 2; do
  nifcloud computing run-instances \
    --description=K8sTHWController \
    --no-disable-api-termination \
    --image-id=221 \
    --instance-id=controller${i} \
    --instance-type=e-medium4 \
    --key-name=K8sTHWKey \
    --network-interface='[{"NetworkId": "net-COMMON_GLOBAL"}, {"IpAddress": "10.240.0.1'${i}'", "NetworkName": "kubernetes"}]' \
    --placement=AvailabilityZone=east-11 \
    --security-group=K8sTHWExternal
done
```

<details><summary>API レスポンス例</summary>

```
{
    "GroupSet": [],
    "InstancesSet": [
        {
            "AccountingType": "2",
            "Architecture": "x86_64",
            "BlockDeviceMapping": [],
            "Description": "K8sTHWController",
            "DnsName": "",
            "ImageId": "Ubuntu Server 20.04 LTS",
            "InstanceId": "controller0",
            "InstanceState": {
                "Code": 0,
                "Name": "pending"
            },
            "InstanceType": "e-medium4",
            "InstanceUniqueId": "<truncated>",
            "IpAddress": "",
            "IpAddressV6": "",
            "IpType": "static",
            "IsoImage": [],
            "KeyName": "K8sTHWKey",
            "LaunchTime": "2021-12-08T00:00:00.000000+09:00",
            "Monitoring": {
                "State": "monitoring-disabled"
            },
            "NetworkInterfaceSet": [
                {
                    "Association": {
                        "IpOwnerId": "",
                        "PublicDnsName": ""
                    },
                    "Attachment": {
                        "AttachTime": null,
                        "AttachmentID": "",
                        "DeleteOnTermination": true,
                        "DeviceIndex": 0,
                        "Status": "attached"
                    },
                    "Description": "",
                    "GroupSet": [],
                    "NetworkInterfaceId": "",
                    "NiftyNetworkId": "net-COMMON_GLOBAL",
                    "OwnerId": "",
                    "PrivateDnsName": "",
                    "PrivateIpAddressesSet": [],
                    "SourceDestCheck": "",
                    "Status": "processing",
                    "SubnetId": "",
                    "VpcId": ""
                },
                {
                    "Attachment": {
                        "AttachTime": null,
                        "AttachmentID": "",
                        "DeleteOnTermination": true,
                        "DeviceIndex": 0,
                        "Status": "attached"
                    },
                    "Description": "",
                    "GroupSet": [],
                    "NetworkInterfaceId": "",
                    "NiftyNetworkId": "<truncated>",
                    "NiftyNetworkName": "kubernetes",
                    "OwnerId": "",
                    "PrivateDnsName": "",
                    "PrivateIpAddressesSet": [],
                    "SourceDestCheck": "",
                    "Status": "processing",
                    "SubnetId": "",
                    "VpcId": ""
                }
            ],
            "NiftyPrivateIpType": "static",
            "Placement": {
                "AvailabilityZone": "east-11"
            },
            "Platform": "ubuntu",
            "PrivateDnsName": "10.240.0.10",
            "PrivateIpAddress": "10.240.0.10",
            "PrivateIpAddressV6": "",
            "RootDeviceType": "disk"
        }
    ],
    "OwnerId": "",
    "RequestId": "<truncated>",
    "ReservationId": ""
}
```

</details>

#### マルチLBに追加

Controller のインスタンスが起動するまで待ちます（先に Worker のインスタンスを立ち上げていても大丈夫です）。

```sh
nifcloud computing nifty-register-instances-with-elastic-load-balancer \
  --elastic-load-balancer-name=K8sTHWLB \
  --protocol=TCP \
  --elastic-load-balancer-port=6443 \
  --instance-port=6443 \
  --instances='[{"InstanceId": "controller0"}, {"InstanceId": "controller1"}, {"InstanceId": "controller2"}]'
```

<details><summary>API レスポンス例</summary>

```
{
    "NiftyRegisterInstancesWithElasticLoadBalancerResult": ""
}
```

</details>

#### マルチLBのIPアドレスをインスタンスに送る

第8章で使う、マルチロードバランサ―の IP アドレスを各インスタンスに送ります。

```sh
echo export KUBERNETES_PUBLIC_ADDRESS=$(nifcloud computing nifty-describe-elastic-load-balancers \
  --elastic-load-balancers=ListOfRequestElasticLoadBalancerName=K8sTHWLB \
  --output text \
  --query 'NiftyDescribeElasticLoadBalancersResult.ElasticLoadBalancerDescriptions[0].NetworkInterfaces[0].IpAddress') > KUBERNETES_PUBLIC_ADDRESS
```

SSH キーのパスフレーズは `theveryhardway` です。

```sh
for instance in controller0 controller1 controller2; do
  scp KUBERNETES_PUBLIC_ADDRESS ${instance}:~/
done
```


### Kubernetes Workers

```sh
for i in 0 1 2; do
  nifcloud computing run-instances \
    --description=K8sTHWWorker \
    --no-disable-api-termination \
    --image-id=221 \
    --instance-id=worker${i} \
    --instance-type=e-medium4 \
    --key-name=K8sTHWKey \
    --network-interface='[{"NetworkId": "net-COMMON_GLOBAL"}, {"IpAddress": "10.240.0.2'${i}'", "NetworkName": "kubernetes"}]' \
    --placement=AvailabilityZone=east-11 \
    --security-group=K8sTHWInternal
done
```

<details><summary>API レスポンス例</summary>

```
{
    "GroupSet": [],
    "InstancesSet": [
        {
            "AccountingType": "2",
            "Architecture": "x86_64",
            "BlockDeviceMapping": [],
            "Description": "K8sTHWWorker",
            "DnsName": "",
            "ImageId": "Ubuntu Server 20.04 LTS",
            "InstanceId": "worker0",
            "InstanceState": {
                "Code": 0,
                "Name": "pending"
            },
            "InstanceType": "e-medium4",
            "InstanceUniqueId": "<truncated>",
            "IpAddress": "",
            "IpAddressV6": "",
            "IpType": "static",
            "IsoImage": [],
            "KeyName": "K8sTHWKey",
            "LaunchTime": "2021-12-08T00:00:00.000000+09:00",
            "Monitoring": {
                "State": "monitoring-disabled"
            },
            "NetworkInterfaceSet": [
                {
                    "Association": {
                        "IpOwnerId": "",
                        "PublicDnsName": ""
                    },
                    "Attachment": {
                        "AttachTime": null,
                        "AttachmentID": "",
                        "DeleteOnTermination": true,
                        "DeviceIndex": 0,
                        "Status": "attached"
                    },
                    "Description": "",
                    "GroupSet": [],
                    "NetworkInterfaceId": "",
                    "NiftyNetworkId": "net-COMMON_GLOBAL",
                    "OwnerId": "",
                    "PrivateDnsName": "",
                    "PrivateIpAddressesSet": [],
                    "SourceDestCheck": "",
                    "Status": "processing",
                    "SubnetId": "",
                    "VpcId": ""
                },
                {
                    "Attachment": {
                        "AttachTime": null,
                        "AttachmentID": "",
                        "DeleteOnTermination": true,
                        "DeviceIndex": 0,
                        "Status": "attached"
                    },
                    "Description": "",
                    "GroupSet": [],
                    "NetworkInterfaceId": "",
                    "NiftyNetworkId": "<truncated>",
                    "NiftyNetworkName": "kubernetes",
                    "OwnerId": "",
                    "PrivateDnsName": "",
                    "PrivateIpAddressesSet": [],
                    "SourceDestCheck": "",
                    "Status": "processing",
                    "SubnetId": "",
                    "VpcId": ""
                }
            ],
            "NiftyPrivateIpType": "static",
            "Placement": {
                "AvailabilityZone": "east-11"
            },
            "Platform": "ubuntu",
            "PrivateDnsName": "10.240.0.20",
            "PrivateIpAddress": "10.240.0.20",
            "PrivateIpAddressV6": "",
            "RootDeviceType": "disk"
        }
    ],
    "OwnerId": "",
    "RequestId": "<truncated>",
    "ReservationId": ""
}
```

</details>

本家ではインスタンス作成時に `--metadata pod-cidr=10.200.${i}.0/24` を指定していますが、ニフクラには該当する機能がないので代わりに SSH で送ります。OS 以上のレイヤーにあまり介入しないニフクラと、そうでない Google Cloud の違いであると思います。

SSH キーのパスフレーズ: `theveryhardway`

```sh
for i in 0 1 2; do
  echo export POD_CIDR=10.200.${i}.0/24 > POD_CIDR-worker${i}
  scp POD_CIDR-worker${i} worker${i}:~/POD_CIDR
done
```

### SSH config 設定

全てのインスタンスが起動してから実行してください。

```sh
for instance in controller0 controller1 controller2 worker0 worker1 worker2; do
  INSTANCE_IP=$(nifcloud computing describe-instance-attribute \
    --instance-id=${instance} \
    --attribute=networkInterfaceSet \
    --output text \
    --query 'NetworkInterfaceSet[0].Association.PublicIp')
  cat <<EOF >> ~/.ssh/config
Host ${instance}
    HostName ${INSTANCE_IP}
    User root
    Port 22
    IdentityFile ~/.ssh/K8sTHWKey.pem

EOF
done
```

次: [認証局のプロビジョニングと TLS 証明書の生成](04-certificate-authority.md)

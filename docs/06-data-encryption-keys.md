# データ暗号化の設定ファイルと鍵の生成

[本家 Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/06-data-encryption-keys.md)

## The Encryption Key

```sh
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## The Encryption Config File

```sh
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

SSH キーのパスフレーズ: `theveryhardway`

```sh
for instance in controller0 controller1 controller2; do
  scp encryption-config.yaml ${instance}:~/
done
```

次: [etcd クラスターの起動](07-bootstrapping-etcd.md)

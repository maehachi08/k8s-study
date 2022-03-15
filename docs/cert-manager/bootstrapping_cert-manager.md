# bootstrapping cert-manager

## 参考

- https://github.com/cert-manager
- https://cert-manager.io
- https://aws.amazon.com/jp/about-aws/whats-new/2022/01/acm-kubernetes-cert-manager-plugin-production/
- https://blog.manabusakai.com/2021/10/custom-resources-for-cert-manager/

## cert-manager について

- TLS証明書や発行者(Let's Encrypt など)をkubernetes resourcesとして表現できるkubernetes addons
    - 証明書が有効で最新であることを確認し、有効期限が切れる前に設定された時間で証明書の更新を試みます
    - 発行者(issuers)は複数のProviderをサポート
        ![](https://cert-manager.io/images/high-level-overview.svg)

### 


## 構築手順

### インストール

- https://cert-manager.io/docs/installation/helm/
- https://cert-manager.io/docs/usage/kubectl-plugin/

### 証明書の発行者として `Let's Encrypt` を登録する

- https://cert-manager.io/docs/configuration/acme/
- https://letsencrypt.org/ja/docs/challenge-types/

### 証明書の取得

- https://cert-manager.io/docs/concepts/certificate/



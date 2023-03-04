# cert-manager

## 参考

- https://cert-manager.io
- https://github.com/cert-manager
- https://aws.amazon.com/jp/about-aws/whats-new/2022/01/acm-kubernetes-cert-manager-plugin-production/
- https://blog.manabusakai.com/2021/10/custom-resources-for-cert-manager/
- https://zenn.dev/masaaania/articles/e54119948bbaa2

## About

- TLS証明書や発行者(Let's Encrypt など)をkubernetes resourcesとして表現できるkubernetes addons
    - 証明書が有効で最新であることを確認し、有効期限が切れる前に設定された時間で証明書の更新を試みます
    - 発行者(issuers)は複数のProviderをサポート
        ![](https://cert-manager.io/images/high-level-overview.svg)

## ACME(Automated Certificate Management Environment)

- https://cert-manager.io/docs/configuration/acme/
- `ACME` とはX.509証明書のドメイン検証、インストール、および管理を自動化するための標準プロトコル
    - X.509証明書を取得するためにはCSRを認証局に送信するなど様々な手続きがあり、更新作業含めて煩雑です。
      `ACME` はそれらの手続きを自動化したものです。
    - `cert-manager` は`ACME` に対応したIssuer(Let's Encrypt など) を指定することで、
      Userは `Certificate` resourceを定義することで `cert-manager` がX.509証明書を自動的に取得します。(後述するChallengesは行う必要はある)

### Challenges

- https://cert-manager.io/docs/configuration/acme/#solving-challenges
- `Challenges` とは証明書を要求したクライアント(user)が当該ドメインの所有者であることを確認するための仕組みです
    - cert-managerは、`HTTP01` および `DNS01` challengesという検証方法をサポートします (refs: [Which ACME Challenge Type Should I Use? HTTP-01 or DNS-01?](https://www.ssl.com/faqs/which-acme-challenge-type-should-i-use-http-01-or-dns-01/))
        - https://cert-manager.io/docs/configuration/acme/http01/
            - IssuerがTokenを発行し、クライアントが発行されたTokenや鍵のfingerprintなど必要な情報を記載したファイルを所有ドメインの `http://<YOUR_DOMAIN>/.well-known/acme-challenge/<TOKEN>` に配置することでIssuerは証明書を要求したクライアントとドメインの所有者が同一であることを検証します
            - クライアントが所有するドメインのWebサーバ上にファイルを配置し、かつ `80/HTTP` でアクセス可能である必要があります
        - https://cert-manager.io/docs/configuration/acme/dns01/
            - IssuerがTokenを発行し、クライアントが発行されたTokenや鍵のfingerprintなど必要な情報を記載したTXTレコード `_acme-challenge.<YOUR_DOMAIN>` を作成することでIssuerは証明書を要求したクライアントとドメインの所有者が同一であることを検証します
            - `HTTP01` とは異なりWebサーバへの `80/HTTP` アクセスは不要
            - ドメインを管理するDNSサーバでTXTレコードの管理をサポートしている必要があります
            - wildcard証明書を作成することが可能です



### 証明書の発行者として `Let's Encrypt` を登録する

- https://cert-manager.io/docs/configuration/acme/
- https://letsencrypt.org/ja/docs/challenge-types/

### 証明書の取得

- https://cert-manager.io/docs/concepts/certificate/


### 
- https://cert-manager.io/docs/configuration/acme/dns01/route53/


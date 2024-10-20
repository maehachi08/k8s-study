# bootstrapping external-dns

## 参考情報

- https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/integrations/external_dns/

### kubernetes 1.31 対応状況について

- https://github.com/kubernetes-sigs/external-dns/blob/master/README.md#kubernetes-version-compatibility
    - [v0.10.0](https://github.com/kubernetes-sigs/external-dns/releases/tag/v0.10.0) で対応済み


### オンプレでAWS Route53向けexternal-dns controllerの起動方法について

- https://stackoverflow.com/questions/60267737/is-it-possible-to-use-aws-route-53-as-a-dns-provider-for-a-bare-metal-k8s-cluste
- https://github.com/kubernetes-sigs/external-dns/issues/539

### オンプレなど自宅環境のWAN IPアドレスをDNS Providerに通知したい

- https://github.com/kubernetes-sigs/external-dns/issues/1394

### オンプレなど自宅環境におけるPublic IPを払い出しつつroute53へレコード登録する

- k8s cluster内部のIPアドレスではなく、worker nodeのinterface(wlan0)に設定しているIPアドレス(便宜的にpublic ipと仮定)と同じレンジでIPアドレスを払い出すような構成
    - これは将来用
    - [metallb](https://metallb.universe.tf/) でk8s clusterの外側にLoadBalancerを作成しPublic IPアドレスを割り当てる
    - https://blog.web-apps.tech/type-loadbalancer_by_metallb/

## 構築手順

### 1. AWS IAM Policy作成

1. `external-dns-controller-policy-document.json` を作成

    <details><summary>external-dns-controller-policy-document.json</summary>

        ```
        sudo tee external-dns-controller-policy-document.json << EOF > /dev/null
        {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Effect": "Allow",
              "Action": [
                "route53:ChangeResourceRecordSets"
              ],
              "Resource": [
                "arn:aws:route53:::hostedzone/*"
              ]
            },
            {
              "Effect": "Allow",
              "Action": [
                "route53:ListHostedZones",
                "route53:ListResourceRecordSets"
              ],
              "Resource": [
                "*"
              ]
            }
          ]
        }
        EOF
        ```

1. policyを作成
    ```
    aws iam create-policy --policy-name k8s-external-dns-policy  --policy-document file://external-dns-controller-policy-document.json
    ```

### 2. AWS IAM User作成

1. userを作成
    ```
    aws iam create-user --user-name k8s-external-dns
    ```

1. 作成したIAM Policyをアタッチする
    ```
    aws iam attach-user-policy --user-name k8s-external-dns --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/k8s-external-dns-policy
    ```

1. 作成したIAM Userのcredentialを確認する(Deploymentsマニフェストで環境変数としてセットする)
    - `AWS_ACCESS_KEY_ID`
    - `AWS_SECRET_ACCESS_KEY`


### 3. external-dns controllerをデプロイ

- point
    - https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md をベースにbare-metal向けに修正
    - namespaceは `kube-system`
    - aws credentialは `k8s-external-dns` iam userのもの
    - hosted_zone_id はexternal-dnsでレコード登録させたいRoute53 zone

1. manifestsに代入する変数を定義

    | variable name | description |
    |:---|:---|
    | `DOMAIN`                | external-dnsで登録したいゾーンのドメイン |
    | `HOSTED_ZONE_ID`        | external-dnsで登録したいゾーンのHosted Zone ID |
    | `AWS_ACCESS_KEY_ID`     | external-dnsでroute53へのレコード登録に使用するAWS IAM Userのcredential情報 |
    | `AWS_SECRET_ACCESS_KEY` | external-dnsでroute53へのレコード登録に使用するAWS IAM Userのcredential情報 |
    | `AWS_DEFAULT_REGION`    | external-dnsでroute53へのレコード登録に使用するAWS IAM Userのcredential情報 |

      ```
      DOMAIN="XXXXXXX.com"
      HOSTED_ZONE_ID="XXXXXXX"
      AWS_ACCESS_KEY_ID="XXXXXXX"
      AWS_SECRET_ACCESS_KEY="XXXXXXX"
      AWS_DEFAULT_REGION="XXXXXXX"
      ```

1. manifestsファイル作成
    <details><summary>/etc/kubernetes/manifests/external-dns.yaml</summary>

        ```
        AWS_DEFAULT_REGION="ap-north-east-1"

        sudo tee /etc/kubernetes/manifests/external-dns.yaml <<  EOF > /dev/null
        ---
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: external-dns
          namespace: kube-system
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRole
        metadata:
          name: external-dns
        rules:
        - apiGroups: [""]
          resources: ["services","endpoints","pods"]
          verbs: ["get","watch","list"]
        - apiGroups: ["extensions","networking.k8s.io"]
          resources: ["ingresses"]
          verbs: ["get","watch","list"]
        - apiGroups: [""]
          resources: ["nodes"]
          verbs: ["list","watch"]
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: external-dns-viewer
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: external-dns
        subjects:
        - kind: ServiceAccount
          name: external-dns
          namespace: kube-system
        ---
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: external-dns
          namespace: kube-system
        spec:
          strategy:
            type: Recreate
          selector:
            matchLabels:
              app: external-dns
          template:
            metadata:
              labels:
                app: external-dns
            spec:
              serviceAccountName: external-dns
              containers:
              - name: external-dns
                image: k8s.gcr.io/external-dns/external-dns:v0.10.0
                env:
                - name: AWS_ACCESS_KEY_ID
                  value: <k8s-external-dns AWSアカウントのAWS_ACCESS_KEY_ID>
                - name: AWS_SECRET_ACCESS_KEY
                  value:  <k8s-external-dns AWSアカウントのAWS_SECRET_ACCESS_KEY>
                - name: AWS_DEFAULT_REGION
                  value: "${AWS_DEFAULT_REGION}"
                args:
                - --source=service
                - --source=ingress
                - --domain-filter=${DOMAIN} # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
                - --provider=aws
                - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
                - --aws-zone-type=public # only look at public hosted zones (valid values are public, private or no value for both)
                - --registry=txt
                - --txt-owner-id=<HOSTED_ZONE_ID>
                - --txt-prefix=prefix_
                - --log-level=debug
              securityContext:
                fsGroup: 65534 # For ExternalDNS to be able to read Kubernetes and AWS token files
        EOF
        ```

    </details>

1. デプロイ
    ```
    kubectl apply -f /etc/kubernetes/manifests/external-dns.yaml
    ```

### 動作確認

- external-dns コンテナログ
    - デプロイ済みingressのhostnameでroute53へのレコード登録を確認
        ```
        time="2021-09-26T04:59:24Z" level=info msg="Instantiating new Kubernetes client"
        time="2021-09-26T04:59:24Z" level=debug msg="apiServerURL: "
        time="2021-09-26T04:59:24Z" level=debug msg="kubeConfig: "
        time="2021-09-26T04:59:24Z" level=info msg="Using inCluster-config based on serviceaccount-token"
        time="2021-09-26T04:59:24Z" level=info msg="Created Kubernetes client https://10.32.0.1:443"
        time="2021-09-26T04:59:30Z" level=debug msg="Refreshing zones list cache"
        time="2021-09-26T04:59:31Z" level=debug msg="Considering zone: /hostedzone/<HOSTED_ZONE_ID> (domain: example.com.)"
        time="2021-09-26T04:59:31Z" level=debug msg="No endpoints could be generated from service kube-system/kube-dns"
        time="2021-09-26T04:59:31Z" level=debug msg="No endpoints could be generated from service kube-system/metrics-server"
        time="2021-09-26T04:59:31Z" level=debug msg="No endpoints could be generated from service default/kubernetes"
        time="2021-09-26T04:59:31Z" level=debug msg="No endpoints could be generated from service default/nginx-service"
        time="2021-09-26T04:59:31Z" level=debug msg="No endpoints could be generated from service ingress-nginx/ingress-nginx-controller"
        time="2021-09-26T04:59:31Z" level=debug msg="No endpoints could be generated from service ingress-nginx/ingress-nginx-controller-admission"
        time="2021-09-26T04:59:31Z" level=debug msg="Endpoints generated from ingress: default/nginx-test-ingress: [dev1.example.com 0 IN A  192.168.10.51 []]"
        time="2021-09-26T04:59:31Z" level=debug msg="Refreshing zones list cache"
        time="2021-09-26T04:59:31Z" level=debug msg="Considering zone: /hostedzone/<HOSTED_ZONE_ID> (domain: example.com.)"
        time="2021-09-26T04:59:31Z" level=info msg="Applying provider record filter for domains: [example.com. .example.com.]"
        time="2021-09-26T04:59:31Z" level=debug msg="Refreshing zones list cache"
        time="2021-09-26T04:59:31Z" level=debug msg="Considering zone: /hostedzone/<HOSTED_ZONE_ID> (domain: example.com.)"
        time="2021-09-26T04:59:31Z" level=debug msg="Adding dev1.example.com. to zone example.com. [Id: /hostedzone/<HOSTED_ZONE_ID>]"
        time="2021-09-26T04:59:31Z" level=debug msg="Adding dev1.example.com. to zone example.com. [Id: /hostedzone/<HOSTED_ZONE_ID>]"
        time="2021-09-26T04:59:31Z" level=info msg="Desired change: CREATE dev1.example.com A [Id: /hostedzone/<HOSTED_ZONE_ID>]"
        time="2021-09-26T04:59:31Z" level=info msg="Desired change: CREATE dev1.example.com TXT [Id: /hostedzone/<HOSTED_ZONE_ID>]"
        time="2021-09-26T04:59:32Z" level=info msg="2 record(s) in zone example.com. [Id: /hostedzone/<HOSTED_ZONE_ID>] were successfully updated"
        ```

- route53 record set
    - 対象のhosted zoneにingressのhostで指定したhostnameでレコードが作成されていることを確認
        ```
        $ aws route53 list-resource-record-sets --output json --hosted-zone-id <HOSTED_ZONE_ID> | jq '.ResourceRecordSets | map(select(.Name == "dev1.example.com."))'
        [
          {
            "Name": "dev1.example.com.",
            "Type": "A",
            "TTL": 300,
            "ResourceRecords": [
              {
                "Value": "192.168.10.51"
              }
            ]
          },
          {
            "Name": "dev1.example.com.",
            "Type": "TXT",
            "TTL": 300,
            "ResourceRecords": [
              {
                "Value": "\"heritage=external-dns,external-dns/owner=<HOSTED_ZONE_ID>,external-dns/resource=ingress/default/nginx-test-ingress\""
              }
            ]
          }
        ]
        ```

- 登録されたAレコードのhostnameが名前解決できることを確認
    - k8s serviceリソースから見るとnode portに対するnode addressは自宅環境のWifiルータで払い出すレンジ(192.168.10.0/24)なので想定通り
        ```
        $ dig +noall +answer dev1.example.com
        dev1.example.com.      283     IN      A       192.168.10.51
        ```

## Appendix

### Ingressリソースで使用可能なannotations

- https://github.com/kubernetes-sigs/external-dns/blob/v0.10.0/source/source.go#L40-L68

    | annotations | describe |
    |:---|:---|
    | `external-dns.alpha.kubernetes.io/controller` | 複数のDNS Controllerがデプロイされている場合にどのDNS Controllerが責任を負っているのかを把握するために指定する |
    | `external-dns.alpha.kubernetes.io/hostname`   | 使用するホスト名を指定する<br>Serviceリソースの場合はこのannotationsで指定する<br>Ingressリソースの場合はこのannotationsかruleのhost attrで指定する |
    | `external-dns.alpha.kubernetes.io/access`     | パブリックインターフェイスアドレスとプライベートインターフェイスアドレスのどちらを使用するかを指定する |
    | `external-dns.alpha.kubernetes.io/target`     | CNAME recordを作成する場合にCNAME recordのvalueとなる値を指定する |
    | `external-dns.alpha.kubernetes.io/ttl`        | DNS recordのTTLを指定する(default: 300) |
    | `external-dns.alpha.kubernetes.io/alias`      | `true` でALIAS recordを作成する |
    | `external-dns.alpha.kubernetes.io/ingress-hostname-source` | Ingressリソースの場合にhostnameの指定方法をannotationsかruleのhost attrかを限定できる |
    | `external-dns.alpha.kubernetes.io/internal-hostname` | Target IPアドレスをCluster IPアドレスとする場合に指定する |
    | `external-dns.alpha.kubernetes.io/set-identifier` | AWS Route53においてDNS NameとTypeが同じ場合に重み付けなどによるルーティングポリシーを定義する際の識別子<br>Record IDとなる値 |

### external-dnsでAレコードではなくCNAMEレコードを作成する

1. controllerの起動引数に `--txt-prefix=<prefix文字列>` を追加
    - 指定した文字列がTXT recordのレコード名(Aレコードの場合はAレコードと同名)のprefixとして追加されます
    - CNAMEレコードは(TXTレコードであっても)他のレコードと共存できない仕様です([RFC 1034セクション3.6.2](https://datatracker.ietf.org/doc/html/rfc1034#section-3.6.2)、[RFC 1912セクション2.4](https://datatracker.ietf.org/doc/html/rfc1912#section-2.4)
    - https://github.com/kubernetes-sigs/external-dns/blob/master/docs/faq.md#im-using-an-elb-with-txt-registry-but-the-cname-record-clashes-with-the-txt-record-how-to-avoid-this

1. Ingressリソースのannotationsを以下のように設定する
    - `external-dns.alpha.kubernetes.io/hostname` にCNAMEレコードのFQDNを指定
    - `external-dns.alpha.kubernetes.io/target` にCNAMEレコードのValueとなる参照先のFQDNまたはIPアドレスなど指定

        ```
        external-dns.alpha.kubernetes.io/hostname: dev1.example.com
        external-dns.alpha.kubernetes.io/target: alias1.example.com
        ```

1. `--txt-prefix=prefix_` で動作確認
    1. Ingressリソース
        <details>

        ```
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          name: nginx-test-ingress
          annotations:
            external-dns.alpha.kubernetes.io/hostname: dev1.example.com
            external-dns.alpha.kubernetes.io/target: alias1.example.com
        spec:
          ingressClassName: nginx
          defaultBackend:
            service:
              name: nginx-service
              port:
                number: 8080
          rules:
            - host: dev1.example.com
              http:
                paths:
                  - path: /
                    pathType: Prefix
                    backend:
                      service:
                        name: nginx-service
                        port:
                          number: 8080
        ```

        </details>

    1. external-dns log
        <details>

        ```
        time="2021-10-01T09:05:18Z" level=debug msg="Endpoints generated from ingress: default/nginx-test-ingress: [dev1.example.com 0 IN CNAME  alias1.example.com [] dev1.example.com 0 IN CNAME  alias1.example.com []]"
        time="2021-10-01T09:05:18Z" level=debug msg="Removing duplicate endpoint dev1.example.com 0 IN CNAME  alias1.example.com []"
        time="2021-10-01T09:05:18Z" level=debug msg="Refreshing zones list cache"
        time="2021-10-01T09:05:18Z" level=debug msg="Considering zone: /hostedzone/<HOSTED_ZONE_ID> (domain: example.com.)"
        time="2021-10-01T09:05:18Z" level=info msg="Applying provider record filter for domains: [example.com. .example.com.]"
        time="2021-10-01T09:05:18Z" level=debug msg="Refreshing zones list cache"
        time="2021-10-01T09:05:18Z" level=debug msg="Considering zone: /hostedzone/<HOSTED_ZONE_ID> (domain: example.com.)"
        time="2021-10-01T09:05:18Z" level=debug msg="Adding dev1.example.com. to zone example.com. [Id: /hostedzone/<HOSTED_ZONE_ID>]"
        time="2021-10-01T09:05:18Z" level=debug msg="Adding prefix_dev1.example.com. to zone example.com. [Id: /hostedzone/<HOSTED_ZONE_ID>]"
        time="2021-10-01T09:05:18Z" level=info msg="Desired change: CREATE dev1.example.com CNAME [Id: /hostedzone/<HOSTED_ZONE_ID>]"
        time="2021-10-01T09:05:18Z" level=info msg="Desired change: CREATE prefix_dev1.example.com TXT [Id: /hostedzone/<HOSTED_ZONE_ID>]"
        time="2021-10-01T09:05:19Z" level=info msg="2 record(s) in zone example.com. [Id: /hostedzone/<HOSTED_ZONE_ID>] were successfully updated"
        ```

        </details>

    1. route53 レコード確認
        <details>

        ```
        $ aws route53 list-resource-record-sets --output json --hosted-zone-id <HOSTED_ZONE_ID> | jq '.ResourceRecordSets | map(select(.Name == "dev1.example.com." or .Name == "prefix_dev1.example.com."))'
        [
          {
            "Name": "prefix_dev1.example.com.",
            "Type": "TXT",
            "TTL": 300,
            "ResourceRecords": [
              {
                "Value": "\"heritage=external-dns,external-dns/owner=<HOSTED_ZONE_ID>,external-dns/resource=ingress/default/nginx-test-ingress\""
              }
            ]
          },
          {
            "Name": "dev1.example.com.",
            "Type": "CNAME",
            "TTL": 300,
            "ResourceRecords": [
              {
                "Value": "alias1.example.com"
              }
            ]
          }
        ]

        $ dig +noall +answer dev1.example.com
        dev1.example.com.      300     IN      CNAME   alias1.example.com.
        ```

        </details>

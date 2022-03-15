# metrics-server

`metrics-server`とはkubeletやkube-apiserverから各種リソースのメトリックスを収集します。
収集したメトリックス情報はautoscaling([HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) や [VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler/))を行うために使用されます。

## 参考

- metrics-server
    - https://github.com/kubernetes-sigs/metrics-server
        - [docs/command-line-flags.txt](https://github.com/kubernetes-sigs/metrics-server/blob/v0.6.1/docs/command-line-flags.txt)
    - https://kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/
- kube-apiserver Aggregation Layer設定について
    - https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/
    - https://kubernetes.io/ja/docs/concepts/cluster-administration/proxies/
    - https://github.com/ansilh/kubernetes-the-hardway-virtualbox/blob/master/15.Deploy-Metric-Server.md

## 要件

- https://github.com/kubernetes-sigs/metrics-server#requirements
    - Metrics Server must be reachable from kube-apiserver by container IP address (or node IP if hostNetwork is enabled).
    - The kube-apiserver must enable an aggregation layer.
    - Nodes must have Webhook authentication and authorization enabled.
    - Kubelet certificate needs to be signed by cluster Certificate Authority (or disable certificate validation by passing --kubelet-insecure-tls to Metrics Server)
    - Container runtime must implement a container metrics RPCs (or have cAdvisor support)

## 構築手順

### kube-apiserver Aggregation Layer設定

!!! info
    https://kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/#metrics-server
    > Metrics Server collects metrics from the Summary API, exposed by Kubelet on each node, and is registered with the main API server via Kubernetes aggregator.

- kube-apiserverでAggregation Layerを有効にする
    - Aggregation Layerが有効でない場合はmetrics-serverで以下のようなエラーログが出ている
        ```
        E0217 15:13:53.378655       1 webhook.go:224] Failed to make webhook authorizer request: Post "https://10.32.0.1:443/apis/authorization.k8s.io/v1/subjectaccessreviews?timeout=10s": cont
        ext canceled
        E0217 15:13:53.378917       1 errors.go:77] Post "https://10.32.0.1:443/apis/authorization.k8s.io/v1/subjectaccessreviews?timeout=10s": context canceled
        E0217 15:13:53.379124       1 timeout.go:137] post-timeout activity - time-elapsed: 121.389µs, GET "/apis/metrics.k8s.io/v1beta1" result: <nil>
        ```
    - [kube-apiserver front-proxy(for aggregation layer)のサーバー証明書](/setup/04_creation_certificate/#kube-apiserver-front-proxyfor-aggregation-layer)
        - front-proxy用CA証明書およびサーバ証明書と秘密鍵を生成する
            - `/var/lib/kubernetes/front-proxy-ca.pem`
            - `/var/lib/kubernetes/front-proxy.pem`
            - `/var/lib/kubernetes/front-proxy-key.pem`
    - [/setup/06_master/03_bootstrapping_kube-apiserver/](/setup/06_master/03_bootstrapping_kube-apiserver/)
        - kube-apiserverの起動オプションを追加
            ```
            --enable-aggregator-routing=true
            --requestheader-client-ca-file=/var/lib/kubernetes/front-proxy-ca.pem
            --requestheader-allowed-names=front-proxy-ca
            --requestheader-extra-headers-prefix=X-Remote-Extra
            --requestheader-group-headers=X-Remote-Group
            --requestheader-username-headers=X-Remote-User
            --proxy-client-cert-file=/var/lib/kubernetes/front-proxy.pem
            --proxy-client-key-file=/var/lib/kubernetes/front-proxy-key.pem
            ```

### metics-serverインストール

1. manifestsをdeploy

    ```
    $ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    serviceaccount/metrics-server created
    clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
    clusterrole.rbac.authorization.k8s.io/system:metrics-server created
    rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
    clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
    clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
    service/metrics-server created
    deployment.apps/metrics-server created
    apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
    ```

1. metrics-server が起動はするがTLS関連のエラーで正常に動作しない
    - `kubectl logs` コマンド
        ```
        Failed to scrape node" err="Get \"https://192.168.10.51:10250/metrics/resource\": x509: certificate is valid for 192.168.10.50, not 192.168.10.51" node="k8s-node1
        ```

1. TLS認証設定の変更
    - metrics-serverの起動引数に `--kubelet-insecure-tls` を追加
        - https://github.com/kubernetes-sigs/metrics-server/issues/131
        - https://github.com/kubernetes-sigs/metrics-server/issues/300
        - kubectl logsコマンドで以下エラーが出続けている場合の対処
        - TLS証明書の検証を行わないようにする(証明書の署名がmetrics-servergが想定するCAではないため)
            ```
            kubectl patch deploy metrics-server -n kube-system --patch "
            spec:
              template:
                spec:
                  containers:
                  - args:
                    - --cert-dir=/tmp
                    - --secure-port=4443
                    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
                    - --kubelet-use-node-status-port
                    - --metric-resolution=15s
                    - --kubelet-insecure-tls
                    name: metrics-server
            "
            ```

1. 起動したことを確認
    ```
    I0217 14:54:41.737667       1 serving.go:342] Generated self-signed cert (/tmp/apiserver.crt, /tmp/apiserver.key)
    I0217 14:54:42.981853       1 requestheader_controller.go:169] Starting RequestHeaderAuthRequestController
    I0217 14:54:42.981913       1 shared_informer.go:240] Waiting for caches to sync for RequestHeaderAuthRequestController
    I0217 14:54:42.981901       1 configmap_cafile_content.go:201] "Starting controller" name="client-ca::kube-system::extension-apiserver-authentication::client-ca-file"
    I0217 14:54:42.981970       1 shared_informer.go:240] Waiting for caches to sync for client-ca::kube-system::extension-apiserver-authentication::client-ca-file
    I0217 14:54:42.981993       1 configmap_cafile_content.go:201] "Starting controller" name="client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
    I0217 14:54:42.982036       1 shared_informer.go:240] Waiting for caches to sync for client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
    I0217 14:54:42.985611       1 secure_serving.go:266] Serving securely on [::]:4443
    I0217 14:54:42.985841       1 tlsconfig.go:240] "Starting DynamicServingCertificateController"
    W0217 14:54:42.986181       1 shared_informer.go:372] The sharedIndexInformer has started, run more than once is not allowed
    I0217 14:54:42.987234       1 dynamic_serving_content.go:131] "Starting controller" name="serving-cert::/tmp/apiserver.crt::/tmp/apiserver.key"
    I0217 14:54:43.082779       1 shared_informer.go:247] Caches are synced for client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
    I0217 14:54:43.082892       1 shared_informer.go:247] Caches are synced for RequestHeaderAuthRequestController
    I0217 14:54:43.082896       1 shared_informer.go:247] Caches are synced for client-ca::kube-system::extension-apiserver-authentication::client-ca-file
    ```

### 動作確認

#### API Path

- pods
    ```
    $ kubectl get --raw "/apis/metrics.k8s.io/v1beta1/pods" | jq '.items[] | select(.metadata.name == "coredns-675db8b7cc-hbzb2")'
    {
      "metadata": {
        "name": "coredns-675db8b7cc-hbzb2",
        "namespace": "kube-system",
        "creationTimestamp": "2022-02-17T16:00:21Z",
        "labels": {
          "k8s-app": "kube-dns",
          "pod-template-hash": "675db8b7cc"
        }
      },
      "timestamp": "2022-02-17T16:00:03Z",
      "window": "15.692s",
      "containers": [
        {
          "name": "coredns",
          "usage": {
            "cpu": "7990556n",
            "memory": "14064Ki"
          }
        }
      ]
    }
    ```
- nodes
    ```
    $ kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq .                                                                                              [22/47786]
    {
      "kind": "NodeMetricsList",
      "apiVersion": "metrics.k8s.io/v1beta1",
      "metadata": {},
      "items": [
        {
          "metadata": {
            "name": "k8s-master",
            "creationTimestamp": "2022-02-17T14:58:58Z",
            "labels": {
              "beta.kubernetes.io/arch": "arm64",
              "beta.kubernetes.io/os": "linux",
              "kubernetes.io/arch": "arm64",
              "kubernetes.io/hostname": "k8s-master",
              "kubernetes.io/os": "linux"
            }
          },
          "timestamp": "2022-02-17T14:58:49Z",
          "window": "10.198s",
          "usage": {
            "cpu": "273214048n",
            "memory": "1024976Ki"
          }
        },
        {
          "metadata": {
            "name": "k8s-node1",
            "creationTimestamp": "2022-02-17T14:58:58Z",
            "labels": {
              "beta.kubernetes.io/arch": "arm64",
              "beta.kubernetes.io/os": "linux",
              "kubernetes.io/arch": "arm64",
              "kubernetes.io/hostname": "k8s-node1",
              "kubernetes.io/os": "linux"
            }
          },
          "timestamp": "2022-02-17T14:58:51Z",
          "window": "10.094s",
          "usage": {
            "cpu": "141038629n",
            "memory": "548580Ki"
          }
        }
      ]
    }
    ```

#### `kubectl top pod`

```
$ kubectl top nodes
NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master   260m         7%     999Mi           80%
k8s-node1    130m         3%     527Mi           42%
```

#### `kubectl top pod`

```
$ kubectl top pods -A
NAMESPACE     NAME                                 CPU(cores)   MEMORY(bytes)
kube-system   coredns-675db8b7cc-hbzb2             7m           13Mi
kube-system   etcd-k8s-master                      31m          108Mi
kube-system   kube-apiserver-k8s-master            62m          220Mi
kube-system   kube-controller-manager-k8s-master   24m          72Mi
kube-system   kube-proxy-2kmcf                     1m           23Mi
kube-system   kube-proxy-fxcgv                     1m           12Mi
kube-system   kube-scheduler-k8s-master            3m           26Mi
kube-system   metrics-server-8bb87844c-v67lj       12m          15Mi
```


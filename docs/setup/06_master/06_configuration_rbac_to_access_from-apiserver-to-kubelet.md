# kube-apiserver から kubelet へのアクセス権を設定する

`kubectl` や他Client toolではkube-apiserverへリクエストを投げます。etcdに格納された情報であれば`kube-apiserver`から返すのですが、Podのmetricsやlogsなどは`kube-apiserver`から該当Podが起動しているNodeの `kubelet` のapi endpointにリクエストし返ってきた情報を`kubectl`に返します。

そのため、 **`kube-apiserver` から `kubelet` の必要なリソースへのアクセス権限を付与する必要があります。**

## Overview

`kubelet` はNode上の (Node metricsとNode上で起動するPodに関する)いくつかの情報を提供するためのHTTPS Endpointを提供しています。

提供するAPI Pathは (v.1.31.1)以下のとおりです。
https://github.com/kubernetes/kubernetes/blob/v1.31.1/pkg/kubelet/server/server.go#L95-L103


| Path | Description |
|:----|:----|
| `/metrics`           | Node自身のmetricsを提供します<br>https://kubernetes.io/docs/reference/instrumentation/metrics/<br>https://github.com/kubernetes/kubernetes/blob/v1.31.1/pkg/kubelet/metrics/metrics.go |
| `/metrics/cadvisor`  | Node上で起動しているコンテナのリソース使用率やパフォーマンス状況のmetricsを提供します<br>https://github.com/kubernetes/kubernetes/blob/v1.31.1/pkg/kubelet/cadvisor/cadvisor_linux.go#L36<br>https://github.com/google/cadvisor/blob/v0.50.0/docs/storage/prometheus.md |
| `/metrics/resource`  | Node, Pod, ContainerそれぞれのCPU使用率とメモリ使用率のmetricsを提供します。metrics-server v0.6.0以上はmetrics resource endpointとしてこのendpointを呼び出します。<br>https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/<br>https://github.com/kubernetes/kubernetes/blob/v1.31.1/pkg/kubelet/metrics/collectors/resource_metrics.go#L29-L113 |
| `/metrics/probes`    | Node上で起動しているPodのProbeに関するmetricsを提供します |
| `/stats/summary`     | Kubernetes Node, Node上で起動するPod及びコンテナのリソース消費やパフォーマンスに関するmetricsを提供します。metrics-server v0.6.0以上はsummary api endpointとしてこのendpointを呼び出します。<br>https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/<br>https://github.com/kubernetes/kubernetes/blob/v1.31.1/pkg/kubelet/server/stats/handler.go#L122<br>https://github.com/kubernetes/kubernetes/blob/v1.31.1/pkg/kubelet/server/stats/summary.go |
| `/logs/`             | Podやコンテナのログを提供します。Request PATHでPodやコンテナを指定します。 |
| `/checkpoint/`       | kubelet checkpointは実行中のコンテナのステートフルコピーを作成するための機能です。<br>https://kubernetes.io/docs/reference/node/kubelet-checkpoint-api/<br>https://kubernetes.io/blog/2022/12/05/forensic-container-checkpointing-alpha/ |
| `/debug/pprof/`      | [pprof](https://github.com/google/pprof) を使ったprofilingを行うためのエンドポイントを提供します |
| `/debug/flags/v`     | kubelet log level(verbosity)を取得および変更するためのエンドポイントを提供します |


`kubelet` がListenするportは `10250` です。
https://github.com/kubernetes/kubernetes/blob/v1.31.1/pkg/cluster/ports/ports.go#L29-L31


(RBACを使用している場合) kubeletの/metricsへのアクセスを許可したuser, groupもしくはClusterRoleのServiceAccountの認証情報が必要とのことです。

https://kubernetes.io/docs/concepts/cluster-administration/system-metrics/

> If your cluster uses RBAC, reading metrics requires authorization via a user, group or ServiceAccount with a ClusterRole that allows accessing /metrics.


### Node上からcurlでkubelet api endpontにアクセスしてみる

今回は管理者用のadmin.kubeconfigから `client-certificate-data` と `client-key-data` の内容をpemファイルに書き込んで使用します。

```
sudo apt install -y yq
cat admin.kubeconfig | yq -r .users[0].user.\"client-certificate-data\" | base64 -d > /tmp/cacert.pem
cat admin.kubeconfig | yq -r .users[0].user.\"client-key-data\" | base64 -d > /tmp/key.pem
```

#### /metrics

```
curl -k --cert /tmp/cacert.pem --key /tmp/key.pem https://k8s-master:10250/metrics
```

#### /logs

```
$ curl -k -s --cert /tmp/cacert.pem --key /tmp/key.pem https://k8s-master:10250/logs/containers/kube-apiserver-v1.31.1-k8s-master_kube-system_kube-apiserver-d51f341e57323df201b44f44046847573c9cfd9bc29623bdc5fd51a3b1a975b2.log  | tail -1
2024-10-14T19:30:13.432170309+09:00 stderr F I1014 10:30:13.431664       1 apf_controller.go:493] "Update CurrentCL" plName="exempt" seatDemandHighWatermark=1 seatDemandAvg=0.0023710613260329557 seatDemandStdev=0.04863578306371911 seatDemandSmoothed=0.07543643603118208 fairFrac=2.331974373907979 currentCL=1 concurrencyDenominator=1 backstop=false
```

- kubectl logsコマンドで同じログが取得できることを確認
    ```
    $ kubectl logs --tail=1 -n kube-system kube-apiserver-v1.31.1-k8s-master
    I1014 10:30:13.431664       1 apf_controller.go:493] "Update CurrentCL" plName="exempt" seatDemandHighWatermark=1 seatDemandAvg=0.0023710613260329557 seatDemandStdev=0.04863578306371911 seatDemandSmoothed=0.07543643603118208 fairFrac=2.331974373907979 currentCL=1 concurrencyDenominator=1 backstop=false
    ```

## 手順

1. ClusterRole `system:kube-apiserver-to-kubelet` を作成
    - `rbac.authorization.kubernetes.io/autoupdate` annotations
        - 起動するたびに、APIサーバーはデフォルトのClusterRoleを不足している権限で更新し、
          デフォルトのClusterRoleBindingを不足しているsubjectsで更新します。
          これにより、誤った変更をクラスタが修復できるようになり、
          新しいKubernetesリリースで権限とsubjectsが変更されても、
          RoleとRoleBindingを最新の状態に保つことができます。
    - `kubernetes.io/bootstrapping: rbac-defaults` labels
        - k8sの既定クラスタロールと既定ロールバインドであることを示す

    ```
    cat << EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: system:kube-apiserver-to-kubelet
      annotations:
        rbac.authorization.kubernetes.io/autoupdate: "true"
      labels:
        kubernetes.io/bootstrapping: rbac-defaults
    rules:
      - apiGroups:
          - ""
        resources:
          - nodes/proxy
          - nodes/stats
          - nodes/log
          - nodes/spec
          - nodes/metrics
        verbs:
          - "*"
    EOF
    ```

1. `Kubernetes` ユーザへ`system:kube-apiserver-to-kubelet` ClusterRoleを紐付ける
    - `roleRef` で紐付けたRoleを指定する
    - `subjects` でRoleを紐付けるAccountを指定する
      ```
      cat << EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
      ---
      apiVersion: rbac.authorization.k8s.io/v1
      kind: ClusterRoleBinding
      metadata:
        name: system:kube-apiserver
        namespace: ""
      roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: system:kube-apiserver-to-kubelet
      subjects:
        - apiGroup: rbac.authorization.k8s.io
          kind: User
          name: Kubernetes
      EOF
      ```

        - この紐付けが正しくない、もしくは未設定の場合、token付きでkubectlを利用した場合に以下エラーとなる
          ```
          Error from server (Forbidden): Forbidden (user=Kubernetes, verb=get, resource=nodes, subresource=proxy) ( pods/log kube-proxy)
          ```

## 参考資料

- https://kubernetes.io/ja/docs/reference/access-authn-authz/rbac/
- https://qiita.com/sheepland/items/67a5bb9b19d8686f389d


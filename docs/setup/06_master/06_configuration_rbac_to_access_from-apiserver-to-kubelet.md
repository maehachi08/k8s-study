# kube-apiserver から kubelet へのアクセス権を設定する

`kubectl` や他Client toolではkube-apiserverへリクエストを投げます。`kube-apiserver` ではetcdに格納された情報を基に各worker node(の `kubelet`) とやりとりする必要があります。(例えばexec,top,logsなど)
`kube-apiserver` から `kubelet` の必要なリソースへのアクセス権限を付与する必要があります。

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


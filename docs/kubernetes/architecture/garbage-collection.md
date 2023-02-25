### 参考

- https://kubernetes.io/ja/docs/concepts/workloads/controllers/garbage-collection/
- https://kubernetes.io/docs/concepts/architecture/garbage-collection/
- https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion
- https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/
- https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/
- https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/
- https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/object-meta


### About Garbage Collection

- KubernetesがCluster Resourcesのcleanupを行う仕組み
    - cleanup対象となるResourcesは以下のようなもの
        1. 終了したPod
        1. 完了したJob
        1. owner referenceのないオブジェクト
        1. 未使用のコンテナとコンテナイメージ
        1. StorageClassの再利用ポリシーがDeleteである動的にプロビジョニングされたPersistentVolume
        1. 失効または期限切れのCertificateSigningRequests (CSRs)
        1. 次のシナリオで削除されたNode:
        1. クラウド上でクラスターがクラウドコントローラーマネージャーを使用する場合
        1. オンプレミスでクラスターがクラウドコントローラーマネージャーと同様のアドオンを使用する場合
        1. Node Leaseオブジェクト

### Owner References

Kubernetesでは依存関係にあるResourceが存在します。
例えば、[`ReplicaSet`](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) の定義に基づいてPodが作成される際、
`metadata.ownerReferences` フィールドにReplicaSetを特定する情報を保持します。
他にも、Service定義に基づいてEndpointSliceが作成されたり、KEDAのscaledObject作成時にHPAが存在しなければ併せて作成する (refs [About KEDA](/k8s-study/addons/keda/about_keda/))などKubernetesの多くのObjectは、`metadata.ownerReferences` を介して相互にリンクしています。

Kubernetesは、ReplicaSetを削除したときに残されたPodなど、owner referenceがなくなったObjectをチェックして削除します。

### Cascading Deletion

KubernetesはReplicaSetを削除した時に残されたPodなどowner referenceがなくなったObjectをチェックして削除します。
owner referenceがなくなったObjectを削除するとき、カスケード削除と呼ばれるプロセスで依存関係にあるobjectを自動的に削除するかどうかを制御できます。

- カスケード削除は2種類あります
    1. foreground cascading deletion
        - Owner Objectが削除進行中となり、先に依存Resourcesをcleanupした後でOwner Objectを削除する
            - `metadata.deletionTimestamp` を削除進行中となった時点のtimestampで更新
            - `metadata.finalizers` フィールドを `foregroundDeletion` に設定
            - Owner Objectは削除処理が完了するまではkubernetes api-serverを介して表示される
        - https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/#use-foreground-cascading-deletion
    1. background cascading deletion
        - default動作
        - kube-apiserverがOwner Resourcesをすぐに削除し、kube-constroller-managerがbackgroundで依存Resourcesをcleanupする
        - https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/#use-background-cascading-deletion

### `metadata.finalizers` field

- https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/
- https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/
- https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/

foreground cascading deletionでは `metadata.finalizers` フィールドを `foregroundDeletion` に設定します。
この `metadata.finalizers` fieldは削除対象としてマークされたリソースを完全に削除する前に、特定の条件が満たされるまでKubernetesを待機させるための名前空間付きのキーです。

finalizersの代表的な利用方法として、`kubernetes.io/pv-protection` があります。
これは `PersistentVolume` オブジェクトが誤って削除されるのを防ぐためのものです。
KubernetesはPodが`PersistentVolume` を利用中の場合は `kubernetes.io/pv-protection` finalizersを追加します。
`kubernetes.io/pv-protection` finalizersが追加された`PersistentVolume`を削除しようとした場合 `Terminating` Statusの状態で待機します。
`PersistentVolume` を利用しているPodが`PersistentVolume`の利用を停止するとKubernetesは `kubernetes.io/pv-protection` finalizersを削除し、`PersistentVolume`が削除されます。

Garbage Collectionにおいてowner objectの `metadata.finalizers` に `foregroundDeletion` が設定された場合、
owner objectはすぐに削除されず、owner referenceにowner objectが設定されている(依存関係にある)object の削除が完了したら `metadata.finalizers` から `foregroundDeletion` が削除され、owner objectが削除されます。




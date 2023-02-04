# KEDA
## References

- https://keda.sh/
- https://github.com/kedacore/keda
- https://github.com/kedacore/keda/blob/main/CREATE-NEW-SCALER.md
- https://tech.quickguard.jp/posts/pod-autoscaling-using-prometheus/
- https://learn.microsoft.com/ja-jp/azure/aks/keda-about
- https://www.bionconsulting.com/blog/using-keda-to-trigger-hpa-with-prometheus-metrics
- https://aws.amazon.com/jp/blogs/mt/proactive-autoscaling-kubernetes-workloads-keda-metrics-ingested-into-aws-amp/

## About KEDA

- `KEDA(Kubernetes Event-driven Autoscaling)`
    - イベント駆動のPod Autoscaler
    - Architecture
        - https://keda.sh/docs/2.9/concepts/#architecture
          ![](https://keda.sh/img/keda-arch.png){ width="300" }
    - KEDA は `HPA(Horizontal Pod Autoscaler)` を置き換えるものではなく、HPAと協調して動作します
        - `ScaledObject` リソース作成時にHPAリソースが未作成の場合は併せて作成してくれます

            !!! info
                **KEDAによって作成されたHPAリソースは`ScaledObject` リソース削除時に併せて削除されます**

                - KEDAはHPA作成時に[.metadata.ownerReferences](https://github.com/kedacore/keda/blob/v2.9.1/vendor/sigs.k8s.io/controller-runtime/pkg/controller/controllerutil/controllerutil.go#L74-L81) を付加します。`.metadata.ownerReferences` は自身が従属する親リソースに関する情報が格納され、kube-controller-managerの [Kubernetes garbage-collection](https://kubernetes.io/ja/docs/concepts/workloads/controllers/garbage-collection/) によって親リソースが存在しない場合は削除されます。
                - KEDAが作成したHPAのmanifestsに `.metadata.ownerReferences` があることを確認
                  <details><summary>HPAリソースの.metadata.ownerReferences</summary>
                  ```
                  $ kubectl get hpa keda-hpa-nginx-scaledobject -o yaml | yq .metadata.ownerReferences -y
                  - apiVersion: keda.sh/v1alpha1
                    blockOwnerDeletion: true
                    controller: true
                    kind: ScaledObject
                    name: nginx-scaledobject
                    uid: 38ffaf3a-8033-4d97-837a-a919e7f9eaec
                  ```
                  </details>

                - `ScaledObject` リソース削除直後にgarbage-collectionによりHPAも削除されることを確認
                  <details><summary>`ScaledObject` リソース削除直後のkube-controler-manager garbage-collection log</summary>
                  ```
                  I1217 16:16:29.579726       1 garbagecollector.go:379] according to the absentOwnerCache, object 84edf0d8-5d3f-418a-996c-24c218549866's owner keda.sh/v1alpha1/ScaledObject, nginx-scaledobject d
                  oes not exist in namespace default
                  I1217 16:16:29.579819       1 garbagecollector.go:518] classify references of [autoscaling/v1/HorizontalPodAutoscaler, namespace: default, name: keda-hpa-nginx-scaledobject, uid: 84edf0d8-5d3f-418a-996c-24c218549866].
                  solid: []v1.OwnerReference(nil)
                  dangling: []v1.OwnerReference{v1.OwnerReference{APIVersion:"keda.sh/v1alpha1", Kind:"ScaledObject", Name:"nginx-scaledobject", UID:"1e910cb9-5cf0-4109-966f-599c3676e7ff", Controller:(*bool)(0x4000b5b0e5), BlockOwnerDeletion:(*bool)(0x4000b5b0e6)}}
                  waitingForDependentsDeletion: []v1.OwnerReference(nil)
                  I1217 16:16:29.579916       1 garbagecollector.go:580] "Deleting object" object="default/keda-hpa-nginx-scaledobject" objectUID=84edf0d8-5d3f-418a-996c-24c218549866 kind="HorizontalPodAutoscaler" propagationPolicy=Background
                  I1217 16:16:29.580128       1 request.go:1181] Request Body: {"kind":"DeleteOptions","apiVersion":"v1","preconditions":{"uid":"84edf0d8-5d3f-418a-996c-24c218549866"},"propagationPolicy":"Background"}
                  I1217 16:16:29.581134       1 round_trippers.go:435] curl -v -XDELETE  -H "User-Agent: kube-controller-manager/v1.22.0 (linux/arm64) kubernetes/c2b5237/system:serviceaccount:kube-system:generic-garbage-collector" -H "Accept: application/vnd.kubernetes.protobuf,application/json" 'https://k8s-master:6443/apis/autoscaling/v1/namespaces/default/horizontalpodautoscalers/keda-hpa-nginx-scaledobject'
                  I1217 16:16:29.584188       1 graph_builder.go:632] GraphBuilder process object: events.k8s.io/v1/Event, namespace default, name nginx-scaledobject.1731a0d8c5b4e8ed, uid 424a0f3c-9565-4950-9c61-ef8fb1d8a081, event type add, virtual=false
                  I1217 16:16:29.593137       1 graph_builder.go:632] GraphBuilder process object: autoscaling/v1/HorizontalPodAutoscaler, namespace default, name keda-hpa-nginx-scaledobject, uid 84edf0d8-5d3f-418a-996c-24c218549866, event type delete, virtual=false
                  I1217 16:16:29.593255       1 resource_quota_monitor.go:355] QuotaMonitor process object: autoscaling/v1, Resource=horizontalpodautoscalers, namespace default, name keda-hpa-nginx-scaledobject, uid 84edf0d8-5d3f-418a-996c-24c218549866, event type delete
                  I1217 16:16:29.593686       1 round_trippers.go:454] DELETE https://k8s-master:6443/apis/autoscaling/v1/namespaces/default/horizontalpodautoscalers/keda-hpa-nginx-scaledobject 200 OK in 12 milliseconds
                  ```
                  </details>

        - `ScaledObject` リソースで既に作成済みHPAリソースを指定することも可能なので、既にHPAが動作する環境に対しても導入可能です

- KEDAの役割
    1. Agent
        - Kubernetes Deploymentをアクティブにしたり、非アクティブにしたりして、イベントなしでゼロからスケールさせます
          これは、KEDAをインストールした際に実行される `keda-operator` コンテナの主要な役割の1つです。

            !!! info
                KEDAが直接HPAのReplicasを更新する訳ではありません。
                Metrics Adapterの項でも記載の通り、HPAはKEDAが公開するcustom metricsを参照しscaleします。

    1. Metrics Adapter
        - https://keda.sh/docs/2.9/operate/metrics-server/
        - HPAで利用可能なmetricsとして[kubernetes-sigs/metrics-server](https://github.com/kubernetes-sigs/metrics-server) で取得可能なCPUやMemoryの使用率(metrics.k8s.io)の他にもcustom metrics(custom.metrics.k8s.io, or external.metrics.k8s.io)(e.g. PrometheusやKEDA) にも対応しています。これらcustom metricsをHPAが参照するためには仲介役となるMetrics Adapterが必要です。(refs https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#scaling-on-custom-metrics)
        - metrics-serverとしてevent sourceから取得したデータを公開します
          HPAはKEDAが公開するmetrics serverから値を取得します。
          これは、KEDAをインストールした際に実行される `keda-operator-metrics-apiserver` コンテナの役割です。

            !!! warning
                [KEDA v2.9.0](https://github.com/kedacore/keda/releases/tag/v2.9.0) で keda-operator-metrics-apiserverは `deprecated` となり、`keda-operator` コンテナがmetrics serverの役割も担います

## Custom Resources

- https://keda.sh/docs/2.9/concepts/scaling-deployments/

### ScaledObject

- https://keda.sh/docs/2.9/concepts/scaling-deployments/
- https://github.com/kedacore/keda/blob/main/apis/keda/v1alpha1/scaledobject_types.go
- `Deployments` や `StatefulSets` などのworkloadsをscaleさせるためのtriigerや対象workloadsを指定するcustom resources
    - custom resource workloads(e.g. argo-rollouts) のscaleをサポートしています
        - [KEDA scales any CustomResource that implements Scale subresource #703](https://github.com/kedacore/keda/issues/703)
        - 対象となるCustom Resourceが/scale subresourceを定義している必要があります

### ScaledJob

- `Job` workloadsをscaleさせるためのtriigerや対象workloadsを指定するcustom resources
- https://keda.sh/docs/2.0/concepts/scaling-jobs/#scaledjob-spec
- https://github.com/kedacore/keda/blob/main/apis/keda/v1alpha1/scaledjob_types.go

### Authentication

- https://keda.sh/docs/2.9/concepts/authentication/
- event sourceからmetricsを取得するために認証情報やsecret情報が必要となることがあります(e.g. datadogのapp key/api key)
    1. ScaledObjectごとに設定する
        - コンテナから参照可能なsecretを利用可能(ScaledObjectからsecretを直接参照はできない)
        - 利用可能かどうかはscalerに依りそう
            - [MySQL scaler](https://keda.sh/docs/2.9/scalers/mysql/)の`passwordFromEnv`
            - [RabbitMQ scaler](https://keda.sh/docs/2.9/scalers/rabbitmq-queue/) の `host` (接続ホストおよびuser/pass を記載した文字列) のsecretを作成し、host parameterにsecret-key-nameを指定する
    1. namespaceごとに利用可能な `TriggerAuthentication` リソースを利用する
    1. cluster wideに利用可能な `ClusterTriggerAuthentication` リソースを利用する

#### TriggerAuthentication

- https://github.com/kedacore/keda/blob/main/apis/keda/v1alpha1/triggerauthentication_types.go

#### ClusterTriggerAuthentication

- https://github.com/kedacore/keda/blob/main/apis/keda/v1alpha1/triggerauthentication_types.go


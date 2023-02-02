- https://argoproj.github.io/argo-rollouts/
- https://github.com/argoproj/argo-rollouts
- https://techstep.hatenablog.com/entry/2020/10/13/084905

Argo Rolloutsは、`Blue/Green` や `Canary` といったdeploy strategiesを提供します。

Kubernetes Deployment標準のdeploy strategies(`RollingUpdate`) と比較し高度なデプロイ機能を提供します。
([参考: Why Argo Rollouts?](https://argoproj.github.io/argo-rollouts/#why-argo-rollouts))

### Architecture

- https://argoproj.github.io/argo-rollouts/architecture/
    ![](https://argoproj.github.io/argo-rollouts/architecture-assets/argo-rollout-architecture.png){ width="600" }

#### `Argo Rollout Controller`
- `Rollout` resource typeを監視し変更が生じると定義と同じ状態にする役割を持つコントローラ

#### `Rollout` resource
- `Argo Rollout` のCustomResource
    - Deployment resource typeとほぼ互換性がありますが、deployment strategiesとしてcanaryやblue-greenをサポートしそれらのためのフィールドを持つ

#### `ReplicaSet`
- [Kubernetesの標準的なReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
    - `Argo Rollout` が作成・管理します
    - rolloutで管理するためのメタデータも追加されます
    - 上図では`Canary ReplicaSet` や `Stable ReplicaSet` と表記されている箇所
        - `旧version application`が動作する`Stable ReplicaSet`
        - stable昇格前の新version applicationが動作する`Canary ReplicaSet`

####  `Ingress` / `Service`
- `Service`に対するtrafficをどのReplicaSetへroutingするのかをselectorで管理します
- `Ingress` と連携することでcanary serviceに対するtraffic routingの比重やrouting ruleを制御できます
    - `Service` のみの場合、stable と canary のPod数でのみtraffic routingの比重を制御します
    - `Ingress` の場合、パーセンテージ / HTTP Header / Mirror といった手法でのtraffic routingを実現できます

        !!! warning

            HTTP Header や MirrorはIngress Controllerによってサポート状況が異なります

#### `AnalysisTemplate` / `AnalysisRun`
- https://argoproj.github.io/argo-rollouts/features/analysis/
- `Analysis` とは`Argo Rollout`がdeploy中にMetricsProviderに接続し特定のMetricsに対する閾値を設定しておくことで新version applicationが正常に動いているかどうかを検証する仕組みです
    - Metricsが良好な場合はrolloutはdeployを継続し、そうでない場合はdeployを中断しrollbackします
- `Analysis` を実行するためのCutomResourceとして `AnalysisTemplate` と `AnalysisRun` があります
    - `AnalysisTemplate`
        - MetricsProviderやAnalysisに使用するMetricsに関する設定
        - Rollout Resourceに直接設定、`AnalysisTemplate` もしくは `ClusterAnalysisTemplate` Resourceとして定義する
    - `AnalysisRun`
        - `AnalysisTemplate` の実行結果
        - 特定の `Rollout` Resourceにscopeされます

### Deploy Strategies

rollout CRDの `.spec.strategy` で `blueGreen` もしくは `canary` を指定します。

#### Blue/Green

- https://argoproj.github.io/argo-rollouts/features/bluegreen/
- rolloutのBlue/Greenデプロイでは`activeService`と`previewService`という2つのServiceを指定します
    - `activeService` は旧version application ReplicaSetへtrafficをroutingします
    - `previewService` は新version application ReplicaSetへtrafficをroutingします
- rolloutの `.spec.template` が定義されている場合は新ReplicaSetを作成します
    - `activeService` にtrafficが流れていない場合はすぐに新ReplicaSetへ切り替え、そうでない場合は新ReplicaSetが利用可能になるまでは旧ReplicaSetへroutingします
    - 新ReplicaSetが利用可能になったら`activeService`を新ReplicaSetへroutingを切り替えます

#### Canary

- https://argoproj.github.io/argo-rollouts/features/canary/
- rolloutのCanaryデプロイでは旧Serviceには旧version applicationに対するtrafficをroutingしつつtrafficを徐々に新Serviceにroutingします
    - rollout CRDの `spec.strategy.canary.steps` で新Serviceへtrafficをroutingする比重と移行間隔を指定します

### Progressive Delivery

`Progressive Delivery` とはContinuous Deliveryを発展させた考え方で、canary deploy途中で新version applicationを解析し正常であればdeployを継続、異常であれば旧version applicationへrollbackするといった考え方や仕組みを指します。Argo Rolloutsでは `Analysis` 機能を利用します。

[引用: Leveling Up Your CD: Unlocking Progressive Delivery on Kubernetes – Daniel Thomson & Jesse Suen, Intuit](https://static.sched.com/hosted_files/kccncna19/f2/Progressive%20Delivery%20%26%20Argo%20Rollouts.pdf)

<figure markdown>
![](/addons/argo/argo-rollouts/conventional.png){ width="300" }
<figcaption>従来のCD</figcaption>
</figure>

<figure markdown>
![](/addons/argo/argo-rollouts/progressive_delivery.png){ width="300" }
<figcaption>`progressive delivery`</figcaption>
</figure>

#### Analysis

- Rollout の `sepc.strategy.canary.analysis`
- blue/green strategyでも`Analysis` を利用可能
    - 新version applicationのReplicaSetのscaleが完了しトラフィックを新しいバージョンに切り替える前後で `AnalysisRun` を起動できます
    - https://argoproj.github.io/argo-rollouts/features/analysis/#bluegreen-pre-promotion-analysis
    - https://argoproj.github.io/argo-rollouts/features/analysis/#bluegreen-post-promotion-analysis
    - https://aws.amazon.com/jp/blogs/architecture/use-amazon-eks-and-argo-rollouts-for-progressive-delivery/

##### Analysisに関するカスタムリソース

- https://argoproj.github.io/argo-rollouts/features/analysis/#custom-resource-definitions
    - `AnalysisTemplate`
        - 解析方法についての定義したカスタムリソース
        - namespaceごと
    - `ClusterAnalysisTemplate`
        - `AnalysisTemplate` と同じ
        - cluster wide
    - `AnalysisRun`
        - `AnalysisTemplate` を実行するためのインスタンス化されたもの
        - Kubernetesの `Job` リソースと似ていて最終的に完了します
        - 実行結果としては `Successful`、`Failed`、`Inconclusive` があり、それぞれがrolloutのdeployが継続、中断、または一時停止されるかに影響します。
    - `Experiment`
        - 後述

#### `experimentation`

- https://argoproj.github.io/argo-rollouts/features/experiment/
- 1つまたは複数のReplicaSetをエフェメラルなリソースとして起動させ、backgroundでAnalysisRunを実行させることで新version applicationの正常性確認を行えます
    - `Experiment` で1つまたは複数のReplicaSetと`AnalysisTemplate`の指定を行います
    - `Rollout`リソース内のstepsとして`experiment`を定義することも可能です
        - https://argoproj.github.io/argo-rollouts/features/experiment/#integration-with-rollouts
        - https://github.com/argoproj/argo-rollouts/blob/master/examples/rollout-experiment-step.yaml


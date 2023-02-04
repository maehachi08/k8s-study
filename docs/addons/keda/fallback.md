# KEDA

業務でfallbackについて調査する機会がありましたので自分なりに整理しておきます。

## Fallback

- https://keda.sh/docs/2.9/concepts/scaling-deployments/#fallback
    - Triggerで指定したevent sourceからmetricsが取得できない場合の代替動作
    - KEDAが公開するcustom metrics(`/metrics`) で取得可能な `metric value` と `replicas` の値を(正しく取得・計算できないため) 正規化した値で返す

    !!! warning
        fallbackを使用する際の注意点

        - `spec.triggers.metricType` が `AverageValue` である場合に限られます
            - [refs v2.9.1](https://github.com/kedacore/keda/blob/v2.9.1/pkg/fallback/fallback.go#L38-L41)
            - CPU/Memory scalerや `spec.triggers.metricType` が `Value` であるscalerは未サポート
        - `ScaledObjects` でのみサポートされています
            - `ScaledJobs` は未サポート

### Fallback behavior

1. event sourceからmetrics取得に失敗した場合、`NumberOfFailures` に1を加算する
    - https://github.com/kedacore/keda/blob/v2.9.1/pkg/fallback/fallback.go#L63
1. `NumberOfFailures > spec.fallback.failureThreshold` の場合、fallback処理を呼び出す
    - https://github.com/kedacore/keda/blob/v2.9.1/pkg/fallback/fallback.go#L74-L75
1. `metric value` と `replicas` として正規化した値を返す
    - 正規化のための数式は後述

### `metric value`

- 以下計算式で求めます
    1. https://keda.sh/docs/2.9/concepts/scaling-deployments/#fallback
        - https://github.com/kedacore/keda/blob/v2.9.1/pkg/fallback/fallback.go#L105
            ```
            target metric value * fallback replicas
            ```

            !!! Note "fallback動作時の `target metric value` は `AverageValue` である"
                - https://github.com/kedacore/keda/blob/v2.9.1/pkg/fallback/fallback.go#L102
                - https://github.com/kedacore/keda/blob/v2.9.1/pkg/scaling/cache/scalers_cache.go#L344-L361

            !!! Note "e.g. (cloudwatch scaler)"

                ```
                spec.triggers.targetMetricValue: 100
                target metric value: 50
                fallback replicas: 2
                ```

                (100 / 2) * 2 = 100

    1. https://keda.sh/docs/2.9/concepts/scaling-deployments/#triggers
        - https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/
        - `With AverageValue, the value returned from the custom metrics API is divided by the number of Pods before being compared to the target.`
        - 最初にtarget metric valueの `AverageValue` を求め、fallback replicasで掛けている
            - https://github.com/kedacore/keda/blob/v2.9.1/pkg/fallback/fallback.go#L102
            - https://github.com/kedacore/keda/blob/v2.9.1/pkg/scaling/cache/scalers_cache.go#L344-L361
                - metricSpecs [ExternalMetricSource](https://github.com/kedacore/keda/blob/v2.9.1/vendor/k8s.io/api/autoscaling/v2/types.go#L302-L307) が入るため要素数は常に `2` になる?(推測)

            ```
            metric value / number of pod
            ```

            !!! Note "e.g."

                ```
                target metric value: 100
                fallback replicas: 2
                ```

                (100 * 2) / 2 = 100

                **このexampleの場合、fallbackが動作した場合のmetric valueは `100` となります**


### `replicas`

- 以下計算式で求めます
    1. https://keda.sh/docs/2.9/concepts/scaling-deployments/#triggers
        ```
        metric value / target metric value
        (`spec.fallback.replicas` と同値となる)
        ```

        !!! Note "e.g."

            ```
            target metric value: 100
            fallback replicas: 2
            ```

            (100 * 2) / 100 = 2

            **このexampleの場合、fallbackが動作した場合のreplicasは `2` となります**




## 検証

1. deploy
    - [Install KEDA > 動作確認](/addons/keda/install/) で使用したmanifestsを使用
        ```
        kubectl apply -f deployment.yaml
        ```
1. Deploymentリソースの `Replicas` を削除
    - HPAリソースの `Replicas` と Deploymentリソースの `Replicas` が競合してしまう
    - https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#migrating-deployments-and-statefulsets-to-horizontal-autoscaling

        <details><summary>1. `kubectl.kubernetes.io/last-applied-configuration` annotationsに `spec.replicas` が存在することを確認</summary>
        ```
        $ kubectl get deployment nginx-deployment -o yaml | yq .metadata.annotations
        deployment.kubernetes.io/revision: "1"
        kubectl.kubernetes.io/last-applied-configuration: |
          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},"spec":{"replicas":2,"selector":{"matchLabels":{"app":"nginx"}},"template":{"metadata":{"annotations":{"prometheus.io/port":"9113","prometheus.io/scrape":"true"},"labels":{"app":"nginx"}},"spec":{"containers":[{"image":"nginx:1.14.2","name":"nginx","ports":[{"containerPort":80}],"volumeMounts":[{"mountPath":"/etc/nginx/nginx.conf","name":"nginx-conf","readOnly":true,"subPath":"nginx.conf"}]},{"args":["-nginx.scrape-uri=http://localhost/nginx_status"],"image":"nginx/nginx-prometheus-exporter:0.11.0","name":"nginx-exporter","ports":[{"containerPort":9113}]}],"volumes":[{"configMap":{"items":[{"key":"nginx.conf","path":"nginx.conf"}],"name":"nginx-conf"},"name":"nginx-conf"}]}}}}
        ```
        </details>

        <details><summary>2. `kubectl.kubernetes.io/last-applied-configuration` annotationsの `spec.replicas` を削除</summary>
        ```
        kubectl apply edit-last-applied deployment/nginx-deployment
        ```
        </details>

        <details><summary>3. `kubectl.kubernetes.io/last-applied-configuration` annotationsから `spec.replicas` が削除されていることを確認</summary>
        ```
        $ kubectl get deployment nginx-deployment -o yaml | yq .metadata.annotations
        deployment.kubernetes.io/revision: "1"
        kubectl.kubernetes.io/last-applied-configuration: |
          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},"spec":{"selector":{"matchLabels":{"app":"nginx"}},"template":{"metadata":{"annotations":{"prometheus.io/port":"9113","prometheus.io/scrape":"true"},"labels":{"app":"nginx"}},"spec":{"containers":[{"image":"nginx:1.14.2","name":"nginx","ports":[{"containerPort":80}],"volumeMounts":[{"mountPath":"/etc/nginx/nginx.conf","name":"nginx-conf","readOnly":true,"subPath":"nginx.conf"}]},{"args":["-nginx.scrape-uri=http://localhost/nginx_status"],"image":"nginx/nginx-prometheus-exporter:0.11.0","name":"nginx-exporter","ports":[{"containerPort":9113}]}],"volumes":[{"configMap":{"items":[{"key":"nginx.conf","path":"nginx.conf"}],"name":"nginx-conf"},"name":"nginx-conf"}]}}}}
        ```
        </details>

        <details><summary>4. `deployment.yaml` のDeployment manifestsから `spec.replicas` を削除する</summary>
        ```
        vim deployment.yaml

        や

        sed -i -e '/replicas:\s2/d' deployment.yaml

        など
        ```
        </details>

1. scaledObjectを確認
    - `Spec` の設定が想定通りであることを確認
    - `Status > Conditions` の `Type: Fallback` でFallbackが発生していないことを確認
        <details><summary>scaledObject</summary>
        ```
        $ kubectl describe scaledObject nginx-scaledobject
        Name:         nginx-scaledobject
        Namespace:    default
        Labels:       deploymentName=nginx-deployment
                      scaledobject.keda.sh/name=nginx-scaledobject
        Annotations:  <none>
        API Version:  keda.sh/v1alpha1
        Kind:         ScaledObject
        Metadata:
          Creation Timestamp:  2023-02-03T14:44:15Z
          Finalizers:
            finalizer.keda.sh
          Generation:  1
          Managed Fields:
            API Version:  keda.sh/v1alpha1
            Fields Type:  FieldsV1
            fieldsV1:
              f:metadata:
                f:finalizers:
                  .:
                  v:"finalizer.keda.sh":
                f:labels:
                  f:scaledobject.keda.sh/name:
            Manager:      keda
            Operation:    Update
            Time:         2023-02-03T14:44:15Z
            API Version:  keda.sh/v1alpha1
            Fields Type:  FieldsV1
            fieldsV1:
              f:status:
                .:
                f:externalMetricNames:
                f:hpaName:
                f:originalReplicaCount:
                f:scaleTargetGVKR:
                  .:
                  f:group:
                  f:kind:
                  f:resource:
                  f:version:
                f:scaleTargetKind:
            Manager:      keda
            Operation:    Update
            Subresource:  status
            Time:         2023-02-03T14:44:15Z
            API Version:  keda.sh/v1alpha1
            Fields Type:  FieldsV1
            fieldsV1:
              f:metadata:
                f:annotations:
                  .:
                  f:kubectl.kubernetes.io/last-applied-configuration:
                f:labels:
                  .:
                  f:deploymentName:
              f:spec:
                .:
                f:fallback:
                  .:
                  f:failureThreshold:
                  f:replicas:
                f:maxReplicaCount:
                f:minReplicaCount:
                f:scaleTargetRef:
                  .:
                  f:name:
                f:triggers:
            Manager:      kubectl-client-side-apply
            Operation:    Update
            Time:         2023-02-03T14:44:15Z
            API Version:  keda.sh/v1alpha1
            Fields Type:  FieldsV1
            fieldsV1:
              f:status:
                f:conditions:
                f:health:
                  .:
                  f:s0-prometheus-nginx_http_requests_total:
                    .:
                    f:numberOfFailures:
                    f:status:
            Manager:         keda-adapter
            Operation:       Update
            Subresource:     status
            Time:            2023-02-03T14:44:31Z
          Resource Version:  19302409
          UID:               20ef1bf8-5476-4e17-ac31-2b33e73c758a
        Spec:
          Fallback:
            Failure Threshold:  3
            Replicas:           5
          Max Replica Count:    5
          Min Replica Count:    1
          Scale Target Ref:
            Name:  nginx-deployment
          Triggers:
            Metadata:
              Metric Name:     nginx_http_requests_total
              Query:           sum(rate(nginx_http_requests_total{app="nginx"}[2m]))
              Server Address:  http://prometheus-server.monitoring.svc.cluster.local
              Threshold:       3
            Type:              prometheus
        Status:
          Conditions:
            Message:  ScaledObject is defined correctly and is ready for scaling
            Reason:   ScaledObjectReady
            Status:   True
            Type:     Ready
            Message:  Scaling is not performed because triggers are not active
            Reason:   ScalerNotActive
            Status:   False
            Type:     Active
            Message:  No fallbacks are active on this scaled object
            Reason:   NoFallbackFound
            Status:   False
            Type:     Fallback
          External Metric Names:
            s0-prometheus-nginx_http_requests_total
          Health:
            s0-prometheus-nginx_http_requests_total:
              Number Of Failures:  0
              Status:              Happy
          Hpa Name:                keda-hpa-nginx-scaledobject
          Original Replica Count:  1
          Scale Target GVKR:
            Group:            apps
            Kind:             Deployment
            Resource:         deployments
            Version:          v1
          Scale Target Kind:  apps/v1.Deployment
        Events:
          Type    Reason              Age   From           Message
          ----    ------              ----  ----           -------
          Normal  KEDAScalersStarted  36m   keda-operator  Started scalers watch
          Normal  ScaledObjectReady   36m   keda-operator  ScaledObject is ready for scaling
        ```
        </details>

1. MetricsProviderの`Prometheus` をdownさせます
    - `prometheus-server` のreplicasを `0` に変更する
        <details><summary>1. prometheus-server podが1つ起動していることを確認</summary>
        ```
        $ kubectl get deployments -n monitoring prometheus-server
        NAME                READY   UP-TO-DATE   AVAILABLE   AGE
        prometheus-server   1/1     1            1           96d
        ```
        </details>

        <details><summary>2. prometheus-server deploymentのreplicasを0に変更する</summary>
        ```
        $ kubectl edit deployments -n monitoring prometheus-server
        deployment.apps/prometheus-server edited
        ```
        </details>

        <details><summary>3. prometheus-server podが0となったことを確認</summary>
        ```
        $ kubectl get deployments -n monitoring prometheus-server
        NAME                READY   UP-TO-DATE   AVAILABLE   AGE
        prometheus-server   0/0     0            0           96d
        ```
        </details>

1. Fallbackが発生することを確認
    - `scaledObject` リソースの `Status` フィールドに変化を確認
        <details><summary>1. `Health` の `Number Of Failures` が `1` にカウントされ、`Status` が `Happy` から `Failing` に変化</summary>
        ```
        $ kubectl describe scaledObject nginx-scaledobject
        
        snip...
        
        Status:
          Conditions:
            Message:  ScaledObject is defined correctly and is ready for scaling
            Reason:   ScaledObjectReady
            Status:   True
            Type:     Ready
            Message:  Scaling is not performed because triggers are not active
            Reason:   ScalerNotActive
            Status:   False
            Type:     Active
            Message:  No fallbacks are active on this scaled object
            Reason:   NoFallbackFound
            Status:   False
            Type:     Fallback
          External Metric Names:
            s0-prometheus-nginx_http_requests_total
          Health:
            s0-prometheus-nginx_http_requests_total:
              Number Of Failures:  1
              Status:              Failing
        ```
        </details>
        
        <details><summary>2. `Number Of Failures`が`2`にカウントアップし、`Events`フィールドに`prometheus-server`へ接続できなかった旨のエラーが記録された。かつ、`Status > Conditions > Type: Fallback` で `Status: True` へ変化があった</summary>
        ```
        $ kubectl describe scaledObject nginx-scaledobject
        
        snip...
        
        Status:
          Conditions:
            Message:  ScaledObject is defined correctly and is ready for scaling
            Reason:   ScaledObjectReady
            Status:   True
            Type:     Ready
            Message:  Scaling is not performed because triggers are not active
            Reason:   ScalerNotActive
            Status:   False
            Type:     Active
            Message:  At least one trigger is falling back on this scaled object
            Reason:   FallbackExists
            Status:   True
            Type:     Fallback
          External Metric Names:
            s0-prometheus-nginx_http_requests_total
          Health:
            s0-prometheus-nginx_http_requests_total:
              Number Of Failures:  2
              Status:              Failing
        
        snip...
        
        Events:
          Type     Reason              Age   From           Message
          ----     ------              ----  ----           -------
          Normal   KEDAScalersStarted  43m   keda-operator  Started scalers watch
          Normal   ScaledObjectReady   43m   keda-operator  ScaledObject is ready for scaling
          Warning  KEDAScalerFailed    1s    keda-operator  Get "http://prometheus-server.monitoring.svc.cluster.local/api/v1/query?query=sum%28rate%28nginx_http_requests_total%7Bapp%3D%22nginx%22%7D%5B2m
        %5D%29%29&time=2023-02-03T15:27:18Z": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
        ```
        </details>

        <details><summary>3. `Number Of Failures`が`3`にカウントアップ</summary>
        ```
        $ kubectl describe scaledObject nginx-scaledobject
        
        snip...
        
        Status:
          Conditions:
            Message:  ScaledObject is defined correctly and is ready for scaling
            Reason:   ScaledObjectReady
            Status:   True
            Type:     Ready
            Message:  Scaling is not performed because triggers are not active
            Reason:   ScalerNotActive
            Status:   False
            Type:     Active
            Message:  No fallbacks are active on this scaled object
            Reason:   NoFallbackFound
            Status:   False
            Type:     Fallback
          External Metric Names:
            s0-prometheus-nginx_http_requests_total
          Health:
            s0-prometheus-nginx_http_requests_total:
              Number Of Failures:  3
              Status:              Failing
        
        ```
        </details>

1. HPAのReplicasがFallbackの設定通りとなったことを確認
    <details><summary>get</summary>
    ```
    $ kubectl get hpa
    NAME                          REFERENCE                     TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
    keda-hpa-nginx-scaledobject   Deployment/nginx-deployment   0/3 (avg)   1         5         5          43m
    ```
    </details>

    <details><summary>describe</summary>
    ```
    $ kubectl describe hpa keda-hpa-nginx-scaledobject
    Name:                                                                keda-hpa-nginx-scaledobject
    Namespace:                                                           default
    Labels:                                                              app.kubernetes.io/managed-by=keda-operator
                                                                         app.kubernetes.io/name=keda-hpa-nginx-scaledobject
                                                                         app.kubernetes.io/part-of=nginx-scaledobject
                                                                         app.kubernetes.io/version=2.8.1
                                                                         deploymentName=nginx-deployment
                                                                         scaledobject.keda.sh/name=nginx-scaledobject
    Annotations:                                                         <none>
    CreationTimestamp:                                                   Fri, 03 Feb 2023 14:44:15 +0000
    Reference:                                                           Deployment/nginx-deployment
    Metrics:                                                             ( current / target )
      "s0-prometheus-nginx_http_requests_total" (target average value):  0 / 3
    Min replicas:                                                        1
    Max replicas:                                                        5
    Deployment pods:                                                     5 current / 5 desired
    Conditions:
      Type            Status  Reason            Message
      ----            ------  ------            -------
      AbleToScale     True    SucceededRescale  the HPA controller was able to update the target scale to 1
      ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from external metric s0-prometheus-nginx_http_requests_total(&LabelSelector{MatchLabels:map[string]string{scaledobject.keda.sh/name: nginx-scaledobject,},MatchExpressions:[]LabelSelectorRequirement{},})
      ScalingLimited  True    TooFewReplicas    the desired replica count is less than the minimum replica count
    Events:
      Type     Reason                        Age                From                       Message
      ----     ------                        ----               ----                       -------
      Warning  FailedGetExternalMetric       29s (x3 over 68s)  horizontal-pod-autoscaler  unable to get external metric default/s0-prometheus-nginx_http_requests_total/&LabelSelector{MatchLabels:map[string]string{scaledobject.keda.sh/name: nginx-scaledobject,},MatchExpressions:[]LabelSelectorRequirement{},}: unable to fetch metrics from external metrics API: no matching metrics found for s0-prometheus-nginx_http_requests_total
      Warning  FailedComputeMetricsReplicas  29s (x3 over 68s)  horizontal-pod-autoscaler  invalid metrics (1 invalid out of 1), first error is: failed to get s0-prometheus-nginx_http_requests_total external metric: unable to get external metric default/s0-prometheus-nginx_http_requests_total/&LabelSelector{MatchLabels:map[string]string{scaledobject.keda.sh/name: nginx-scaledobject,},MatchExpressions:[]LabelSelectorRequirement{},}: unable to fetch metrics from external metrics API: no matching metrics found for s0-prometheus-nginx_http_requests_total
      Normal   SuccessfulRescale             14s                horizontal-pod-autoscaler  New size: 1; reason: All metrics below target
    ```
    </details>

1. keda-operator logからFallbackが発生したことを確認
    - 以下のような纏まりのlog
    - `Successfully set ScaleTarget replicas count to ScaledObject fallback.replicas`
        <details><summary>keda-operator log</summary>
        ```
        2023-02-03T15:27:18Z    ERROR   prometheus_scaler       error executing prometheus query        {"type": "ScaledObject", "namespace": "default", "name": "nginx-scaledobject", "error": "Get \"http://prometheus-server.monitoring.svc.cluster.local/api/v1/query?query=sum%28rate%28nginx_http_requests_total%7Bapp%3D%22nginx%22%7D%5B2m%5D%29%29&time=2023-02-03T15:27:15Z\": context deadline exceeded (Client.Timeout exceeded while awaiting headers)"}
        github.com/kedacore/keda/v2/pkg/scaling/cache.(*ScalersCache).IsScaledObjectActive
                /workspace/pkg/scaling/cache/scalers_cache.go:89
        github.com/kedacore/keda/v2/pkg/scaling.(*scaleHandler).checkScalers
                /workspace/pkg/scaling/scale_handler.go:278
        github.com/kedacore/keda/v2/pkg/scaling.(*scaleHandler).startScaleLoop
                /workspace/pkg/scaling/scale_handler.go:149
        
        2023-02-03T15:27:22Z    ERROR   prometheus_scaler       error executing prometheus query        {"type": "ScaledObject", "namespace": "default", "name": "nginx-scaledobject", "error": "Get \"http://prometheus-server.monitoring.svc.cluster.local/api/v1/query?query=sum%28rate%28nginx_http_requests_total%7Bapp%3D%22nginx%22%7D%5B2m%5D%29%29&time=2023-02-03T15:27:18Z\": context deadline exceeded (Client.Timeout exceeded while awaiting headers)"}
        github.com/kedacore/keda/v2/pkg/scaling/cache.(*ScalersCache).IsScaledObjectActive
                /workspace/pkg/scaling/cache/scalers_cache.go:94
        github.com/kedacore/keda/v2/pkg/scaling.(*scaleHandler).checkScalers
                /workspace/pkg/scaling/scale_handler.go:278
        github.com/kedacore/keda/v2/pkg/scaling.(*scaleHandler).startScaleLoop
                /workspace/pkg/scaling/scale_handler.go:149
        
        2023-02-03T15:27:22Z    ERROR   scalehandler    Error getting scale decision    {"scaledobject.Name": "nginx-scaledobject", "scaledObject.Namespace": "default", "scaleTarget.Name": "nginx-deployment", "error": "Get \"http://prometheus-server.monitoring.svc.cluster.local/api/v1/query?query=sum%28rate%28nginx_http_requests_total%7Bapp%3D%22nginx%22%7D%5B2m%5D%29%29&time=2023-02-03T15:27:18Z\": context deadline exceeded (Client.Timeout exceeded while awaiting headers)"}
        github.com/kedacore/keda/v2/pkg/scaling.(*scaleHandler).checkScalers
                /workspace/pkg/scaling/scale_handler.go:278
        github.com/kedacore/keda/v2/pkg/scaling.(*scaleHandler).startScaleLoop
                /workspace/pkg/scaling/scale_handler.go:149
        
        2023-02-03T15:27:22Z    DEBUG   events  Warning {"object": {"kind":"ScaledObject","namespace":"default","name":"nginx-scaledobject","uid":"20ef1bf8-5476-4e17-ac31-2b33e73c758a","apiVersion":"keda.sh/v1alpha1","resourceVersion":"19308916"}, "reason": "KEDAScalerFailed", "message": "Get \"http://prometheus-server.monitoring.svc.cluster.local/api/v1/query?query=sum%28rate%28nginx_http_requests_total%7Bapp%3D%22nginx%22%7D%5B2m%5D%29%29&time=2023-02-03T15:27:18Z\": context deadline exceeded (Client.Timeout exceeded while awaiting headers)"}
        
        2023-02-03T15:27:22Z    INFO    scaleexecutor   Successfully set ScaleTarget replicas count to ScaledObject fallback.replicas   {"scaledobject.Name": "nginx-scaledobject", "scaledObject.Namespace": "default", "scaleTarget.Name": "nginx-deployment", "Original Replicas Count": 1, "New Replicas Count": 5}
        ```
        </details>


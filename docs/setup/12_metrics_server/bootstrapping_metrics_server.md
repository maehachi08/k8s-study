# metrics server

`metrics-server`とはkubeletやkube-apiserverから各種リソースのメトリックスを収集します。
収集したメトリックス情報はautoscaling([HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) や [VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler/))を行うために使用されます。

## 参考

- https://kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/
- https://github.com/kubernetes-sigs/metrics-server

## 構築手順

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

- `kubectl top pod` などで以下エラー

```
Error from server (ServiceUnavailable): the server is currently unable to handle the request (get pods.metrics.k8s.io)
```

- `kube-apiserver` で以下ログ

```
E0829 14:15:26.199732       1 controller.go:116] loading OpenAPI spec for "v1beta1.metrics.k8s.io" failed with: failed to retrieve openAPI spec, http error: ResponseCode: 503, Body: service unavailable
, Header: map[Content-Type:[text/plain; charset=utf-8] X-Content-Type-Options:[nosniff]]
I0829 14:15:26.199769       1 controller.go:129] OpenAPI AggregationController: action for item v1beta1.metrics.k8s.io: Rate Limited Requeue.
```



# About monitoring

## kubernetes monitoring

### Resource metrics pipeline

- https://kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/
    - Metrics APIは `/apis/metrics.k8s.io/` 下のパス
    - cpu usage と memory usage
    - [Metrics Serve](https://github.com/kubernetes-sigs/metrics-server)

### Resource usage monitoring

- https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/
    - Resource metrics pipeline
        - HPAや`kubectl top`のための限定的なメトリックス情報を提供する
        - metrics-serverが収集し `metrics.k8s.io` で提供する
    - Full metrics pipeline
        - kubernetesの完全なメトリックス情報を提供する
        - `custom.metrics.k8s.io`または`external.metrics.k8s.io` APIを実装することで、アダプターを介してそれらをKubernetesに公開する
        - Prometheusはfull metricsを提供する

### Monitor Node Health

- https://kubernetes.io/docs/tasks/debug-application-cluster/monitor-node-health/
    - [Node Problem Detector](https://github.com/kubernetes/node-problem-detector)

## Monitoring kubernetes with datadog

- https://docs.datadoghq.com/ja/agent/kubernetes/?tab=helm
- https://www.datadoghq.com/ja/blog/monitoring-kubernetes-datadog/
    - application podsからNode(kubelet)やcoredns, etcdのmetrics収集などkubernetes全般のmonitoringについて言及したページ
       - Pattern1: Agent as side-cars container
       - Pattern2: Agent with Autodiscovery
       - dd-agentコンテナをDaemonsetでデプロイする



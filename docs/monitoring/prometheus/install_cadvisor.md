## About cadvisor

https://github.com/google/cadvisor

cadvisor(Container Advisor) は実行中のコンテナのリソースのメトリックスを収集するためのツールです。
[kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) はKubernetes Objectに対するメトリックス収集を行うものなので、`Pod` のメトリックスは収集されますがコンテナごとのメトリックスは収集されません。

`cadvisor` で収集されるメトリックスについては以下ページに記載されています。

https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md

## Install

- install kustomize
   ```
   curl -s -o /tmp/install_kustomize.sh "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"

   # Raspberry Pi 4 is aarch64 (ARM 64-bit architecture)
   # refs https://github.com/kubernetes-sigs/kustomize/issues/4696
   sed -i -e 's/arm64/aarch64/g' /tmp/install_kustomize.sh

   bash -x ./tmp/install_kustomize.sh
   sudo mv ./kustomize /usr/local/bin/
   ```

- install cadvisor
    - https://github.com/google/cadvisor/tree/master/deploy/kubernetes#cadvisor-kubernetes-daemonset
      ```
      # https://github.com/google/cadvisor/releases
      VERSION=v0.46.0

      git clone https://github.com/google/cadvisor.git ~/work/cadvisor
      cd deploy/kubernetes/base && kustomize edit set image gcr.io/cadvisor/cadvisor:${VERSION} && cd ../../..
      kubectl kustomize deploy/kubernetes/base | kubectl apply -f -
      ```

## Dashboard

- https://grafana.com/grafana/dashboards/14282-cadvisor-exporter/
- [Install Grafana > Install Node Exporter Full Dashboards](/prometheus/install_grafana/#install-node-exporter-full-dashboards) 参照


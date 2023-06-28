## Prometheus Installing

- https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus

### Install


1. valuesファイルの修正
    - Ephemeral Storageの `storageClass` を OpenEBSのjiva storageのものとする( e.g. `openebs-jiva-csi-default`)

        !!! info
            - [Installing OpenEBS](../../addons/openebs/install.md) を実施済みの前提

    - `Prometheus` や `Alertmanager` の WebUIを利用したい場合
        - NodeIp:NodePort でアクセスする場合
            - `service.type: NodePort`
        - `MetalLB` で払い出したexternal ipでアクセスする場合
            - `service.type: LoadBalancer`
            - `annotations` に `metallb.universe.tf/address-pool: ip-pool`

            !!! info
                - [MetalLB](../../addons/metallb.md) を実施済みの前提

    ```
    mkdir -p ~/work/prometheus
    curl -so ~/work/prometheus/prometheus.yaml https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/prometheus/values.yaml
    vim ~/work/prometheus/prometheus.yaml
    ```

      <details><summary>prometheus.yaml 修正後のdiff</summary>
      ```
      $ diff -u <(curl -s https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/prometheus/values.yaml) <(cat ~/work/prometheus/prometheus.yaml)
      --- /dev/fd/63  2022-10-30 14:07:09.532561438 +0000
      +++ /dev/fd/62  2022-10-30 14:07:09.540561310 +0000
      @@ -231,7 +231,7 @@
           ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
           ##   GKE, AWS & OpenStack)
           ##
      -    # storageClass: "-"
      +    storageClass: "openebs-jiva-csi-default"
      
           ## alertmanager data Persistent Volume Binding Mode
           ## If defined, volumeBindingMode: <volumeBindingMode>
      @@ -947,7 +947,7 @@
           ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
           ##   GKE, AWS & OpenStack)
           ##
      -    # storageClass: "-"
      +    storageClass: "openebs-jiva-csi-default"
      
           ## Prometheus server data Persistent Volume Binding Mode
           ## If defined, volumeBindingMode: <volumeBindingMode>
      @@ -1403,7 +1403,7 @@
           ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
           ##   GKE, AWS & OpenStack)
           ##
      -    # storageClass: "-"
      +    storageClass: "openebs-jiva-csi-default"
      
           ## pushgateway data Persistent Volume Binding Mode
           ## If defined, volumeBindingMode: <volumeBindingMode>
      ```
      </details>

1. install

    ```
    helm upgrade -i prometheus -n monitoring --create-namespace prometheus-community/prometheus -f ~/work/prometheus/prometheus.yaml
    ```

1. 作成されたリソースを確認する
    <details><summary>Service</summary>
    ```
    $ kubectl get services -n monitoring
    NAME                            TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
    prometheus-alertmanager         ClusterIP      10.32.0.114   <none>          80/TCP         33h
    prometheus-kube-state-metrics   ClusterIP      10.32.0.246   <none>          8080/TCP       33h
    prometheus-node-exporter        ClusterIP      10.32.0.31    <none>          9100/TCP       33h
    prometheus-pushgateway          ClusterIP      10.32.0.138   <none>          9091/TCP       33h
    prometheus-server               ClusterIP      10.32.0.75    <none>          80/TCP         33h
    ```
    </details>

    <details><summary>Deployment</summary>
    ```
    $ kubectl get deployments -n monitoring
    NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
    prometheus-alertmanager         1/1     1            1           33h
    prometheus-kube-state-metrics   1/1     1            1           33h
    prometheus-pushgateway          1/1     1            1           33h
    prometheus-server               1/1     1            1           33h
    ```
    </details>


    <details><summary>DaemonSet</summary>
    ```
    $ kubectl get DaemonSet -n monitoring -l app=prometheus
    NAME                       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
    prometheus-node-exporter   2         2         2       2            2           <none>          35h
    ```
    </details>

    <details><summary>PersistentVolumeClaim</summary>
    ```
    $ kubectl get PersistentVolumeClaim -n monitoring
    NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS               AGE
    prometheus-alertmanager   Bound    pvc-93589533-c5b1-4dc4-8089-8dcccd42b8cd   2Gi        RWO            openebs-jiva-csi-default   33h
    prometheus-server         Bound    pvc-836ef65a-da18-4453-96a6-7b909d0c668b   8Gi        RWO            openebs-jiva-csi-default   33h
    ```
    </details>

    <details><summary>PersistentVolume</summary>
    ```
    $ kubectl get PersistentVolume -n monitoring pvc-93589533-c5b1-4dc4-8089-8dcccd42b8cd pvc-836ef65a-da18-4453-96a6-7b909d0c668b
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                STORAGECLASS               REASON   AGE
    pvc-93589533-c5b1-4dc4-8089-8dcccd42b8cd   2Gi        RWO            Delete           Bound    monitoring/prometheus-alertmanager   openebs-jiva-csi-default            33h
    pvc-836ef65a-da18-4453-96a6-7b909d0c668b   8Gi        RWO            Delete           Bound    monitoring/prometheus-server         openebs-jiva-csi-default            33h
    ```
    </details>

    <details><summary>ServiceAccount</summary>
    ```
    $ kubectl get ServiceAccount -n monitoring -l app=prometheus
    NAME                                   SECRETS   AGE
    default                                1         9d
    prometheus-alertmanager                1         35h
    prometheus-kube-prometheus-admission   1         4d23h
    prometheus-kube-state-metrics          1         35h
    prometheus-node-exporter               1         35h
    prometheus-pushgateway                 1         35h
    prometheus-server                      1         35h
    ```
    </details>

    <details><summary>ClusterRole</summary>
    ```
    $ kubectl get ClusterRole -n monitoring -l app=prometheus
    NAME                      CREATED AT
    prometheus-alertmanager   2022-10-30T03:25:38Z
    prometheus-pushgateway    2022-10-30T03:25:38Z
    prometheus-server         2022-10-30T03:25:38Z
    ```
    </details>

    <details><summary>ClusterRoleBinding</summary>
    ```
    $ kubectl get ClusterRoleBinding -n monitoring -l app=prometheus
    NAME                      ROLE                                  AGE
    prometheus-alertmanager   ClusterRole/prometheus-alertmanager   35h
    prometheus-pushgateway    ClusterRole/prometheus-pushgateway    35h
    prometheus-server         ClusterRole/prometheus-server         35h
    ```
    </details>

    <details><summary>ConfigMap</summary>
    ```
    $ kubectl get ConfigMap -n monitoring -l app=prometheus
    NAME                      DATA   AGE
    prometheus-alertmanager   2      35h
    prometheus-server         6      35h
    ```
    </details>

    !!! info
        - ConfigMapに格納されている `prometheus.yml` の内容を確認したい場合は以下コマンドで確認
            ```
            kubectl get ConfigMap -n monitoring prometheus-server -o jsonpath="{.data.prometheus\.yml}"
            ```

1. Confirm FQDN of prometheus-server service A Record
    - Grafanaをhelm installする際に `datasources.datasources[0].url` に設定するFQDNを確認する
    - `prometheus-server.monitoring.svc.cluster.local`

        ```
        $ nslookup prometheus-server.monitoring.svc.cluster.local `kubectl get ep -n kube-system kube-dns -o jsonpath="{.subsets[0].addresses[0].ip}"`
        Server:         10.200.2.235
        Address:        10.200.2.235#53
        
        Name:   prometheus-server.monitoring.svc.cluster.local
        Address: 10.32.0.75
        ```


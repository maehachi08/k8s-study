## Install Grafana

- https://github.com/grafana/helm-charts/tree/main/charts/grafana

### Install

1. valuesファイルの修正
    - Ephemeral Storageの `storageClass` を OpenEBSのjiva storageのものとする( e.g. `openebs-jiva-csi-default`)

        !!! info
            - [Installing OpenEBS](/addons/openebs/install/) を実施済みの前提

    - Grafana WebUIへのアクセスに`MetalLB`のexternal IPアドレスを使用する
        - `service.type: LoadBalancer`
        - `annotations` に `metallb.universe.tf/address-pool: ip-pool`

    ```
    mkdir -p ~/work/prometheus/grafana
    curl -so ~/work/prometheus/grafana/grafana.yaml https://raw.githubusercontent.com/grafana/helm-charts/main/charts/grafana/values.yaml
    vim ~/work/prometheus/grafana/grafana.yaml
    ```

      <details><summary>grafana.yaml 修正後のdiff</summary>

      ```
      $ diff -u <(curl -s https://raw.githubusercontent.com/grafana/helm-charts/main/charts/grafana/values.yaml) <(cat ~/work/prometheus/grafana/grafana.yaml)
      --- /dev/fd/63  2022-10-30 15:05:32.554153834 +0000
      +++ /dev/fd/62  2022-10-30 15:05:32.566153656 +0000
      @@ -157,12 +157,13 @@
       ##
       service:
         enabled: true
      -  type: ClusterIP
      +  type: LoadBalancer
         port: 80
         targetPort: 3000
           # targetPort: 4181 To be used with a proxy extraContainer
         ## Service annotations. Can be templated.
      -  annotations: {}
      +  annotations:
      +    metallb.universe.tf/address-pool: ip-pool
         labels: {}
         portName: service
         # Adds the appProtocol field to the service. This allows to work with istio protocol selection. Ex: "http" or "tcp"
      @@ -297,10 +297,10 @@
       persistence:
         type: pvc
         enabled: false
      -  # storageClassName: default
      +  storageClassName: openebs-jiva-csi-default
         accessModes:
           - ReadWriteOnce
      -  size: 10Gi
      +  size: 5Gi
         # annotations: {}
         finalizers:
           - kubernetes.io/pvc-protection
      @@ -507,15 +507,14 @@
       ## Configure grafana datasources
       ## ref: http://docs.grafana.org/administration/provisioning/#datasources
       ##
      -datasources: {}
      -#  datasources.yaml:
      -#    apiVersion: 1
      -#    datasources:
      -#    - name: Prometheus
      -#      type: prometheus
      -#      url: http://prometheus-prometheus-server
      -#      access: proxy
      -#      isDefault: true
      +datasources:
      +  datasources.yaml:
      +    apiVersion: 1
      +    datasources:
      +    - name: Prometheus
      +      type: prometheus
      +      url: http://prometheus-server.monitoring.svc.cluster.local
      +      isDefault: true
       #    - name: CloudWatch
       #      type: cloudwatch
       #      access: proxy
      ```

      </details>

1. install

    ```
    helm upgrade -i grafana grafana/grafana -n monitoring -f ~/work/prometheus/grafana/grafana.yaml
    ```

      <details><summary>実行ログ</summary>

      ```
      $ helm upgrade -i grafana grafana/grafana -n monitoring -f ~/work/prometheus/grafana/grafana.yaml
      Release "grafana" does not exist. Installing it now.
      W1030 15:08:59.041312  221007 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
      W1030 15:08:59.265293  221007 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
      NAME: grafana
      LAST DEPLOYED: Sun Oct 30 15:08:54 2022
      NAMESPACE: monitoring
      STATUS: deployed
      REVISION: 1
      NOTES:
      1. Get your 'admin' user password by running:
      
         kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
      
      2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:
      
         grafana.monitoring.svc.cluster.local
      
         Get the Grafana URL to visit by running these commands in the same shell:
      NOTE: It may take a few minutes for the LoadBalancer IP to be available.
              You can watch the status of by running 'kubectl get svc --namespace monitoring -w grafana'
           export SERVICE_IP=$(kubectl get svc --namespace monitoring grafana -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
           http://$SERVICE_IP:80

      3. Login with the password from step 1 and the username: admin
      #################################################################################
      ######   WARNING: Persistence is disabled!!! You will lose your data when   #####
      ######            the Grafana pod is terminated.                            #####
      #################################################################################
      ```

      </details>

1. grafana ServiceのMetalLBで払い出されたExternal IPアドレスを確認する
    ```
    kubectl get service -n monitoring grafana -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
    ```

1. ブラウザからアクセス
    1. ログイン画面

        | <!-- --> | <!-- --> |
        |:---|:---|
        | username | `admin` |
        | `password` | grafanaインストール時の表示を参考に以下コマンドで`admin` の `password` を取得<br><br>```kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo``` |

        <img src="/monitoring/prometheus/grafana_01.png" style="width:300px;"/>
        <img src="/monitoring/prometheus/grafana_02.png" style="width:300px;"/>

    1. ログイン成功

        <img src="/monitoring/prometheus/grafana_03.png" style="width:300px;"/>

### Install `Node Exporter Full` Dashboards

#### grafanaと一緒にインストールする場合

!!! info
    - grafana helm chartではgrafana dashboard import設定が可能です。
      `.Values.dashboards` に設定が存在する場合、[configmap.yaml](https://github.com/grafana/helm-charts/blob/grafana-6.43.5/charts/grafana/templates/configmap.yaml#L74-L133) にある `download_dashboards.sh` で指定したdashboard import設定が入ります。そして [PodのinitContainers](https://github.com/grafana/helm-charts/blob/grafana-6.43.5/charts/grafana/templates/_pod.tpl#L45-L87) で `download-dashboards.sh` を実行しています。
        - https://github.com/grafana/helm-charts/blob/grafana-6.43.5/charts/grafana/README.md
        - https://github.com/grafana/helm-charts/blob/grafana-6.43.5/charts/grafana/values.yaml#L629-L658

1. valuesファイルの修正
    ```
    vim ~/work/prometheus/grafana/grafana.yaml
    ```

      <details><summary>grafana.yaml 修正後のdiff(`cadvisor-exporter` Dashboardも入れている例です)</summary>

      ```
      $ diff -u <(curl -s https://raw.githubusercontent.com/grafana/helm-charts/main/charts/grafana/values.yaml) <(cat ~/work/prometheus/grafana/grafana.yaml)

      ~ snip ~

      @@ -632,7 +632,17 @@
       ##
       ## dashboards per provider, use provider name as key.
       ##
      -dashboards: {}
      +dashboards:
      +  default:
      +    node-exporter-full:
      +      # https://grafana.com/grafana/dashboards/1860-node-exporter-full/
      +      gnetId: 1860
      +      datasource: Prometheus
      +    cadvisor-exporter:
      +      # https://grafana.com/grafana/dashboards/14282-cadvisor-exporter/
      +      gnetId: 14282
      +      datasource: Prometheus
      +
         # default:
         #   some-dashboard:
         #     json: |

      ```

      </details>

1. install
    ```
    helm upgrade -i grafana grafana/grafana -n monitoring -f ~/work/prometheus/grafana/grafana.yaml
    ```

1. `grafanaコンテナ:/var/lib/grafana/dashboards/default/` に Dashboard定義であるjsonファイルが配置済みであることを確認
    ```
    $ kubectl exec -it -c grafana -n monitoring grafana-5f4c7d46db-xtktb -- ls -l /var/lib/grafana/dashboards/default/
    total 232
    -rw-r--r--    1 grafana  472          18484 Nov  8 16:03 cadvisor-exporter.json
    -rw-r--r--    1 grafana  472         215154 Nov  8 16:03 node-exporter-full.json
    ```

#### 手動でインストールする場合

1. Dashboardsページにアクセス

    <img src="/monitoring/prometheus/grafana_04.png" style="width:300px;"/>

1. `New` -> `New Dashboard` -> `Import` を選択

    <img src="/monitoring/prometheus/grafana_05.png" style="width:300px;"/>

1. `Node Exporter Full` の Dashboard IDを入力
    - https://grafana.com/grafana/dashboards/1860-node-exporter-full/

    <img src="/monitoring/prometheus/grafana_06.png" style="width:300px;"/>

1. `Import` を実行する

    <img src="/monitoring/prometheus/grafana_07.png" style="width:300px;"/>

1. `Node Exporter Full` が表示されることを確認

    <img src="/monitoring/prometheus/grafana_08.png" style="width:300px;"/>


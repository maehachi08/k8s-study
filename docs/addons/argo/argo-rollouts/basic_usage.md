### canary

#### manifests

- 今回はServiceやIngressを指定しないためPod数の増減となります
- canary へのtraffic shift が分かりやすいように `.spec.replicas` は `5` とします
- 今回は以下のようなstepでcanary traffic への割り振りを行います

     | # | step |
     |:---|:---|
     | 0 | 20%のtrafficをcanaryに割り振る |
     | 1 | 60sec pause |
     | 2 | 80%のtrafficをcanaryに割り振る |
     | 3 | 60sec pause |
     | 4 | 100%のtrafficをcanaryに割り振る |
     | 5 | canary を stable に切り替え |

    <details><summary>argo-rollouts/nginx.yaml</summary>
    ```
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: rollout-test-canary-nginx-conf
    data:
      nginx.conf: |
        user nginx;
        worker_processes  1;
        error_log  /var/log/nginx/error.log;
        events {
          worker_connections  1024;
        }
        http {
          server {
              listen       80;
              server_name  _;
    
              location / {
                  root   html;
                  index  index.html index.htm;
              }
    
              location /nginx_status {
                  stub_status on;
                  access_log off;
                  allow 127.0.0.1;
                  deny all;
              }
    
          }
        }
    
    ---
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    metadata:
      name: rollout-test-canary-nginx
    spec:
      strategy:
        canary:
          steps:
          - setWeight: 20
          - pause: {duration: 60}
          - setWeight: 80
          - pause: {duration: 60}
      selector:
        matchLabels:
          app: nginx
      replicas: 5
      template:
        metadata:
          annotations:
            prometheus.io/scrape: 'true'
            prometheus.io/port: '9113'
          labels:
            app: nginx
        spec:
          volumes:
          - name: nginx-conf
            configMap:
              name: rollout-test-canary-nginx-conf
              items:
                - key: nginx.conf
                  path: nginx.conf
          containers:
          - name: nginx
            image: nginx:1.14.1
            ports:
            - containerPort: 80
            volumeMounts:
            - mountPath: /etc/nginx/nginx.conf
              readOnly: true
              name: nginx-conf
              subPath: nginx.conf
          - name: nginx-exporter
            image: nginx/nginx-prometheus-exporter:0.11.0
            args:
              - -nginx.scrape-uri=http://localhost/nginx_status
            ports:
              - containerPort: 9113
    ```
    </details>

#### 初回deploy

1. deploy後の起動確認
    <details><summary>list</summary>
    ```
    $ kubectl argo rollouts list rollouts
    NAME                STRATEGY   STATUS        STEP  SET-WEIGHT  READY  DESIRED  UP-TO-DATE  AVAILABLE
    rollout-test-canary-nginx  Canary     Healthy       4/4   100         5/5    5        5           5
    ```
    </details>

    <details><summary>get</summary>
    ```
    $ kubectl argo rollouts get rollout rollout-test-canary-nginx
    Name:            rollout-test-canary-nginx
    Namespace:       default
    Status:          ✔ Healthy
    Strategy:        Canary
      Step:          4/4
      SetWeight:     100
      ActualWeight:  100
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (stable)
                     nginx:1.14.1 (stable)
    Replicas:
      Desired:       5
      Current:       5
      Updated:       5
      Ready:         5
      Available:     5
    
    NAME                                            KIND        STATUS     AGE    INFO
    ⟳ rollout-test-canary-nginx                            Rollout     ✔ Healthy  2m58s
    └──# revision:1
       └──⧉ rollout-test-canary-nginx-598c9cf7c8           ReplicaSet  ✔ Healthy  2m58s  stable
          ├──□ rollout-test-canary-nginx-598c9cf7c8-47h6s  Pod         ✔ Running  2m58s  ready:2/2
          ├──□ rollout-test-canary-nginx-598c9cf7c8-dkq5v  Pod         ✔ Running  2m58s  ready:2/2
          ├──□ rollout-test-canary-nginx-598c9cf7c8-ph8pf  Pod         ✔ Running  2m58s  ready:2/2
          ├──□ rollout-test-canary-nginx-598c9cf7c8-pw8md  Pod         ✔ Running  2m58s  ready:2/2
          └──□ rollout-test-canary-nginx-598c9cf7c8-rqbr5  Pod         ✔ Running  2m58s  ready:2/2
    ```
    </details>

#### 2回目のdeploy

1. `nginx:1.14.2` へbump upしてdeploy
1. `kubectl argo rollouts get rollout rollout-test-canary-nginx -w` で状態遷移を確認
    <details><summary>0. 20%のtrafficをcanaryに割り振る</summary>
    ```
    Name:            rollout-test-canary-nginx
    Namespace:       default
    Status:          ◌ Progressing
    Message:         more replicas need to be updated
    Strategy:        Canary
      Step:          0/4
      SetWeight:     20
      ActualWeight:  0
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (canary, stable)
                     nginx:1.14.1 (stable)
                     nginx:1.14.2 (canary)
    Replicas:
      Desired:       5
      Current:       5
      Updated:       1
      Ready:         4
      Available:     4
    
    NAME                                            KIND        STATUS         AGE    INFO
    ⟳ rollout-test-canary-nginx                            Rollout     ◌ Progressing  4m57s
    ├──# revision:2
    │  └──⧉ rollout-test-canary-nginx-bc478cd89            ReplicaSet  ◌ Progressing  7s     canary
    │     └──□ rollout-test-canary-nginx-bc478cd89-w7clb   Pod         ✔ Running      6s     ready:2/2
    └──# revision:1
       └──⧉ rollout-test-canary-nginx-598c9cf7c8           ReplicaSet  ✔ Healthy      4m57s  stable
          ├──□ rollout-test-canary-nginx-598c9cf7c8-dkq5v  Pod         ✔ Running      4m57s  ready:2/2
          ├──□ rollout-test-canary-nginx-598c9cf7c8-ph8pf  Pod         ✔ Running      4m57s  ready:2/2
          ├──□ rollout-test-canary-nginx-598c9cf7c8-pw8md  Pod         ✔ Running      4m57s  ready:2/2
          └──□ rollout-test-canary-nginx-598c9cf7c8-rqbr5  Pod         ✔ Running      4m57s  ready:2/2
    ```
    </details>
    
    <details><summary>1. 60sec pause</summary>
    ```
    Name:            rollout-test-canary-nginx
    Namespace:       default
    Status:          ॥ Paused
    Message:         CanaryPauseStep
    Strategy:        Canary
      Step:          1/4
      SetWeight:     20
      ActualWeight:  20
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (canary, stable)
                     nginx:1.14.1 (stable)
                     nginx:1.14.2 (canary)
    Replicas:
      Desired:       5
      Current:       5
      Updated:       1
      Ready:         5
      Available:     5
    
    NAME                                            KIND        STATUS     AGE    INFO
    ⟳ rollout-test-canary-nginx                            Rollout     ॥ Paused   4m57s
    ├──# revision:2
    │  └──⧉ rollout-test-canary-nginx-bc478cd89            ReplicaSet  ✔ Healthy  7s     canary
    │     └──□ rollout-test-canary-nginx-bc478cd89-w7clb   Pod         ✔ Running  6s     ready:2/2
    └──# revision:1
       └──⧉ rollout-test-canary-nginx-598c9cf7c8           ReplicaSet  ✔ Healthy  4m57s  stable
          ├──□ rollout-test-canary-nginx-598c9cf7c8-dkq5v  Pod         ✔ Running  4m57s  ready:2/2
          ├──□ rollout-test-canary-nginx-598c9cf7c8-ph8pf  Pod         ✔ Running  4m57s  ready:2/2
          ├──□ rollout-test-canary-nginx-598c9cf7c8-pw8md  Pod         ✔ Running  4m57s  ready:2/2
          └──□ rollout-test-canary-nginx-598c9cf7c8-rqbr5  Pod         ✔ Running  4m57s  ready:2/2
    ```
    </details>
    
    <details><summary>2. 80%のtrafficをcanaryに割り振る</summary>
    ```
    Name:            rollout-test-canary-nginx
    Namespace:       default
    Status:          ◌ Progressing
    Message:         more replicas need to be updated
    Strategy:        Canary
      Step:          2/4
      SetWeight:     80
      ActualWeight:  25
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (canary, stable)
                     nginx:1.14.1 (stable)
                     nginx:1.14.2 (canary)
    Replicas:
      Desired:       5
      Current:       4
      Updated:       1
      Ready:         4
      Available:     4
    
    NAME                                            KIND        STATUS               AGE    INFO
    ⟳ rollout-test-canary-nginx                            Rollout     ◌ Progressing        5m57s
    ├──# revision:2
    │  └──⧉ rollout-test-canary-nginx-bc478cd89            ReplicaSet  ◌ Progressing        67s    canary
    │     ├──□ rollout-test-canary-nginx-bc478cd89-w7clb   Pod         ✔ Running            66s    ready:2/2
    │     ├──□ rollout-test-canary-nginx-bc478cd89-9dcqt   Pod         ◌ Pending            0s     ready:0/2
    │     ├──□ rollout-test-canary-nginx-bc478cd89-cgf6w   Pod         ◌ ContainerCreating  0s     ready:0/2
    │     └──□ rollout-test-canary-nginx-bc478cd89-hjlrp   Pod         ◌ Pending            0s     ready:0/2
    └──# revision:1
       └──⧉ rollout-test-canary-nginx-598c9cf7c8           ReplicaSet  ✔ Healthy            5m57s  stable
          ├──□ rollout-test-canary-nginx-598c9cf7c8-dkq5v  Pod         ✔ Running            5m57s  ready:2/2
          ├──□ rollout-test-canary-nginx-598c9cf7c8-ph8pf  Pod         ✔ Running            5m57s  ready:2/2
          ├──□ rollout-test-canary-nginx-598c9cf7c8-pw8md  Pod         ◌ Terminating        5m57s  ready:2/2
          └──□ rollout-test-canary-nginx-598c9cf7c8-rqbr5  Pod         ✔ Running            5m57s  ready:2/2
    ```
    </details>
    
    <details><summary>3. 60sec pause</summary>
    ```
    Name:            rollout-test-canary-nginx
    Namespace:       default
    Status:          ॥ Paused
    Message:         CanaryPauseStep
    Strategy:        Canary
      Step:          3/4
      SetWeight:     80
      ActualWeight:  80
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (canary, stable)
                     nginx:1.14.1 (stable)
                     nginx:1.14.2 (canary)
    Replicas:
      Desired:       5
      Current:       5
      Updated:       4
      Ready:         5
      Available:     5
    
    NAME                                            KIND        STATUS         AGE    INFO
    ⟳ rollout-test-canary-nginx                            Rollout     ॥ Paused       6m14s
    ├──# revision:2
    │  └──⧉ rollout-test-canary-nginx-bc478cd89            ReplicaSet  ✔ Healthy      84s    canary
    │     ├──□ rollout-test-canary-nginx-bc478cd89-w7clb   Pod         ✔ Running      83s    ready:2/2
    │     ├──□ rollout-test-canary-nginx-bc478cd89-9dcqt   Pod         ✔ Running      17s    ready:2/2
    │     ├──□ rollout-test-canary-nginx-bc478cd89-cgf6w   Pod         ✔ Running      17s    ready:2/2
    │     └──□ rollout-test-canary-nginx-bc478cd89-hjlrp   Pod         ✔ Running      17s    ready:2/2
    └──# revision:1
       └──⧉ rollout-test-canary-nginx-598c9cf7c8           ReplicaSet  ✔ Healthy      6m14s  stable
          ├──□ rollout-test-canary-nginx-598c9cf7c8-dkq5v  Pod         ✔ Running      6m14s  ready:2/2
          ├──□ rollout-test-canary-nginx-598c9cf7c8-ph8pf  Pod         ◌ Terminating  6m14s  ready:2/2
          └──□ rollout-test-canary-nginx-598c9cf7c8-rqbr5  Pod         ◌ Terminating  6m14s  ready:2/2
    ```
    </details>
    
    <details><summary>4. 100%のtrafficをcanaryに割り振る</summary>
    ```
    Name:            rollout-test-canary-nginx
    Namespace:       default
    Status:          ◌ Progressing
    Message:         more replicas need to be updated
    Strategy:        Canary
      Step:          4/4
      SetWeight:     100
      ActualWeight:  100
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (canary, stable)
                     nginx:1.14.1 (stable)
                     nginx:1.14.2 (canary)
    Replicas:
      Desired:       5
      Current:       5
      Updated:       4
      Ready:         5
      Available:     5
    
    NAME                                            KIND        STATUS         AGE    INFO
    ⟳ rollout-test-canary-nginx                            Rollout     ◌ Progressing  7m13s
    ├──# revision:2
    │  └──⧉ rollout-test-canary-nginx-bc478cd89            ReplicaSet  ◌ Progressing  2m23s  canary
    │     ├──□ rollout-test-canary-nginx-bc478cd89-w7clb   Pod         ✔ Running      2m22s  ready:2/2
    │     ├──□ rollout-test-canary-nginx-bc478cd89-9dcqt   Pod         ✔ Running      76s    ready:2/2
    │     ├──□ rollout-test-canary-nginx-bc478cd89-cgf6w   Pod         ✔ Running      76s    ready:2/2
    │     ├──□ rollout-test-canary-nginx-bc478cd89-hjlrp   Pod         ✔ Running      76s    ready:2/2
    │     └──□ rollout-test-canary-nginx-bc478cd89-ct9xs   Pod         ◌ Pending      0s     ready:0/2
    └──# revision:1
       └──⧉ rollout-test-canary-nginx-598c9cf7c8           ReplicaSet  • ScaledDown   7m13s  stable
          └──□ rollout-test-canary-nginx-598c9cf7c8-dkq5v  Pod         ◌ Terminating  7m13s  ready:2/2


    snip...


    Name:            rollout-test-canary-nginx
    Namespace:       default
    Status:          ◌ Progressing
    Message:         updated replicas are still becoming available
    Strategy:        Canary
      Step:          4/4
      SetWeight:     100
      ActualWeight:  100
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (canary)
                     nginx:1.14.2 (canary)
    Replicas:
      Desired:       5
      Current:       5
      Updated:       5
      Ready:         4
      Available:     4
    
    NAME                                           KIND        STATUS         AGE    INFO
    ⟳ rollout-test-canary-nginx                           Rollout     ◌ Progressing  7m22s
    ├──# revision:2
    │  └──⧉ rollout-test-canary-nginx-bc478cd89           ReplicaSet  ◌ Progressing  2m32s  canary
    │     ├──□ rollout-test-canary-nginx-bc478cd89-w7clb  Pod         ✔ Running      2m31s  ready:2/2
    │     ├──□ rollout-test-canary-nginx-bc478cd89-9dcqt  Pod         ✔ Running      85s    ready:2/2
    │     ├──□ rollout-test-canary-nginx-bc478cd89-cgf6w  Pod         ✔ Running      85s    ready:2/2
    │     ├──□ rollout-test-canary-nginx-bc478cd89-hjlrp  Pod         ✔ Running      85s    ready:2/2
    │     └──□ rollout-test-canary-nginx-bc478cd89-ct9xs  Pod         ✔ Running      9s     ready:2/2
    └──# revision:1
       └──⧉ rollout-test-canary-nginx-598c9cf7c8          ReplicaSet  • ScaledDown   7m22s  stable
    ```
    </details>
    
    <details><summary>5. canary を stable に切り替え</summary>
    ```
    Name:            rollout-test-canary-nginx
    Namespace:       default
    Status:          ✔ Healthy
    Strategy:        Canary
      Step:          4/4
      SetWeight:     100
      ActualWeight:  100
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (stable)
                     nginx:1.14.2 (stable)
    Replicas:
      Desired:       5
      Current:       5
      Updated:       5
      Ready:         5
      Available:     5
    
    NAME                                           KIND        STATUS        AGE    INFO
    ⟳ rollout-test-canary-nginx                           Rollout     ✔ Healthy     7m23s
    ├──# revision:2
    │  └──⧉ rollout-test-canary-nginx-bc478cd89           ReplicaSet  ✔ Healthy     2m33s  stable
    │     ├──□ rollout-test-canary-nginx-bc478cd89-w7clb  Pod         ✔ Running     2m32s  ready:2/2
    │     ├──□ rollout-test-canary-nginx-bc478cd89-9dcqt  Pod         ✔ Running     86s    ready:2/2
    │     ├──□ rollout-test-canary-nginx-bc478cd89-cgf6w  Pod         ✔ Running     86s    ready:2/2
    │     ├──□ rollout-test-canary-nginx-bc478cd89-hjlrp  Pod         ✔ Running     86s    ready:2/2
    │     └──□ rollout-test-canary-nginx-bc478cd89-ct9xs  Pod         ✔ Running     10s    ready:2/2
    └──# revision:1
       └──⧉ rollout-test-canary-nginx-598c9cf7c8          ReplicaSet  • ScaledDown  7m23s
    ```
    </details>

---

### blue/green

#### manifests


- Argo Rolloutの `blue/green` はServiceからtraffic routingするReplicaSetを切り替えることで実現します。
  そのため、`Service`も併せて作成します。

    <details><summary>argo-rollouts/nginx_blue-green.yaml</summary>
    ```
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: rollout-test-bluegreen-nginx-conf
    data:
      nginx.conf: |
        user nginx;
        worker_processes  1;
        error_log  /var/log/nginx/error.log;
        events {
          worker_connections  1024;
        }
        http {
          server {
              listen       80;
              server_name  _;
    
              location / {
                  root   html;
                  index  index.html index.htm;
              }
    
              location /nginx_status {
                  stub_status on;
                  access_log off;
                  allow 127.0.0.1;
                  deny all;
              }
    
          }
        }
    
    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: rollout-test-bluegreen-active
    spec:
      selector:
        app: rollout-test-bluegreen-nginx
      ports:
      - protocol: TCP
        port: 80
        targetPort: 80
    
    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: rollout-test-bluegreen-preview
    spec:
      selector:
        app: rollout-test-bluegreen-nginx
      ports:
      - protocol: TCP
        port: 80
        targetPort: 80
    
    ---
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    metadata:
      name: rollout-test-bluegreen-nginx
    spec:
      strategy:
        blueGreen:
          activeService: rollout-test-bluegreen-active
          previewService: rollout-test-bluegreen-preview
    
      selector:
        matchLabels:
          app: rollout-test-bluegreen-nginx
      replicas: 5
      template:
        metadata:
          annotations:
            prometheus.io/scrape: 'true'
            prometheus.io/port: '9113'
          labels:
            app: rollout-test-bluegreen-nginx
        spec:
          volumes:
          - name: nginx-conf
            configMap:
              name: rollout-test-bluegreen-nginx-conf
              items:
                - key: nginx.conf
                  path: nginx.conf
          containers:
          - name: nginx
            image: nginx:1.14.2
            ports:
            - containerPort: 80
            volumeMounts:
            - mountPath: /etc/nginx/nginx.conf
              readOnly: true
              name: nginx-conf
              subPath: nginx.conf
          - name: nginx-exporter
            image: nginx/nginx-prometheus-exporter:0.11.0
            args:
              - -nginx.scrape-uri=http://localhost/nginx_status
            ports:
              - containerPort: 9113
    ```
    </details>

    !!! Warning
        - `Service`よりも先に`Rollout`が作成された場合、以下のように`Degraded`となりますので注意が必要です
            - `Rollout` よりも先に `Service` を作成する
            - 同じmanifestsファイルに記載する場合、上から順にリソースを作成していくため記載順も注意すること
            - https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/
              > The resources will be created in the order they appear in the file. Therefore, it's best to specify the service first, since that will ensure the scheduler can spread the pods associated with the service as they are created by the controller(s), such as Deployment.

        ```
        $ kubectl argo rollouts get rollout rollout-test-bluegreen-nginx
        Name:            rollout-test-bluegreen-nginx
        Namespace:       default
        Status:          ✖ Degraded
        Message:         InvalidSpec: The Rollout "rollout-test-bluegreen-nginx" is invalid: spec.strategy.blueGreen.activeService: Invalid value: "rollout-test-bluegreen-active": service "rollout-test-bl
        uegreen-active" not found
        Strategy:        BlueGreen
        Replicas:
          Desired:       5
          Current:       0
          Updated:       0
          Ready:         0
          Available:     0
        ```



#### 初回deploy

1. deploy後の起動確認
    <details><summary>get</summary>
    ```
    $ kubectl argo rollouts get rollout rollout-test-bluegreen-nginx
    Name:            rollout-test-bluegreen-nginx
    Namespace:       default
    Status:          ✔ Healthy
    Strategy:        BlueGreen
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (stable, active)
                     nginx:1.14.1 (stable, active)
    Replicas:
      Desired:       5
      Current:       5
      Updated:       5
      Ready:         5
      Available:     5

    NAME                                                      KIND        STATUS     AGE  INFO
    ⟳ rollout-test-bluegreen-nginx                            Rollout     ✔ Healthy  16s
    └──# revision:1
       └──⧉ rollout-test-bluegreen-nginx-5bb9dbdb65           ReplicaSet  ✔ Healthy  16s  stable,active
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-6rpsr  Pod         ✔ Running  15s  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-9jcz9  Pod         ✔ Running  15s  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-9v7vb  Pod         ✔ Running  15s  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-gg6kv  Pod         ✔ Running  15s  ready:2/2
          └──□ rollout-test-bluegreen-nginx-5bb9dbdb65-s87hs  Pod         ✔ Running  15s  ready:2/2
    ```
    </details>

    <details><summary>Service</summary>
    ```
    $ kubectl get service
    NAME                             TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE

    snip...

    rollout-test-bluegreen-active    ClusterIP   10.32.0.33    <none>        80/TCP           46s
    rollout-test-bluegreen-preview   ClusterIP   10.32.0.253   <none>        80/TCP           46s
    ```
    </details>

#### 2回目のdeploy

1. `nginx:1.14.2` へbump upしてdeploy
1. `kubectl argo rollouts get rollout rollout-test-bluegreen-nginx -w` で状態遷移を確認
    <details><summary>1. 新しいReplicaSetが作成され、Podが起動する</summary>
    ```
    Name:            rollout-test-bluegreen-nginx
    Namespace:       default
    Status:          ◌ Progressing
    Message:         active service cutover pending
    Strategy:        BlueGreen
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (active, preview, stable)
                     nginx:1.14.1 (stable, active)
                     nginx:1.14.2 (preview)
    Replicas:
      Desired:       5
      Current:       10
      Updated:       5
      Ready:         5
      Available:     5

    NAME                                                      KIND        STATUS         AGE  INFO
    ⟳ rollout-test-bluegreen-nginx                            Rollout     ◌ Progressing  14m
    ├──# revision:2
    │  └──⧉ rollout-test-bluegreen-nginx-559fd99986           ReplicaSet  ◌ Progressing  13s  preview
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-4kr6t  Pod         ✔ Running      13s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-jbgnb  Pod         ✔ Running      13s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-pzpdn  Pod         ✔ Running      13s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-9p276  Pod         ✔ Running      12s  ready:2/2
    │     └──□ rollout-test-bluegreen-nginx-559fd99986-n87x2  Pod         ✔ Running      12s  ready:2/2
    └──# revision:1
       └──⧉ rollout-test-bluegreen-nginx-5bb9dbdb65           ReplicaSet  ✔ Healthy      14m  stable,active
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-6rpsr  Pod         ✔ Running      14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-9jcz9  Pod         ✔ Running      14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-9v7vb  Pod         ✔ Running      14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-gg6kv  Pod         ✔ Running      14m  ready:2/2
          └──□ rollout-test-bluegreen-nginx-5bb9dbdb65-s87hs  Pod         ✔ Running      14m  ready:2/2
    ```
    </details>

    <details><summary>2. active serviceが新しいReplicaSetに向き、古いReplicaSetがscale downするまでscaleDownDelaySeconds` sec(default: 30) delay</summary>
    ```
    Name:            rollout-test-bluegreen-nginx
    Namespace:       default
    Status:          ✔ Healthy
    Strategy:        BlueGreen
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (active, stable)
                     nginx:1.14.1
                     nginx:1.14.2 (stable, active)
    Replicas:
      Desired:       5
      Current:       10
      Updated:       5
      Ready:         5
      Available:     5
    
    NAME                                                      KIND        STATUS     AGE  INFO
    ⟳ rollout-test-bluegreen-nginx                            Rollout     ✔ Healthy  14m
    ├──# revision:2
    │  └──⧉ rollout-test-bluegreen-nginx-559fd99986           ReplicaSet  ✔ Healthy  14s  stable,active
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-4kr6t  Pod         ✔ Running  14s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-jbgnb  Pod         ✔ Running  14s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-pzpdn  Pod         ✔ Running  14s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-9p276  Pod         ✔ Running  13s  ready:2/2
    │     └──□ rollout-test-bluegreen-nginx-559fd99986-n87x2  Pod         ✔ Running  13s  ready:2/2
    └──# revision:1
       └──⧉ rollout-test-bluegreen-nginx-5bb9dbdb65           ReplicaSet  ✔ Healthy  14m  delay:28s
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-6rpsr  Pod         ✔ Running  14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-9jcz9  Pod         ✔ Running  14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-9v7vb  Pod         ✔ Running  14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-gg6kv  Pod         ✔ Running  14m  ready:2/2
          └──□ rollout-test-bluegreen-nginx-5bb9dbdb65-s87hs  Pod         ✔ Running  14m  ready:2/2
    
    
    Name:            rollout-test-bluegreen-nginx
    Namespace:       default
    Status:          ✔ Healthy
    Strategy:        BlueGreen
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (active, stable)
                     nginx:1.14.1
                     nginx:1.14.2 (stable, active)
    Replicas:
      Desired:       5
      Current:       10
      Updated:       5
      Ready:         5
      Available:     5
    
    NAME                                                      KIND        STATUS     AGE  INFO
    ⟳ rollout-test-bluegreen-nginx                            Rollout     ✔ Healthy  14m
    ├──# revision:2
    │  └──⧉ rollout-test-bluegreen-nginx-559fd99986           ReplicaSet  ✔ Healthy  42s  stable,active
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-4kr6t  Pod         ✔ Running  42s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-jbgnb  Pod         ✔ Running  42s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-pzpdn  Pod         ✔ Running  42s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-9p276  Pod         ✔ Running  41s  ready:2/2
    │     └──□ rollout-test-bluegreen-nginx-559fd99986-n87x2  Pod         ✔ Running  41s  ready:2/2
    └──# revision:1
       └──⧉ rollout-test-bluegreen-nginx-5bb9dbdb65           ReplicaSet  ✔ Healthy  14m  delay:0s
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-6rpsr  Pod         ✔ Running  14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-9jcz9  Pod         ✔ Running  14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-9v7vb  Pod         ✔ Running  14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-gg6kv  Pod         ✔ Running  14m  ready:2/2
          └──□ rollout-test-bluegreen-nginx-5bb9dbdb65-s87hs  Pod         ✔ Running  14m  ready:2/2
    ```
    </details>


    <details><summary>4. `scaleDownDelaySeconds` secが経過</summary>
    ```
    Name:            rollout-test-bluegreen-nginx
    Namespace:       default
    Status:          ✔ Healthy
    Strategy:        BlueGreen
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (active, stable)
                     nginx:1.14.1
                     nginx:1.14.2 (stable, active)
    Replicas:
      Desired:       5
      Current:       10
      Updated:       5
      Ready:         5
      Available:     5
    
    NAME                                                      KIND        STATUS     AGE  INFO
    ⟳ rollout-test-bluegreen-nginx                            Rollout     ✔ Healthy  14m
    ├──# revision:2
    │  └──⧉ rollout-test-bluegreen-nginx-559fd99986           ReplicaSet  ✔ Healthy  43s  stable,active
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-4kr6t  Pod         ✔ Running  43s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-jbgnb  Pod         ✔ Running  43s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-pzpdn  Pod         ✔ Running  43s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-9p276  Pod         ✔ Running  42s  ready:2/2
    │     └──□ rollout-test-bluegreen-nginx-559fd99986-n87x2  Pod         ✔ Running  42s  ready:2/2
    └──# revision:1
       └──⧉ rollout-test-bluegreen-nginx-5bb9dbdb65           ReplicaSet  ✔ Healthy  14m  delay:passed
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-6rpsr  Pod         ✔ Running  14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-9jcz9  Pod         ✔ Running  14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-9v7vb  Pod         ✔ Running  14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-gg6kv  Pod         ✔ Running  14m  ready:2/2
          └──□ rollout-test-bluegreen-nginx-5bb9dbdb65-s87hs  Pod         ✔ Running  14m  ready:2/2
    ```
    </details>


    <details><summary>5. 古いReplicaSetがscale downする</summary>
    ```
    Name:            rollout-test-bluegreen-nginx
    Namespace:       default
    Status:          ✔ Healthy
    Strategy:        BlueGreen
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (stable, active)
                     nginx:1.14.2 (stable, active)
    Replicas:
      Desired:       5
      Current:       10
      Updated:       5
      Ready:         5
      Available:     5
    
    NAME                                                      KIND        STATUS         AGE  INFO
    ⟳ rollout-test-bluegreen-nginx                            Rollout     ✔ Healthy      14m
    ├──# revision:2
    │  └──⧉ rollout-test-bluegreen-nginx-559fd99986           ReplicaSet  ✔ Healthy      43s  stable,active
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-4kr6t  Pod         ✔ Running      43s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-jbgnb  Pod         ✔ Running      43s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-pzpdn  Pod         ✔ Running      43s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-9p276  Pod         ✔ Running      42s  ready:2/2
    │     └──□ rollout-test-bluegreen-nginx-559fd99986-n87x2  Pod         ✔ Running      42s  ready:2/2
    └──# revision:1
       └──⧉ rollout-test-bluegreen-nginx-5bb9dbdb65           ReplicaSet  • ScaledDown   14m
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-6rpsr  Pod         ◌ Terminating  14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-9jcz9  Pod         ◌ Terminating  14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-9v7vb  Pod         ◌ Terminating  14m  ready:2/2
          ├──□ rollout-test-bluegreen-nginx-5bb9dbdb65-gg6kv  Pod         ◌ Terminating  14m  ready:2/2
          └──□ rollout-test-bluegreen-nginx-5bb9dbdb65-s87hs  Pod         ◌ Terminating  14m  ready:2/2
    ```
    </details>

    <details><summary>6. 古いReplicaSetのscale downが完了</summary>
    ```
    Name:            rollout-test-bluegreen-nginx
    Namespace:       default
    Status:          ✔ Healthy
    Strategy:        BlueGreen
    Images:          nginx/nginx-prometheus-exporter:0.11.0 (stable, active)
                     nginx:1.14.2 (stable, active)
    Replicas:
      Desired:       5
      Current:       5
      Updated:       5
      Ready:         5
      Available:     5
    
    NAME                                                      KIND        STATUS        AGE  INFO
    ⟳ rollout-test-bluegreen-nginx                            Rollout     ✔ Healthy     14m
    ├──# revision:2
    │  └──⧉ rollout-test-bluegreen-nginx-559fd99986           ReplicaSet  ✔ Healthy     52s  stable,active
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-4kr6t  Pod         ✔ Running     52s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-jbgnb  Pod         ✔ Running     52s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-pzpdn  Pod         ✔ Running     52s  ready:2/2
    │     ├──□ rollout-test-bluegreen-nginx-559fd99986-9p276  Pod         ✔ Running     51s  ready:2/2
    │     └──□ rollout-test-bluegreen-nginx-559fd99986-n87x2  Pod         ✔ Running     51s  ready:2/2
    └──# revision:1
       └──⧉ rollout-test-bluegreen-nginx-5bb9dbdb65           ReplicaSet  • ScaledDown  14m
    ```
    </details>


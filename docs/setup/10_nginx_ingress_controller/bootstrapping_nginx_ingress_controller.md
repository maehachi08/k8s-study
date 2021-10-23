# bootstrapping nginx ingress controller

## 参考

- https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/
- https://kubernetes.github.io/ingress-nginx/
- https://github.com/nginxinc/kubernetes-ingress

## 手順

### 構築

- https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal
    ```
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.0/deploy/static/provider/baremetal/deploy.yaml
    ```

### 動作確認

#### nginx IngressClass が作成されていることを確認

```
$ kubectl describe ingressclasses nginx
Name:         nginx
Labels:       app.kubernetes.io/component=controller
              app.kubernetes.io/instance=ingress-nginx
              app.kubernetes.io/managed-by=Helm
              app.kubernetes.io/name=ingress-nginx
              app.kubernetes.io/version=1.0.0
              helm.sh/chart=ingress-nginx-4.0.1
Annotations:  <none>
Controller:   k8s.io/ingress-nginx
Events:       <none>
```

#### Ingressを作成してworker nodeのIPアドレスでアクセス可能であることを確認

1. manifestsファイル作成

    <details><summary>/etc/kubernetes/manifests/04_nginx_ingress_controller.yaml</summary>
        ```
        sudo tee /etc/kubernetes/manifests/04_nginx_ingress_controller.yaml << EOF > /dev/null
        ---
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: nginx-test-deployment
        spec:
          selector:
            matchLabels:
              app: nginx
          replicas: 1
          template:
            metadata:
              labels:
                app: nginx
            spec:
              containers:
              - name: nginx
                image: nginx:1.14.2
                ports:
                - containerPort: 80
        ---
        apiVersion: v1
        kind: Service
        metadata:
          name: nginx-service
        spec:
          type: NodePort
          ports:
            - name: node-port
              protocol: TCP
              port: 8080
              targetPort: 80
              nodePort: 30011
          selector:
            app: nginx
        ---
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          name: nginx-test-ingress
          annotations:
            external-dns.alpha.kubernetes.io/hostname: <Route53にA recordとして登録したいFQDN>
        spec:
          ingressClassName: nginx
          defaultBackend:
            service:
              name: nginx-service
              port:
                number: 8080
          rules:
            - http:
              paths:
                - path: /
                  pathType: Prefix
                  backend:
                    service:
                      name: nginx-service
                      port:
                        number: 8080
            # external-dns.alpha.kubernetes.io/hostname annotationsを指定せずrule毎に設定する例
            # - host: <Route53にA recordとして登録したいFQDN>
            #   http:
            #     paths:
            #       - path: /
            #         pathType: Prefix
            #         backend:
            #           service:
            #             name: nginx-service
            #             port:
            #               number: 8080
        EOF
        ```
    </details>

1. `<Route53にA recordとして登録したいFQDN>` の箇所を修正する
    ```
    sudo vim /etc.kubernetes/manifests/04_nginx_ingress_controller.yaml
    ```

1. リソース作成
    ```
    kubectl apply -f /etc.kubernetes/manifests/04_nginx_ingress_controller.yaml
    ```

1. serviceでnode port(30011/TCP) で公開していることを確認
    ```
    $ kubectl describe services nginx-service
    Name:                     nginx-service
    Namespace:                default
    Labels:                   <none>
    Annotations:              <none>
    Selector:                 app=nginx
    Type:                     NodePort
    IP Family Policy:         SingleStack
    IP Families:              IPv4
    IP:                       10.32.0.145
    IPs:                      10.32.0.145
    Port:                     node-port  8080/TCP
    TargetPort:               80/TCP
    NodePort:                 node-port  30011/TCP
    Endpoints:                10.200.1.184:80
    Session Affinity:         None
    External Traffic Policy:  Cluster
    Events:                   <none>
    ```


1. ingressでworker nodeのIPアドレス(wlan0: `192.168.10.51`) で公開していることを確認
    ```
    $ kubectl describe ingresses nginx-test-ingress
    Name:             nginx-test-ingress
    Namespace:        default
    Address:          192.168.10.51
    Default backend:  nginx-service:8080 (10.200.1.184:80)
    Rules:
      Host        Path  Backends
      ----        ----  --------
      *
                  /   nginx-service:8080 (10.200.1.184:80)
    Annotations:  <none>
    Events:
      Type    Reason  Age                  From                      Message
      ----    ------  ----                 ----                      -------
      Normal  Sync    5m39s (x7 over 25h)  nginx-ingress-controller  Scheduled for sync
    ```

1. MacBookProのターミナルでcurlにてアクセス可能であることを確認
    ```
    $ curl --include http://192.168.10.51:30011/
    HTTP/1.1 200 OK
    Date: Tue, 21 Sep 2021 16:35:07 GMT
    Content-Type: text/html
    Content-Length: 612
    Connection: keep-alive
    Last-Modified: Tue, 04 Dec 2018 14:44:49 GMT
    ETag: "5c0692e1-264"
    Accept-Ranges: bytes
    
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
        body {
            width: 35em;
            margin: 0 auto;
            font-family: Tahoma, Verdana, Arial, sans-serif;
        }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>
    
    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>
    
    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
    ```

## エラー事例


- `Error when getting IngressClass nginx: the server could not find the requested resource`
    - https://github.com/nginxinc/kubernetes-ingress/issues/1906
    - https://qiita.com/smallpalace/items/7a6844651d1d7b43b411
        - https://github.com/kubernetes/ingress-nginx/issues/7448
        - https://github.com/kubernetes/ingress-nginx/blob/3c0bfc1ca3eb48246b12e77d40bde1162633efae/deploy/static/provider/baremetal/deploy.yaml


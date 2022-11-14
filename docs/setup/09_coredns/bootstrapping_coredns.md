# bootstrapping CoreDNS

## CoreDNS とは

[CoreDNS](https://coredns.io/) は[CNCFのgraduated project](https://www.cncf.io/announcements/2019/01/24/coredns-graduation/) としてホストされているDNSサーバで、[Kubernetes 1.13 以降にてデフォルトDNSサーバとして採用](https://kubernetes.io/blog/2018/12/03/kubernetes-1-13-release-announcement/#coredns-is-now-the-default-dns-server-for-kubernetes) されており、Cluster内でのServiceリソースの名前解決に利用しています ([Kubernetes DNS-Based Service Discovery](https://github.com/kubernetes/dns/blob/master/docs/specification.md))。[AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/deploy/installation/)などCluster外へエンドポイントを公開する場合は別途外部DNSサーバ(Route53など)を利用します(CoreDNSをCluster外部に構築することも可能)。

CoreDNSはあらゆる処理をPluginとして実装しています。CoreDNS単体はDNSクエリーを解釈して設定ファイル(`./Corefile`)に記述されたPluginに処理を受け渡します。([参考](https://coredns.io/manual/toc/#plugins))

[kubernetes plugin](https://coredns.io/plugins/kubernetes/) は[Kubernetes DNS-Based Service Discovery Specification.](https://coredns.io/plugins/kubernetes/)の実装です。
`Corefile` でkubernetes plugin設定を記述して利用します。以下設定は [kubernetes/coredns.yaml.sed](https://github.com/coredns/deployment/blob/master/kubernetes/coredns.yaml.sed) の内容です。`fallthrough` とはNXDOMAIN(不在応答)が返ってきた場合に処理を下流のPluginに渡してくれます(この設定では逆引きでNXDOMAINが返ってきた場合)。

  ```
  .:53 {

    kubernetes cluster.local in-addr.arpa ip6.arpa {
      fallthrough in-addr.arpa ip6.arpa
    }

  }
  ```

### 参考文献

- https://coredns.io/
- https://github.com/coredns/coredns
- https://github.com/coredns/deployment
- https://github.com/coredns/helm
- https://engineer.retty.me/entry/2020/12/15/161544
- https://www.netone.co.jp/knowledge-center/netone-blog/20191226-1/
- https://www.scsk.jp/sp/sysdig/blog/container_monitoring/coredns.html

## 手順

1. coredns k8s manifestsを[公式](https://github.com/coredns/deployment)から取得する
    ```
    git clone https://github.com/coredns/deployment.git coredns_deployment
    cd coredns_deployment/kubernetes/

    bash deploy.sh -i 10.32.0.10  -s -t coredns.yaml.sed | kubectl apply -f -
    ```

## CoreDNSのメトリックス取得について

- `Corefile` で以下設定を記述しておくことでPrometheus用に `9153/TCP` でメトリックスを公開できます
   ```
   .:53 {

     prometheus :9153

   }
   ```


## エラー事例

1. `kubectl get pods -n kube-system` でいつまで経っても`ContainerCreating` のまま
    - `kubectl describe pod -n kube-system <POD_ID>`
        ```
        Failed to create pod sandbox: rpc error: code = Unknown desc = [failed to set up sandbox container "<CONTAINER_ID>" network for pod "<POD_ID>": networkPlugin cni failed to set up pod "<<POD_NAME>" network: failed to Statfs "/proc/15875/ns/net": no such file or directory, failed to clean up sandbox container "<CONTAINER_ID>" network for pod "<POD_ID>": networkPlugin cni failed to teardown pod "<POD_NAME>" network: neither iptables nor ip6tables usable]
        ```
    - controller-manager
        - https://github.com/kubernetes/kubernetes/blob/v1.20.2/pkg/controller/endpointslice/utils.go#L407-L415
            ```
            couldn't find ipfamilies for headless service: kube-system/kube-dns. This could happen if controller manager is connected to an old apiserver that does not support ip families yet. EndpointSlices for this Service will use IPv4 as the IP Family based on familyOf(ClusterIP:10.32.0.10).
            ```
            ```
            $ kubectl get service -n kube-system -o jsonpath='{.items[*].spec.clusterIP}'
            10.32.0.10
            ```

1. cni0(flannel)の起動に失敗している可能性がある
    ```
    failed to set bridge addr: "cni0" already has an IP address different from 10.200.1.1/24
    ```

1. 名前解決に失敗している可能性がある
    - https://coredns.io/plugins/loop/#troubleshooting
        ```
        [FATAL] plugin/loop: Loop (127.0.0.1:36286 -> :53) detected for zone ".", see https://coredns.io/plugins/loop#troubleshooting. Query: "HINFO 1048258276942848743.906062863108256161."
        ```

1. `i/o timeout`

    ```
    [ERROR] plugin/errors: 2 1233258971421873826.4416823678189275919. HINFO: read udp 10.200.0.5:35249->8.8.4.4:53: i/o timeout
    ```

      - Podのコンテナから名前解決ができない可能性がある
        - `/etc/resolv.conf` の設定が正しいか確認する

          ```
          kubectl run nginx --image=nginx
          POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}")
          kubectl exec -it $POD_NAME -- bash

          cat /etc/resolv.conf

          apt-get update              # 外にすら出れない場合は失敗する
          apt-get install dnsutils
          nslookup kubernetes
          ```

      - kube-apiserver への疎通ができていない可能性がある
        - kube-proxy -> flannel の順でpodの再起動を行ってみる


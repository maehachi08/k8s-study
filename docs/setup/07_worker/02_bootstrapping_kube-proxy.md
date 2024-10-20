# bootstrapping kube-proxy

## kube-proxy とは

[kube-proxy](https://kubernetes.io/docs/concepts/overview/components/#kube-proxy) とは各worker nodeで動作するネットワークプロキシを実現するコンポーネントです。具体的には[Sertvice](https://kubernetes.io/docs/concepts/services-networking/service/)リソースで作成されるCluster IPやNode Portの管理とそのルーティングテーブルの管理、またnginx ingress controllerを利用したIngressリソースではPodへの負荷分散にkube-proxyを活用指定たりするそうです([`With NGINX, we’ll use the DNS name or virtual IP address to identify the service, and rely on kube-proxy to perform the internal load-balancing across the pool of pods.`](https://github.com/nginxinc/kubernetes-ingress/blob/v1.12.1/examples/tcp-udp/README.md))

## 手順

1. kube-proxy-985dw
    ```
    sudo cp kube-proxy.kubeconfig /var/lib/kubernetes/
    ```
1. `Dockerfile_kube-proxy.armhf` を作成する
  <details><summary>Dockerfile_kube-proxy.armhf</summary>
    ```
    cat << 'EOF' > Dockerfile_kube-proxy.armhf
    FROM arm64v8/ubuntu:bionic

    ARG VERSION="v1.31.1"
    ARG ARCH="arm64"

    RUN set -ex \
      && apt update \
      && apt install -y wget \
      && apt clean \
      && wget -P /usr/bin/ https://dl.k8s.io/$VERSION/bin/linux/$ARCH/kube-proxy \
      && chmod +x /usr/bin/kube-proxy

    ENTRYPOINT ["/usr/bin/kube-proxy"]
    EOF
    ```
  </details>

1. image build
   ```
   sudo nerdctl build --namespace k8s.io -f Dockerfile_kube-proxy.armhf -t k8s-kube-proxy:v1.31.1 ./
   ```

1. kernel parameter
   ```
   cat <<EOF | sudo tee /etc/sysctl.d/kubelet.conf
   # kube-proxy
   net.ipv4.conf.all.route_localnet = 1
   net.netfilter.nf_conntrack_max = 131072
   net.netfilter.nf_conntrack_tcp_timeout_established = 86400
   net.netfilter.nf_conntrack_tcp_timeout_close_wait = 3600
   EOF

   sudo sysctl --system

   cat <<EOF | sudo tee /etc/modprobe.d/nf_conntrack.conf
   options nf_conntrack hashsize=32768
   EOF

   sudo /sbin/modprobe nf_conntrack hashsize=32768
   ```

1. pod manifestsを `/etc/kubernetes/manifests/` へ作成する
  <details><summary>/etc/kubernetes/manifests/kube-proxy.yaml</summary>
    ```
    cluster_cidr="10.200.0.0/16"
    sudo mkdir -p /etc/kubernetes/manifests

    cat << EOF | sudo tee /etc/kubernetes/manifests/kube-proxy.yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      labels:
        app: kube-proxy
      name: kube-proxy-configuration
      namespace: kube-system
    data:
      kube-proxy-config.yaml: |-
        ---
        apiVersion: kubeproxy.config.k8s.io/v1alpha1
        kind: KubeProxyConfiguration
        clientConnection:
          kubeconfig: "/var/lib/kubernetes/kube-proxy.kubeconfig"
        mode: "iptables"
        clusterCIDR: "${cluster_cidr}"

        # https://kubernetes.io/docs/reference/config-api/kube-proxy-config.v1alpha1/
        # metricsBindAddress: 127.0.0.1:10249
        metricsBindAddress: 0.0.0.0:10249
    ---
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: kube-proxy
      namespace: kube-system
      labels:
        component: kube-proxy
        # TODO
        # master nodeにaddon-managerを導入したらコメント外す
        # addonmanager.kubernetes.io/mode=Reconcile
    spec:
      selector:
        matchLabels:
          name: kube-proxy
      # https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/#performing-a-rolling-update
      updateStrategy:
        type: RollingUpdate
        rollingUpdate:
          maxUnavailable: 1
      template:
        # template 以下はpod templates
        #   (apiVersionやkindをもたないことを除いては、Podのテンプレートと同じスキーマ)
        #   https://kubernetes.io/ja/docs/concepts/workloads/controllers/daemonset/
        metadata:
          labels:
            name: kube-proxy
        spec:
          # https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
          priorityClassName: system-node-critical
          hostNetwork: true
          dnsPolicy: ClusterFirstWithHostNet
          containers:
            - name: kube-proxy
              image: k8s-kube-proxy:v1.31.1
              securityContext:
                capabilities:
                  add:
                    - SYS_ADMIN
                    - NET_ADMIN
                    - NET_RAW
              command:
                - /usr/bin/kube-proxy
                - --config=/var/lib/kube-proxy/kube-proxy-config.yaml
              imagePullPolicy: IfNotPresent
              resources:
                requests:
                  cpu: "256m"
              volumeMounts:
              - mountPath: /var/lib/kubernetes
                name: kubernetes-dir
              - mountPath: /etc/kubernetes/config
                name: kubernetes-config-dir
              - name: kube-proxy-configuration
                mountPath: /var/lib/kube-proxy
              - name: conntrack-command
                mountPath: /usr/sbin/conntrack
              - name: iptables-command
                mountPath: /usr/sbin/iptables
              - name: iptables-restore-command
                mountPath: /usr/sbin/iptables-restore
              - name: iptables-save-command
                mountPath: /usr/sbin/iptables-save
              - name: xtables-lock-file
                mountPath: /run/xtables.lock
              - name: usr-lib-dir
                mountPath: /usr/lib
              - name: lib-dir
                mountPath: /lib
              - name: sys-dir
                mountPath: /sys
          volumes:
          - name: kubernetes-dir
            hostPath:
              path: /var/lib/kubernetes
              type: Directory
          - name: kubernetes-config-dir
            hostPath:
              path: /etc/kubernetes/config
              type: Directory
          - name: kube-proxy-configuration
            configMap:
              name: kube-proxy-configuration
          - name: conntrack-command
            hostPath:
              path: /usr/sbin/conntrack
          - name: iptables-command
            hostPath:
              path: /usr/sbin/iptables
          - name: iptables-restore-command
            hostPath:
              path: /usr/sbin/iptables-restore
          - name: iptables-save-command
            hostPath:
              path: /usr/sbin/iptables-save
          - name: xtables-lock-file
            hostPath:
              path: /run/xtables.lock
          - name: usr-lib-dir
            hostPath:
              path: /usr/lib
          - name: lib-dir
            hostPath:
              path: /lib
          - name: sys-dir
            hostPath:
              path: /sys
    EOF
    ```
  </details>

1. podをデプロイする
   ```
   kubectl apply -f /etc/kubernetes/manifests/kube-proxy.yaml
   ```

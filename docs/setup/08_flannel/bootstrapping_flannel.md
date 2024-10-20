# bootstrapping flannel

## 参考文献

- https://github.com/flannel-io/flannel/
- https://github.com/flannel-io/flannel/blob/master/Documentation/troubleshooting.md
- `/opt/bin/flanneld` の起動オプション
   - https://github.com/flannel-io/flannel/blob/master/main.go#L110-L132

## 手順

1. vxlan moduleをインストールする
    ```
    sudo apt install -y linux-modules-extra-raspi locate
    sudo updatedb
    sudo modprobe vxlan
    sudo lsmod | grep vxlan
    ```

1. flannel k8s manifestsを公式から取得する
   - masterブランチから取得していますが2024/10/16 時点では release tag `v0.25.7` の内容

      ```
      sudo mkdir -p /etc/kubernetes/manifests
      sudo curl -o /etc/kubernetes/manifests/kube-flannel.yml -sSL https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
      ```

1. manifestsを修正する
    - `net-conf.json` の `Network` をcontroller-managerの`--cluster-cidr`で指定した値に変更する
    - etcdに用いたcertificateやadminのkubeconfigをkube-flannelコンテナへbind mountする

       ```
               volumeMounts:
               - name: var-lib-kubernetes-dir
                 mountPath: /var/lib/kubernetes/

             volumes:
             - name: var-lib-kubernetes-dir
               hostPath:
                 path: /var/lib/kubernetes
       ```

    - `/opt/bin/flanneld` の起動オプション
        - `--kubeconfig-file`
        - `--etcd-endpoints`
        - `--etcd-prefix`
        - `--etcd-keyfile`
        - `--etcd-certfile`
        - `--etcd-cafile`
        - `--v`

            ```
            sudo vim /etc/kubernetes/manifests/kube-flannel.yml
            ```
            <details><summary>修正後の`/etc/kubernetes/manifests/kube-flannel.yml`</summary>
               ```
               ---
               apiVersion: policy/v1beta1
               kind: PodSecurityPolicy
               metadata:
                 name: psp.flannel.unprivileged
                 annotations:
                   seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
                   seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
                   apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
                   apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
               spec:
                 privileged: false
                 volumes:
                 - configMap
                 - secret
                 - emptyDir
                 - hostPath
                 allowedHostPaths:
                 - pathPrefix: "/etc/cni/net.d"
                 - pathPrefix: "/etc/kube-flannel"
                 - pathPrefix: "/run/flannel"
                 readOnlyRootFilesystem: false
                 # Users and groups
                 runAsUser:
                   rule: RunAsAny
                 supplementalGroups:
                   rule: RunAsAny
                 fsGroup:
                   rule: RunAsAny
                 # Privilege Escalation
                 allowPrivilegeEscalation: false
                 defaultAllowPrivilegeEscalation: false
                 # Capabilities
                 allowedCapabilities: ['NET_ADMIN', 'NET_RAW']
                 defaultAddCapabilities: []
                 required
                 seLinux:
                   # SELinux is unused in CaaSP
                   rule: 'RunAsAny'
               ---
               kind: ClusterRole
               apiVersion: rbac.authorization.k8s.io/v1
               metadata:
                 name: flannel
               rules:
               - apiGroups: ['extensions']
                 resources: ['podsecuritypolicies']
                 verbs: ['use']
                 resourceNames: ['psp.flannel.unprivileged']
               - apiGroups:
                 - ""
                 resources:
                 - pods
                 verbs:
                 - get
               - apiGroups:
                 - ""
                 resources:
                 - nodes
                 verbs:
                 - list
                 - watch
               - apiGroups:
                 - ""
                 resources:
                 - nodes/status
                 verbs:
                 - patch
               ---
               kind: ClusterRoleBinding
               apiVersion: rbac.authorization.k8s.io/v1
               metadata:
                 name: flannel
               roleRef:
                 apiGroup: rbac.authorization.k8s.io
                 kind: ClusterRole
                 name: flannel
               subjects:
               - kind: ServiceAccount
                 name: flannel
                 namespace: kube-system
               ---
               apiVersion: v1
               kind: ServiceAccount
               metadata:
                 name: flannel
                 namespace: kube-system
               ---
               kind: ConfigMap
               apiVersion: v1
               metadata:
                 name: kube-flannel-cfg
                 namespace: kube-system
                 labels:
                   tier: node
                   app: flannel
               data:
                 cni-conf.json: |
                   {
                     "name": "cbr0",
                     "cniVersion": "0.3.1",
                     "plugins": [
                       {
                         "type": "flannel",
                         "delegate": {
                           "hairpinMode": true,
                           "isDefaultGateway": true
                         }
                       },
                       {
                         "type": "portmap",
                         "capabilities": {
                           "portMappings": true
                         }
                       }
                     ]
                   }
                 net-conf.json: |
                   {
                     "Network": "10.200.0.0/16",
                     "Backend": {
                       "Type": "vxlan"
                     }
                   }
               ---
               apiVersion: apps/v1
               kind: DaemonSet
               metadata:
                 name: kube-flannel-ds
                 namespace: kube-system
                 labels:
                   tier: node
                   app: flannel
               spec:
                 selector:
                   matchLabels:
                     app: flannel
                 template:
                   metadata:
                     labels:
                       tier: node
                       app: flannel
                   spec:
                     affinity:
                       nodeAffinity:
                         requiredDuringSchedulingIgnoredDuringExecution:
                           nodeSelectorTerms:
                           - matchExpressions:
                             - key: kubernetes.io/os
                               operator: In
                               values:
                               - linux
                     hostNetwork: true
                     priorityClassName: system-node-critical
                     tolerations:
                     - operator: Exists
                       effect: NoSchedule
                     serviceAccountName: flannel
                     initContainers:
                     - name: install-cni-plugin
                      #image: flannelcni/flannel-cni-plugin:v1.0.1 for ppc64le and mips64le (dockerhub limitations may apply)
                       image: rancher/mirrored-flannelcni-flannel-cni-plugin:v1.0.1
                       command:
                       - cp
                       args:
                       - -f
                       - /flannel
                       - /opt/cni/bin/flannel
                       volumeMounts:
                       - name: cni-plugin
                         mountPath: /opt/cni/bin
                     - name: install-cni
                      #image: flannelcni/flannel:v0.16.3 for ppc64le and mips64le (dockerhub limitations may apply)
                       image: rancher/mirrored-flannelcni-flannel:v0.16.3
                       command:
                       - cp
                       args:
                       - -f
                       - /etc/kube-flannel/cni-conf.json
                       - /etc/cni/net.d/10-flannel.conflist
                       volumeMounts:
                       - name: cni
                         mountPath: /etc/cni/net.d
                       - name: flannel-cfg
                         mountPath: /etc/kube-flannel/
                     containers:
                     - name: kube-flannel
                      #image: flannelcni/flannel:v0.16.3 for ppc64le and mips64le (dockerhub limitations may apply)
                       image: rancher/mirrored-flannelcni-flannel:v0.16.3
                       command:
                       - /opt/bin/flanneld
                       args:
                       - --ip-masq
                       - --kube-subnet-mgr
                       - --kubeconfig-file=/var/lib/kubernetes/admin.kubeconfig
                       - --etcd-endpoints=https://k8s-master:4001
                       - --etcd-prefix=/coreos.com/network
                       - --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem
                       - --etcd-certfile=/var/lib/kubernetes/kubernetes.pem
                       - --etcd-cafile=/var/lib/kubernetes/ca.pem
                       - --v=10
                       resources:
                         requests:
                           cpu: "100m"
                           memory: "50Mi"
                         limits:
                           cpu: "100m"
                           memory: "50Mi"
                       securityContext:
                         privileged: false
                         capabilities:
                           add: ["NET_ADMIN", "NET_RAW"]
                       env:
                       - name: POD_NAME
                         valueFrom:
                           fieldRef:
                             fieldPath: metadata.name
                       - name: POD_NAMESPACE
                         valueFrom:
                           fieldRef:
                             fieldPath: metadata.namespace
                       volumeMounts:
                       - name: run
                         mountPath: /run/flannel
                       - name: flannel-cfg
                         mountPath: /etc/kube-flannel/
                       - name: xtables-lock
                         mountPath: /run/xtables.lock
                       - name: var-lib-kubernetes-dir
                         mountPath: /var/lib/kubernetes/
                     volumes:
                     - name: run
                       hostPath:
                         path: /run/flannel
                     - name: cni-plugin
                       hostPath:
                         path: /opt/cni/bin
                     - name: cni
                       hostPath:
                         path: /etc/cni/net.d
                     - name: flannel-cfg
                       configMap:
                         name: kube-flannel-cfg
                     - name: xtables-lock
                       hostPath:
                         path: /run/xtables.lock
                         type: FileOrCreate
                     - name: var-lib-kubernetes-dir
                       hostPath:
                         path: /var/lib/kubernetes
               ```

            </details>


1. flannel Podをデプロイ

   ```
   kubectl apply -f /etc/kubernetes/manifests/kube-flannel.yml
   ```

## エラー事例

1. flannelが参照するkubeconfigが正しくない
    - [flannel-io/flannel Documentation/kube-flannel.yml](https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml) そのままだと発生した
        ```
        Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
        error creating inClusterConfig, falling back to default config: open /var/run/secrets/kubernetes.io/serviceaccount/token: no such file or directory
        Failed to create SubnetManager: fail to create kubernetes config: invalid configuration: no configuration has been provided, try setting KUBERNETES_MASTER environment variable
        ```

1. `Failed to create SubnetManager: fail to create kubernetes config: stat "/var/lib/kubernetes/admin.kubeconfig": no such file or directory`
    - `--kubeconfig-file` オプションで指定したkubeconfig pathが正しくない
        - 私のケースでは `--kubeconfig-file="/var/lib/kubernetes/admin.kubeconfig"` としているとダブルクォート(") がパスに含まれてしまっていることに気付くのに時間かかりました(正しくは `--kubeconfig-file=/var/lib/kubernetes/admin.kubeconfig`)

1. `Error registering network: failed to acquire lease: node "k8s-node1" pod cidr not assigned`
    - Nodeリソースの `.spec.podCIDR` が登録されていない
    - 確認方法
        ```
        kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'
        ```
    - `kube-controller-manager` の設定不備の可能性
        - 以下設定変更後、kubeletの起動前に `kubectl delete node <NODE>` を実行する
            - https://github.com/flannel-io/flannel/issues/728#issuecomment-325347810
                - podCIDR割り当て設定が正しくない
                    - `--cluster-cidr=<CIDR>`
                    - `--allocate-node-cidrs=true`
            - https://blog.net.ist.i.kyoto-u.ac.jp/2019/11/06/kubernetes-%E6%97%A5%E8%A8%98-2019-11-05/
                - `--service-cluster-ip-range=<CIDR>` と (flannnelの)`--pod-cidr=<CIDR>` が被っている可能性がある

    - (kube-controller-manager が正しい場合) Nodeリソースの `.spec.podCIDR` を手動設定して回避する
        ```
        kubectl patch node <NODE_NAME> -p '{"spec":{"podCIDR":"<SUBNET>"}}'
        ```

1. `Error registering network: failed to configure interface flannel.1: failed to ensure address of interface flannel.1: link has incompatible addresses. Remove additional addresses and try again.`
    - `kubectl delete pod kube-flannel-...` で ネットワークインターフェース `flannel.1` が開放されない
        - https://github.com/flannel-io/flannel/issues/1060
        - 古い `flannel.1` を削除
            ```
            ip a
            sudo ip link delete flannel.1
            ```

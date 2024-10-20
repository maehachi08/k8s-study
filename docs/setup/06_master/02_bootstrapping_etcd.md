# bootstrapping etcd

[coreosがetcd docker image を提供](https://quay.io/repository/coreos/etcd?tab=tags) していますが、Raspberry Piに搭載されているARM CPUのアーキテクチャ `armv8(64bit)` で利用可能なimageは提供していないため、imageをbuildします。

## 手順

1. `Dockerfile_etcd.armhf` を作成する
   <details><summary>Dockerfile_etcd.armhf</summary>
      ```
      cat << 'EOF' > Dockerfile_etcd.armhf
      FROM arm64v8/ubuntu:bionic AS etcd-builder

      RUN set -ex \
        && apt update \
        && apt install -y git tar zip curl \
        && apt clean

      RUN set -ex \
        && curl -L https://go.dev/dl/go1.23.1.linux-arm64.tar.gz | tar -zxC /usr/local

      RUN set -ex \
        && git clone https://github.com/etcd-io/etcd.git -b v3.5.16 /tmp/etcd \
        && cd /tmp/etcd \
        && PATH=$PATH:/usr/local/go/bin:~/go/bin ./build



      FROM arm64v8/ubuntu:bionic

      COPY --from=etcd-builder /tmp/etcd/bin/etcd /usr/local/bin/
      COPY --from=etcd-builder /tmp/etcd/bin/etcdctl /usr/local/bin/

      RUN set -ex \
        && apt update \
        && apt upgrade -y \
        && apt clean

      EXPOSE 2379 2380

      ENTRYPOINT ["/usr/local/bin/etcd"]
      EOF
      ```
   </details>

1. image build
   ```
   sudo mkdir -p /var/lib/etcd_data
   sudo nerdctl build --namespace k8s.io -t k8s-etcd:v3.5.16 --file=Dockerfile_etcd.armhf ./
   ```

1. pod manifestsを `/etc/kubelet.d` へ作成する
   <details><summary>/etc/kubelet.d/etcd.yaml</summary>
      ```
      cat << EOF | sudo tee /etc/kubelet.d/etcd.yaml
      ---
      apiVersion: v1
      kind: Pod
      metadata:
        name: etcd-v3.5.16
        namespace: kube-system
        labels:
          tier: control-plane
          component: etcd

      spec:
        # https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/
        priorityClassName: system-node-critical
        hostNetwork: true
        volumes:
        - name: kubernetes-dir
          hostPath:
            path: /var/lib/kubernetes
            type: Directory
        - name: etcd-data-dir
          hostPath:
            path: /var/lib/etcd_data
            type: Directory
        containers:
          - name: etcd
            image: k8s-etcd:v3.5.16
            imagePullPolicy: IfNotPresent
            volumeMounts:
            - mountPath: /var/lib/kubernetes
              name: kubernetes-dir
            - mountPath: /var/lib/etcd_data
              name: etcd-data-dir
            env:
            - name: ETCD_UNSUPPORTED_ARCH
              value: "arm64"
            ports:
            - containerPort: 2379
              protocol: TCP
              targetPort: 2379
            - containerPort: 2380
              protocol: TCP
              targetPort: 2380
            resources:
              requests:
                cpu: 0.5
                memory: "384Mi"
              limits:
                cpu: 1
                memory: "384Mi"
            command:
              - /usr/local/bin/etcd
              - --data-dir=/var/lib/etcd_data
              - --advertise-client-urls=https://k8s-master:2379,https://k8s-master:2380
              - --listen-client-urls=https://0.0.0.0:2379
              - --initial-advertise-peer-urls=https://k8s-master:2380
              - --listen-peer-urls=https://0.0.0.0:2380
              - --name=etcd0
              - --cert-file=/var/lib/kubernetes/kubernetes.pem
              - --key-file=/var/lib/kubernetes/kubernetes-key.pem
              - --peer-cert-file=/var/lib/kubernetes/kubernetes.pem
              - --peer-key-file=/var/lib/kubernetes/kubernetes-key.pem
              - --trusted-ca-file=/var/lib/kubernetes/ca.pem
              - --peer-trusted-ca-file=/var/lib/kubernetes/ca.pem
              - --peer-client-cert-auth
              - --client-cert-auth
              - --initial-cluster-token=etcd-cluster-1
              - --initial-cluster=etcd0=https://k8s-master:2380
              - --initial-cluster-state=new
      EOF
      ```
   </details>

1. `crictl` でコンテナ起動を確認する
   ```
   $ sudo crictl ps --name etcd
   CONTAINER           IMAGE                                                              CREATED             STATE               NAME                ATTEMPT             POD ID
   72f58248ec087       6e8b8110dc13cfe61d75f867a22c39766a397989413570500f51dedf94be7a12   25 seconds ago       Running             etcd                0                   206c5b952097a
   ```

## 参考文献

- https://etcd.io/docs/v2/docker_guide/
- https://quay.io/repository/coreos/etcd?tag=latest&tab=tags
- https://github.com/etcd-io/etcd


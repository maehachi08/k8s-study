### swapを無効にする

1. swapが使用されていることを確認
   ```
   $ free -h
                 total        used        free      shared  buff/cache   available
   Mem:          1.8Gi        54Mi       1.5Gi       8.0Mi       244Mi       1.7Gi
   Swap:          99Mi          0B        99Mi
   ```

1. swapを無効に設定する
   ```
   sudo swapoff --all

   # Ubuntu Desktop 22.04 LTS (for RPi4)
   sudo systemctl stop dphys-swapfile
   sudo systemctl disable dphys-swapfile
   ```

1. swapが無効であることを確認する
   ```
   $ free -h
                 total        used        free      shared  buff/cache   available
   Mem:          1.8Gi        57Mi       1.5Gi       8.0Mi       251Mi       1.7Gi
   Swap:            0B          0B          0B

   $ systemctl status dphys-swapfile
   ● dphys-swapfile.service - dphys-swapfile - set up, mount/unmount, and delete a swap file
      Loaded: loaded (/lib/systemd/system/dphys-swapfile.service; disabled; vendor preset: enabled)
      Active: inactive (dead)
        Docs: man:dphys-swapfile(8)

   12月 30 20:48:54 k8s-master1 systemd[1]: Starting dphys-swapfile - set up, mount/unmount, and delete a swap file...
   12月 30 20:48:55 k8s-master1 dphys-swapfile[330]: want /var/swap=100MByte, checking existing: keeping it
   12月 30 20:48:55 k8s-master1 systemd[1]: Started dphys-swapfile - set up, mount/unmount, and delete a swap file.
   12月 31 06:57:57 k8s-master1 systemd[1]: Stopping dphys-swapfile - set up, mount/unmount, and delete a swap file...
   12月 31 06:57:57 k8s-master1 systemd[1]: dphys-swapfile.service: Succeeded.
   12月 31 06:57:57 k8s-master1 systemd[1]: Stopped dphys-swapfile - set up, mount/unmount, and delete a swap file.
   ```

### cgroupfs のmemoryを有効にする

1. kernelのboot optionに `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory` を追記する
   ```
   sudo vim /boot/firmware/cmdline.txt
   ```

     - `cmdline.txt` の有効行の確認

        ```
        $ sudo sed -e 's/\s/\n/g' /boot/firmware/cmdline.txt
        console=serial0,115200
        multipath=off
        dwc_otg.lpm_enable=0
        console=tty1
        root=LABEL=writable
        rootfstype=ext4
        rootwait
        fixrtc
        cfg80211.ieee80211_regdom=JP
        cgroup_enable=cpuset
        cgroup_memory=1
        cgroup_enable=memory

        $ sudo reboot

        $ cat /proc/cgroups
        #subsys_name    hierarchy       num_cgroups     enabled
        cpuset  9       1       1
        cpu     5       34      1
        cpuacct 5       34      1
        blkio   10      34      1
        memory  8       80      1
        devices 4       34      1
        freezer 7       1       1
        net_cls 2       1       1
        perf_event      6       1       1
        net_prio        2       1       1
        pids    3       39      1
        ```

### Container RunTimeインストール

- `containerd` を採用
  - 初期は `cri-o` を採用していたが以下理由で`containerd`へ変更
    - [CNCFでGraduated Project](https://landscape.cncf.io/?selected=containerd)([cri-oはincubating](https://www.cncf.io/projects/cri-o/))
    - AWS eks-optimized AMIではcontainerdが標準となりそう(eks 1.22 ではdockerがdefault)
    - `buildkit` と組み合わせてimage buildが可能 (cri-oでの可否は未確認)
        - https://speakerdeck.com/ktock/dockerkaracontainerdhefalseyi-xing?slide=7

### 前提作業

- https://kubernetes.io/docs/setup/production-environment/container-runtimes/
    <details>
    ```
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF

    sudo modprobe overlay
    sudo modprobe br_netfilter

    # sysctl params required by setup, params persist across reboots
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF

    # Apply sysctl params without reboot
    sudo sysctl --system
    ```
    </details>

#### Containerd

https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd
https://github.com/containerd/containerd/blob/main/docs/getting-started.md

1. インストール
    - for apt package
        - refs https://docs.docker.com/engine/install/ubuntu/
          ```
          sudo apt update
          sudo apt install -y ca-certificates curl gnupg lsb-release
          sudo mkdir -p /etc/apt/keyrings
          curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
          echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

          sudo apt update
          sudo apt install -y containerd.io
          ```
1. `/etc/containerd/config.toml`
    ```
    sudo containerd config default | sudo tee /etc/containerd/config.toml
    sudo vim /etc/containerd/config.toml
    ```
       - refs https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd
         - Configuring the systemd cgroup driver
         - Overriding the sandbox (pause) image
1. restart containerd
    ```
    sudo systemctl restart containerd
    ```

#### CRI-O

https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cri-o

1. kernel moduleのload
    - overlayファイルシステムを利用するためのkernel module `overlay`
    - iptablesがbridgeを通過するパケットを処理するためのkernel module `br_netfilter`
1. kernel parameterのset
    - iptablesがbridgeを通過するパケットを処理するための設定

1. kernel moduleのload
    - overlayファイルシステムを利用するためのkernel module `overlay`
    - iptablesがbridgeを通過するパケットを処理するためのkernel module `br_netfilter`
      ```
      cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
      overlay
      br_netfilter
      EOF

      sudo modprobe overlay
      sudo modprobe br_netfilter
      ```

1. kernel parameterのset
    - iptablesがbridgeを通過するパケットを処理するための設定
      ```
      cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf

      # https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cri-o
      net.bridge.bridge-nf-call-iptables  = 1
      net.ipv4.ip_forward                 = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      EOF

      sudo sysctl --system
      ```

1. system起動時に kernel parameter を再読み込みさせる
    - kube-proxyにて必要なkernel parameter設定(kubelet設定手順にて後述) がiptables起動時のkernel module loadで上書きされるため
    - 利用環境が `/etc/sysconfig/iptables-config` を利用可能なら `IPTABLES_MODULES_UNLOAD="no"` を設定することで本手順は不要です
        ```
        egrep  "sysctl\s+--system" /etc/rc.local > /dev/null || sudo bash -c "echo \"sysctl --system\" >> /etc/rc.local"
        egrep  "sysctl\s+--system" /etc/rc.local
        ```

1. CRI-Oインストール
    - https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/1.21/xUbuntu_20.04/arm64/
      ```
      VERSION=1.21
      OS=xUbuntu_20.04

      sudo bash -c "echo \"deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /\" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list"
      sudo bash -c "echo \"deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /\" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list"

      curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | sudo apt-key add -
      curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key add -

      sudo apt update
      sudo apt install -y cri-o cri-o-runc

      sudo apt install -y conntrack
      ```

1. storage driverを `overlay2` へ変更する
   ```
   sudo vim /etc/containers/storage.conf
   sudo vim /etc/crio/crio.conf
   ```

      -  `/etc/crio/crio.conf` へgraph driver設定を入れる
        - podmanやbuildahでbuildしたlocal imageを参照するため
        - `[crio]` セクションに入れる
           ```
           graphroot = "/var/lib/containers/storage"
           ```
    <details><summary>/etc/containers/storage.conf</summary>
    ```
    [storage]
    driver = "overlay2"
    runroot = "/run/containers/storage"
    graphroot = "/var/lib/containers/storage"

    [storage.options]
    additionalimagestores = [
    ]

    [storage.options.overlay]
    mountopt = "nodev"

    [storage.options.thinpool]
    ```
    </details>
    <details><summary>/etc/crio/crio.conf</summary>
    ```
    [crio]
    storage_driver = "overlay2"
    graphroot = "/var/lib/containers/storage"
    log_dir = "/var/log/crio/pods"
    version_file = "/var/run/crio/version"
    version_file_persist = "/var/lib/crio/version"
    clean_shutdown_file = "/var/lib/crio/clean.shutdown"

    [crio.api]
    listen = "/var/run/crio/crio.sock"
    stream_address = "127.0.0.1"
    stream_port = "0"
    stream_enable_tls = false
    stream_idle_timeout = ""
    stream_tls_cert = ""
    stream_tls_key = ""
    stream_tls_ca = ""
    grpc_max_send_msg_size = 16777216
    grpc_max_recv_msg_size = 16777216

    [crio.runtime]
    no_pivot = false
    decryption_keys_path = "/etc/crio/keys/"
    conmon = ""
    conmon_cgroup = "system.slice"
    conmon_env = [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    ]
    default_env = [
    ]
    seccomp_profile = ""
    seccomp_use_default_when_empty = false
    apparmor_profile = "crio-default"
    irqbalance_config_file = "/etc/sysconfig/irqbalance"
    cgroup_manager = "systemd"
    separate_pull_cgroup = ""
    default_capabilities = [
            "CHOWN",
            "DAC_OVERRIDE",
            "FSETID",
            "FOWNER",
            "SETGID",
            "SETUID",
            "SETPCAP",
            "NET_BIND_SERVICE",
            "KILL",
    ]
    default_sysctls = [
    ]
    additional_devices = [
    ]
    hooks_dir = [
            "/usr/share/containers/oci/hooks.d",
    ]pids_limit = 1024
    log_size_max = -1
    log_to_journald = false
    container_exits_dir = "/var/run/crio/exits"
    container_attach_socket_dir = "/var/run/crio"
    bind_mount_prefix = ""
    read_only = false
    log_level = "info"
    log_filter = ""
    uid_mappings = ""
    gid_mappings = ""
    ctr_stop_timeout = 30
    drop_infra_ctr = false
    infra_ctr_cpuset = ""
    namespaces_dir = "/var/run"
    pinns_path = ""
    default_runtime = "runc"

    [crio.runtime.runtimes.runc]
    runtime_path = ""
    runtime_type = "oci"
    runtime_root = "/run/runc"
    allowed_annotations = [
            "io.containers.trace-syscall",
    ]

    [crio.image]
    default_transport = "docker://"
    global_auth_file = ""
    pause_image = "k8s.gcr.io/pause:3.2"
    pause_image_auth_file = ""
    pause_command = "/pause"
    signature_policy = ""
    image_volumes = "mkdir"
    big_files_temporary_dir = ""

    [crio.network]
    network_dir = "/etc/cni/net.d/"
    plugin_dirs = [
            "/opt/cni/bin/",
    ]
    [crio.metrics]
    enable_metrics = false
    metrics_port = 9090
    metrics_socket = ""
    ```
    </details>

1. crioを再起動する
   ```
   sudo systemctl daemon-reload
   sudo systemctl restart crio
   ```

### CLI TOOL

1. nerdctl
    - containerdプロジェクトで公開しているdocker-cli互換のCLI
    - https://github.com/containerd/nerdctl
    - https://speakerdeck.com/ktock/dockerkaracontainerdhefalseyi-xing?slide=22
        ```
        NERDCTL_VERSION=`curl -s -L https://api.github.com/repos/containerd/nerdctl/releases/latest | jq -r .tag_name`
        curl -L -s https://github.com/containerd/nerdctl/releases/download/${NERDCTL_VERSION}/nerdctl-`echo ${NERDCTL_VERSION} | sed -e 's/^v//'`-linux-arm64.tar.gz | sudo tar -zxC /usr/local/bin/

        ls -l /usr/local/bin
        ```

1. buildkit
    - https://github.com/moby/buildkit
    - `nerdctl build` を実行するために必要
        ```
        BUILDKIT_VERSION=`curl -s -L https://api.github.com/repos/moby/buildkit/releases/latest | jq -r .tag_name`
        curl -L -s https://github.com/moby/buildkit/releases/download/${BUILDKIT_VERSION}/buildkit-${BUILDKIT_VERSION}.linux-arm64.tar.gz | sudo tar -zxC /tmp/
        sudo mv /tmp/bin/* /usr/local/bin/
        ls -l /usr/local/bin

        sudo curl -sL https://raw.githubusercontent.com/moby/buildkit/${BUILDKIT_VERSION}/examples/systemd/system/buildkit.socket -o /etc/systemd/system/buildkit.socket
        sudo curl -sL https://raw.githubusercontent.com/moby/buildkit/${BUILDKIT_VERSION}/examples/systemd/system/buildkit.service -o /etc/systemd/system/buildkit.service
        sudo systemctl enable buildkit.socket buildkit.service
        sudo systemctl start buildkit.socket buildkit.service
        ```

1. buildah 
    - cri-o 導入時期にimage buildで使用(containerdでは前述のnerdctlへ移行済み)
    - https://github.com/containers/buildah/blob/master/install.md
       ```
       sudo apt-get -qq -y install buildah
       ```

1. cri-tools(crictl) インストール
    - https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md
       ```
       VERSION="v1.31.1"
       ARCH="arm64"
       DOWNLOAD_URL="https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-${VERSION}-linux-${ARCH}.tar.gz"
       curl -L ${DOWNLOAD_URL} | sudo tar -zxC /usr/local/bin

       ls -l /usr/local/bin
       ```

1. podman インストール(optional)
    - https://podman.io/getting-started/installation
       ```
       sudo apt-get -y install podman
       sudo rm -f /etc/cni/net.d/87-podman-bridge.conflist
       ```

### CNI Pluginインストール

- https://github.com/containernetworking/plugins
   ```
   sudo mkdir -p /opt/cni/bin
   CNI_PLUGIN_VERSION=`curl -s -L https://api.github.com/repos/containernetworking/plugins/releases/latest | jq -r .tag_name`
   ARCH="arm64"
   DOWNLOAD_URL="https://github.com/containernetworking/plugins/releases/download/${CNI_PLUGIN_VERSION}/cni-plugins-linux-${ARCH}-${CNI_PLUGIN_VERSION}.tgz"
   curl -L ${DOWNLOAD_URL} | sudo tar -zxC /opt/cni/bin

   ls -l /opt/cni/bin/
   ```
- cni configを作成する
    - https://www.cni.dev/plugins/current/main/bridge/
        ```
        POD_CIDR="10.200.0.0/24"
        cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
        {
            "cniVersion": "0.4.0",
            "name": "bridge",
            "type": "bridge",
            "bridge": "cnio0",
            "isGateway": true,
            "ipMasq": true,
            "ipam": {
                "type": "host-local",
                "ranges": [
                  [{"subnet": "${POD_CIDR}"}]
                ],
                "routes": [{"dst": "0.0.0.0/0"}]
            }
        }
        EOF

        cat <<EOF | sudo tee /etc/cni/net.d/20-loopback.conf
        {
            "cniVersion": "0.4.0",
            "name": "lo",
            "type": "loopback"
        }
        EOF
        ```

### kubectl インストール

- https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
    ```
    VERSION="v1.31.1"
    curl -LO "https://dl.k8s.io/release/${VERSION}/bin/linux/arm64/kubectl"
    chmod +x kubectl
    chown root:root kubectl
    sudo mv kubectl /usr/local/bin/
    ls -l /usr/local/bin/
    ```

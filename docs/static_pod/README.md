

8月 31 12:57:50 k8s-master kubelet[1912]: E0831 12:57:50.411795    1912 kubelet.go:1635] Failed creating a mirror pod for "etcd-k8s-master_kube-system(8fff3e1f31b52ebeb520767ae50cc739)": pods "etcd-k8s-master" is forbidden: PodSecurityPolicy: no providers available to validate pod request


8月 31 12:58:55 k8s-master kubelet[1912]: I0831 12:58:55.446865    1912 kubelet.go:1885] SyncLoop (ADD, "api"): "kube-apiserver-k8s-master_kube-system(b2f9759c-9ffe-436c-9ebb-0b5f9c68a452)"
8月 31 12:58:58 k8s-master kubelet[1912]: I0831 12:58:58.433779    1912 kubelet.go:1885] SyncLoop (ADD, "api"): "kube-scheduler-k8s-master_kube-system(b502a669-2571-41f7-a705-3aa194ade526)"
8月 31 12:59:00 k8s-master kubelet[1912]: I0831 12:59:00.432199    1912 kubelet.go:1885] SyncLoop (ADD, "api"): "kube-controller-manager-k8s-master_kube-system(f834a62b-d802-40c5-90df-24a49a38efb6)"
8月 31 12:59:11 k8s-master kubelet[1912]: I0831 12:59:11.421459    1912 kubelet.go:1885] SyncLoop (ADD, "api"): "etcd-k8s-master_kube-system(e612e1ab-a907-4955-8b4c-97d5a37b11fa)"
8月 31 12:59:13 k8s-master kubelet[1912]: I0831 12:59:13.849447    1912 kubelet_getters.go:176] "Pod status updated" pod="kube-system/etcd-k8s-master" status=Running
8月 31 12:59:13 k8s-master kubelet[1912]: I0831 12:59:13.849581    1912 kubelet_getters.go:176] "Pod status updated" pod="kube-system/kube-apiserver-k8s-master" status=Running
8月 31 12:59:13 k8s-master kubelet[1912]: I0831 12:59:13.849654    1912 kubelet_getters.go:176] "Pod status updated" pod="kube-system/kube-controller-manager-k8s-master" status=Running
8月 31 12:59:13 k8s-master kubelet[1912]: I0831 12:59:13.849719    1912 kubelet_getters.go:176] "Pod status updated" pod="kube-system/kube-scheduler-k8s-master" status=Running




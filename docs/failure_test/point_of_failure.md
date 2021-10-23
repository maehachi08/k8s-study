## Point of failure

### Kubernetes Component

#### Controll Plane

- etcd
- kube-apiserver
- kube-controller-manager
- kube-scheduler

#### Worker Node

- resources(cpu, memory, disk, network...)
   - OOM Killer
- kubelet
- kube-proxy
- cni
- container-runtime

#### Addons

- 参考
    - https://github.com/kubernetes/kubernetes/tree/master/cluster/addons
    - https://registry.terraform.io/modules/particuleio/addons/kubernetes/latest
    - https://aws.amazon.com/jp/blogs/news/introducing-amazon-eks-add-ons-jp/
    - https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/eks-add-ons.html

- CoreDNS
- Amazon VPC CNI(EKS)

#### Controller

- Ingress Controller
- External-dns

#### AutoScale

- MetricsServer
- Pod
    - HPA
    - VPA

- Node(Horizontal Scale)
    - Cluster Autoscaler

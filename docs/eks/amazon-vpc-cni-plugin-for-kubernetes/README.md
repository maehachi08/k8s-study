# Amazon VPC CNI Plugin for Kubernetes

- Amazon VPC CNI(Container Networking Interface) Plugin for Kubernetes
- https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/managing-vpc-cni.html

    > Amazon VPC CNI plugin for Kubernetes アドオンは Amazon EKS クラスター内の各 Amazon EC2 ノードにデプロイされます。アドオンは Elastic Network Interface を作成し、Amazon EC2 ノードにアタッチします。またアドオンは、VPC のプライベートIPv4 または IPv6 アドレスを各 Pod およびサービスに割り当てます。

## Reference

- https://github.com/aws/amazon-vpc-cni-k8s
- https://github.com/aws/amazon-vpc-cni-k8s/tree/master/docs
- https://github.com/aws/eks-charts/tree/master/stable/aws-vpc-cni
- https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html
- https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html
- https://aws.github.io/aws-eks-best-practices/networking/vpc-cni/
- https://aws.github.io/aws-eks-best-practices/networking/custom-networking/
- https://repost.aws/ja/knowledge-center/eks-custom-subnet-for-pod
- https://repost.aws/knowledge-center/eks-multiple-cidr-ranges
- https://aws.amazon.com/jp/blogs/news/leveraging-cni-custom-networking-alongside-security-groups-for-pods-in-amazon-eks/
- https://speakerdeck.com/sshota0809/netutowakushi-dian-dexue-bu-amazon-eks-kurasutanosukerabiritei
- https://github.com/aws-ia/terraform-aws-eks-blueprints/tree/main/docs

## Install

- Amazon EKS Clusterを作成すると自動的にインストールされる
    - https://aws-ia.github.io/terraform-aws-eks-blueprints/add-ons/managed-add-ons/
    - https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/managing-vpc-cni.html
- 手動インストール
    - https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html#vpc-add-on-self-managed-update
    - https://github.com/aws/amazon-vpc-cni-k8s/tree/master/charts/aws-vpc-cni

## What you can do?

- PodにVPCネットワークと同じCIDRのIPアドレスを割り当てることができます
- PodにNodeと異なるENIをアタッチするができます(InstanceTypeにより1 Nodeあたりアタッチ可能なENI数が決まります)
- PodにNodeと異なるSecurityGroupをアタッチすることができます(v1.7.7以降)
- PodにNodeと異なるSubnetセグメントのIPアドレスを割り当てることができます
- Prefix Modeをサポート
- Kubernetes NetworkPolicyによりPod間の細かなトラフィックフロー制御ができます
    - https://aws.amazon.com/jp/blogs/news/amazon-vpc-cni-now-supports-kubernetes-network-policies/

## ENI Assign Pattern

PodにVPCネットワークと同じCIDRのIPアドレスを割り当てる時、ENIの割り当て方法にいつかのパターンが存在します。

大きくは以下3つのポイントがあります。

1. ENIをWorker Nodeと共有する (default)
1. ENIをWorker Nodeと共有せず、Pod間で共有する (Custom Networking)
1. ENIをWorker NodeともPod間でも共有せず、1 PodでENIを占有する (SecurityGroup for Pods)

Custom NetworkingやSecurityGroup for Podsは排他利用ではないのでどちらも有効にすることが可能です。
この場合(以下マトリックス表の4番)、SecurityGroupPolicyのselectorにMatchしないPodはCustom Networkingを利用するようになります。

| # | Custom Networking | SecurityGroup for Pods |
|:---|:---|:---|
| 1 | N | N |
| 2 | N | Y |
| 3 | Y | N |
| 4 | Y | Y |

### Default

デフォルトではAmazon VPC CNI PluginはWorker NodeのPrimary SubnetからIPアドレスを取得しPodに割り当てます。
Primary SubnetとはWorker NodeのPrimary ENIが接続しているSubnetを指し、Primary ENIとはWorker Nodeにアタッチされている1つ目のENI(eth0)を指します。
Podに割り当てるIPアドレスはPrimary ENIに割り当てられるSecondary IPアドレスが使用され、PodにアタッチされるSecurityGroupはWorker NodeのPrimary ENIのも>のと同じとなります。

デフォルトの動作では以下のような問題が懸念されます。

1. SubnetのIPアドレスが枯渇しPodにIPアドレスが割り当てられない
    - Worker Nodeに割り当てるIPアドレスやアプリケーションPodはもちろん、様々なWorkloadsによりIPアドレスが消費される
    - 特にAddonsなどでDaemonSet Workloadsがある場合、Worker NodeごとにPodが起動するためより多くのIPアドレスを消費することになる
1. (運用上) Worker NodeとPodでSubnetを分けたい場合
    - IPアドレスの枯渇のケースもですが、運用上Worker NodeとPodでSubnetを分けたい場合
1. (運用上) NodeとPodでSecurityGroupを分けたい場合
    - SecurityGroupはWorker Nodeやアプリケーションが稼働するPodなどでアクセス制御のsource/destが異なります。
      Worker Node SGで大きく許可することでPodのアクセス制御を兼ねることも可能ですが、Securityの観点ではよくないです

Custom Networkingはこれらの問題を解決することができます。

### Custom Networking

Custom NetworkingとはPodに割り当てるIPアドレスを取得するSubnetやPodにアタッチされるSecurityGroupをPrimary ENIと区別して指定できる機能です。

[Custom Networking](custom-networking.md)

#### Prefix Mode

[Prefix Mode](prefix-mode.md)


### Network Policy

[Network Policy](network-policy.md)

### SecurityGroup for Pods

[SecurityGroup for Pods](security_group_for_pod.md)


## References

- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://docs.aws.amazon.com/eks/latest/userguide/cni-network-policy.html
- https://github.com/aws/amazon-vpc-cni-k8s/blob/master/README.md#network-policies
- https://aws.amazon.com/jp/blogs/news/amazon-vpc-cni-now-supports-kubernetes-network-policies/
- https://aws.github.io/aws-eks-best-practices/security/docs/network/
- https://github.com/aws-samples/eks-network-policy-examples
- https://www.linkedin.com/pulse/how-enable-network-policies-eks-using-aws-vpc-cni-plugin-engin-diri

## About

- KubernetesのNetworkPolicy
    - Kubernetes Cluster内のPod間のnetwork trafficを制御することが可能(default: 全てのPod間で通信が許可されます)
    !!! warn
        SecurityGroup for Podの代替ではありません。

        - https://aws.amazon.com/jp/blogs/news/amazon-vpc-cni-now-supports-kubernetes-network-policies/
           > Security Groups for Pods と NetworkPolicy を組み合わせて活用することで、セキュリティ体制を強化できます。NetworkPolicy が有効になっている場合、Security Groups for Pods は多層防御戦略の追加のレイヤーとして機能します。NetworkPolicy を使うと、クラスター内のネットワークトラフィックの流れをきめ細かく制御できます。一方で、Security Groups for Pods は Amazon のセマンティックコントロールを活用し Amazon RDS データベースなどの Virtual Private Cloud (VPC) 内のリソースとの通信を管理することで保護を強化します。


    !!! info

        https://docs.aws.amazon.com/eks/latest/userguide/cni-network-policy.html

        > - You can use network policies with security groups for Pods. With network policies, you can control all in-cluster communication. With security groups for Pods, you can control access to AWS services from applications within a Pod.
        >
        > - You can use network policies with custom networking and prefix delegation.

           - amazon-vpc-cni-pluginでのNetworkPolicyは以下環境で使用可能です
               - SecurityGroup for Pod
               - Custom NetworkingおよびPrefix Mode

## Custom Networking

!!! info "参考になるページ"
    - https://aws.amazon.com/jp/blogs/news/leveraging-cni-custom-networking-alongside-security-groups-for-pods-in-amazon-eks/
    - https://aws.github.io/aws-eks-best-practices/networking/custom-networking/
    - https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html
    - https://www.eksworkshop.com/docs/networking/custom-networking/configure-vpc-cni


hostNetworkingを指定しているPodはPrimary ENIにアタッチされているSecondary IPアドレスを使用します。


#### Enabled Custom Networking

Custom Networkingを有効にすると `ENIConfig` カスタムリソースで定義したSubnetとSecurityGroupをPodに割り当てることができるようになります。

1. `AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG` 環境変数を設定することでCustom Networkingを有効にできます。
    ```
    kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
    ```

1. `ENIConfig` カスタムリソースを作成
    - https://github.com/aws/amazon-vpc-cni-k8s/blob/b01e57f96567b192d96f04d239f58c3d5337ad22/pkg/apis/crd/v1alpha1/eniconfig_types.go#L26-L30
    - https://aws.github.io/aws-eks-best-practices/networking/custom-networking/
        - `AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true` の場合、CNI は ENIConfig で定義されたサブネットから Pod の IP アドレスを割り当てます。
          ENIConfig カスタムリソースは、Pod がスケジューリングされるサブネットを定義するために使用されます。
            ```
            apiVersion : crd.k8s.amazonaws.com/v1alpha1
            kind : ENIConfig
            metadata:
              name: us-west-2a
            spec: 
              securityGroups:
                - sg-0dff111a1d11c1c11
              subnet: subnet-011b111c1f11fdf11
            ```
1. 

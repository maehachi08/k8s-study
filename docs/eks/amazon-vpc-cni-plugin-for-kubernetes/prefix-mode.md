## References

- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-prefix-eni.html
- https://aws.amazon.com/jp/blogs/containers/amazon-vpc-cni-increases-pods-per-node-limits/
- https://aws.amazon.com/jp/blogs/news/amazon-vpc-cni-increases-pods-per-node-limits/
- https://aws.github.io/aws-eks-best-practices/networking/prefix-mode/index_linux/
- https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html
- https://aws.amazon.com/jp/blogs/containers/automating-custom-networking-to-solve-ipv4-exhaustion-in-amazon-eks/
- https://github.com/aws/amazon-vpc-cni-k8s#enable_prefix_delegation-v190
- https://www.eksworkshop.com/docs/networking/prefix/

## About


- Worker NodeにアタッチされるENIのSecondary IPアドレス用スロットに/28 のIP Prefix(16 IP addresses)を割り当てることができます

    !!! info
        ![](https://aws.github.io/aws-eks-best-practices/networking/prefix-mode/image.png)
        [IP addresses per network interface per instance type](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI) にある通り c5.large では1つのENIに割り当てることができるIPアドレス数は10個となります。そのうち、Podに割り当て可能なSecondary IPアドレス数は9個となります。

        Prefix Modeを有効にしている場合、ENIのSecondary IPアドレスを割り当てるスロットに /28 のIP Prefix(16 IP addresses)を割り当てることが可能となり、より多くのIPアドレスを持つことができます。

        1 Woker Nodeで起動可能なPodの最大数はEC2にアタッチ可能なENI数と割り当て可能なIPアドレス数に依存するためPrefix Modeではより多くのPodを1 Worker Node上で起動可能となります。

    !!! warning
        - /28 のIP Prefix(16 IP addresses)の連続した未使用のIPアドレスを確保できない場合はPrefix Modeの使用は控えてください
            - Subnetで払い出し可能なIPアドレスが断片化され、/28 の連続した未使用のIPアドレスを確保できない時はCNI Pluginが以下のようなエラーを記録します
                ```
                failed to allocate a private IP/Prefix address: InsufficientCidrBlocks: There are not enough free cidr blocks in the specified subnet to satisfy the request.
                ```
            - /28 のIP Prefixを確保できずにエラーとなることを回避するためには [VPC Subnet CIDR reservations](https://aws.github.io/aws-eks-best-practices/networking/prefix-mode/index_linux/#use-subnet-reservations-to-avoid-subnet-fragmentation-ipv4) でSubnet内にPrefix Mode用に予約領域を設定することができます。CNI PluginはEC2 APIを呼び出し予約領域から自動的に割り当てられるPrefixを割り当てます。

        - Prefix ModeではSecurityGroupはWorker Node上のPodで共有されます。SecurityGroupをPodごとに分けたい場合はSecurityGroup for Podを使用してください
        - SecurityGroup for PodsではPrefix Modeを使用できません


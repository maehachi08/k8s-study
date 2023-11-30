# SecurityGroup For Pod

## About

- EKSではSecurityGroupによるアクセス制御をPodに適用することが可能です
- https://aws.github.io/aws-eks-best-practices/networking/sgpp/
- https://aws.amazon.com/jp/blogs/containers/introducing-security-groups-for-pods/
- https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/managing-vpc-cni.html
- https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/cni-custom-network.html
- https://pages.awscloud.com/rs/112-TZM-766/images/20220804-AWS-kubernetes_2_ZOZO.pdf
- https://archive.eksworkshop.com/beginner/115_sg-per-pod/

### Why

なぜ `SecurityGroup For Pod` が必要なのか?

!!! info "前提"

    - [Amazon VPC CNI plugin for Kubernetes](https://github.com/aws/amazon-vpc-cni-k8s)
        - Node ENI(Primary NetworkInterface)のSecondary IPアドレスをInstance Typeごとの上限数分([refs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI)) 確保する(確保数は`WARM_IP_TARGET`により変更可能)
        - `SecurityGroup For Pod` が無効の場合はPodのIPアドレスはNode ENIに割り当てられているSecondary IPアドレスから割り当てられます


1. Node上に起動可能なPod数をNode ENI(Primary NetworkInterface)で払い出し可能なIPアドレス数以上とすることが可能
1. Node と Pod のアクセス制御を分離することが可能
    - `SecurityGroup For Pod`を使わない場合はNode ENIのSecondary IPアドレスを割り当てる、つまりNode SGでのアクセス制御が適用されます

## Usage

### Preparation


- `ENABLE_POD_ENI=true`

# aws-efs-csi-driver

- Amazon EFS CSI Driver
- https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/efs-csi.html
   > Amazon Elastic File System (Amazon EFS) は、サーバーレスで伸縮自在なファイルストレージを提供するため、ストレージ容量およびパフォーマンスのプロビジョニングや管理を行うことなくファイルデータを共有できます。Amazon EFS Container Storage Interface (CSI) ドライバーは、AWS で動作する Kubernetes クラスターが Amazon EFS ファイルシステムのライフサイクルを管理できるようにする CSI インターフェイスを提供します。

## Reference

- https://github.com/kubernetes-sigs/aws-efs-csi-driver
- https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master/docs
- https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html
- https://aws.amazon.com/jp/blogs/news/amazon-efs-csi-dynamic-provisioning/
- https://www.eksworkshop.com/docs/fundamentals/storage/efs/efs-csi-driver/
- aws blogのtags
    - https://aws.amazon.com/jp/blogs/news/category/storage/amazon-elastic-file-system-efs/
    - https://aws.amazon.com/jp/blogs/storage/category/storage/amazon-elastic-file-system-efs/
    - https://repost.aws/tags/TAmTqmXysORgKO6xcE3oh0MA/amazon-elastic-file-system

## Install

- https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html#efs-install-driver
- https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master?tab=readme-ov-file#deploy-the-driver
- https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master/charts/aws-efs-csi-driver


## About

- aws-efs-csi-driverはEKS Cluster上でEFS Volumeのライフサイクルを管理するためのCSI Interfaceを提供します。
    - EKS PodでEFS Volumeをマウントして読み書きを行うことができるようになります。
    - https://docs.aws.amazon.com/ja_jp/prescriptive-guidance/latest/patterns/run-stateful-workloads-with-persistent-data-storage-by-using-amazon-efs-on-amazon-eks-with-aws-fargate.html
        - ![](https://docs.aws.amazon.com/ja_jp/prescriptive-guidance/latest/patterns/images/pattern-img/2487e285-269b-415b-a270-877f973e3aaf/images/58e135b2-99ac-456f-ace4-0326b372ed2f.png)
- EKS Cluster上でEFS Volumeを使うためにはEFS VolumeをPersistentVolume Storageとして表現します。

### EFS Volume

aws-efs-csi-driverでPVを作成する方法はStatic ProvisioningとDynamic Provisioningがありますが、どちらの方法であっても事前にEFS Volumeを作成しておく必要があります。

!!! warning "EFS Volume作成時の設定が正しくないとPersistentVolumeを作成できません"
    - https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/docs/efs-create-filesystem.md
    - https://repost.aws/ja/knowledge-center/eks-troubleshoot-efs-volume-mount-issues

        > EFS ファイルシステムのセキュリティグループには、クラスターの VPC の CIDR からの NFS トラフィックを許可するインバウンドルールが必要です。インバウンドトラフィックにポート 2049 を許可します。
           - EFS VolumeがEKS Node上にマウントしているため

        > ポッドが EFS ボリュームのマウントに失敗しているワーカーノードに関連付けられているセキュリティグループには、アウトバウンドルールが必要です。具体的には、このアウトバウンドルールでは、EFS ファイルシステムへの NFS トラフィック (ポート 2049) を許可する必要があります。
           - EFS VolumeがEKS Node上にマウントしているため

        > EKS ノードが実行されている各アベイラビリティーゾーンに EFS マウントターゲットを作成してください。たとえば、ワーカーノードが** us-east-1a** と** us-east-1b に分散しているとします**。この場合、マウントする EFS ファイルシステムの両方のアベイラビリティーゾーンにマウントターゲットを作成します。

### Provisioning

### Static Provisioning

- https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master/examples/kubernetes/static_provisioning
- Static Provisioningは事前にPersistentVolumeを作成しておく管理方法です。

### Dynamic Provisioning

- https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/
- https://aws.amazon.com/jp/blogs/news/amazon-efs-csi-dynamic-provisioning/
- https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master/examples/kubernetes/dynamic_provisioning
- Dynamic ProvisioningはPersistentVolumeClaimが作成されたら要求された条件に合致するStorageClassからPersistentVolumeを動的に作成される管理方法です。



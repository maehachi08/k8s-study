## 参考

- https://kubernetes.io/docs/tasks/administer-cluster/cluster-upgrade/
- https://kubernetes.io/docs/reference/using-api/deprecation-guide/
- https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/update-cluster.html
- https://speakerdeck.com/shmurata/strategy-to-upgrade-kubernetes-clusters-in-production
- https://repost.aws/ja/knowledge-center/eks-plan-upgrade-cluster
- https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html#kubernetes-release-calendar

## アップグレード戦略

1. Live Upgrade(in-place upgrade)
    - https://speakerdeck.com/shmurata/strategy-to-upgrade-kubernetes-clusters-in-production?slide=23
1. Blue/Green Upgrade
    - https://speakerdeck.com/shmurata/strategy-to-upgrade-kubernetes-clusters-in-production?slide=32

### 準備

1. Kubernetesバージョンの変更点の確認
    - https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG
    - https://github.com/kubernetes/kubernetes/releases
    - https://kubernetes.io/releases/
    - AWS EKS
       - https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html
       - https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/update-cluster.html
1. 非推奨や削除されたAPIを使っているaddonを見つける
    - https://kubernetes.io/docs/reference/using-api/deprecation-guide/
    - Pluto
        - https://github.com/FairwindsOps/pluto
        - Kubernetesの非推奨(deplicated api)を自動検出する
        - https://kakakakakku.hatenablog.com/entry/2022/07/20/091424
        - https://zenn.dev/johnn26/articles/detect-kubernetes-deplicated-api-automatically
    - e.g.
        - Kyverno
            - https://kyverno.io/policies/best-practices/check_deprecated_apis/check_deprecated_apis/
1. Node

1. Addonのアップグレード計画



### アップグレード

#### AWS EKS

1. EKS Cluster Upgrade
1. ManagedNodeGroup Upgrade
   - eks-optimized AMIをUpgreade対象のcluster versionと一致するものにする


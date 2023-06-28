## 参考

https://kubernetes.io/docs/tasks/administer-cluster/cluster-upgrade/
https://kubernetes.io/docs/reference/using-api/deprecation-guide/
https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/update-cluster.html
https://speakerdeck.com/shmurata/strategy-to-upgrade-kubernetes-clusters-in-production

## アップグレード戦略

1. Live Upgrade(in-place upgrade)
    - https://speakerdeck.com/shmurata/strategy-to-upgrade-kubernetes-clusters-in-production?slide=23
1. Blue/Green Upgrade
    - https://speakerdeck.com/shmurata/strategy-to-upgrade-kubernetes-clusters-in-production?slide=32

### 準備

1. 非推奨や削除されたAPIを使っているaddonを見つける
    - Kyverno
        - https://kyverno.io/policies/best-practices/check_deprecated_apis/check_deprecated_apis/
    - Pluto
        - https://github.com/FairwindsOps/pluto
        - Kubernetesの非推奨(deplicated api)を自動検出する
        - https://kakakakakku.hatenablog.com/entry/2022/07/20/091424
        - https://zenn.dev/johnn26/articles/detect-kubernetes-deplicated-api-automatically


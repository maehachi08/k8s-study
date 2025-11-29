## Concepts

- https://karpenter.sh/v1.0/concepts/


KarpenterのNode管理には



#### [NodePools](https://karpenter.sh/docs/concepts/nodepools/)

NodePoolとはKarpenterがProvisionすることができるNode設定、Karpenterが起動したNodeに適用されるconstraintsなどの設定を定義します。
Karpenterをインストールするとdefault NodePoolが作成されます。

KarpenterのNodePoolでは以下のような設定を定義します。

- taints
    - ScheduleされるPodに関するTaints
- startupTaints
    - Karpenterに通知するためにNode起動直後のみ付与されるTaints
- expireAfter
    - Karpenterが起動したNodeのlifecycle
- disruption
    - 



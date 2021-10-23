## Worker Node

### Resources

#### OOM Killer

##### 参考

- k8s.af から
    - https://www.bluematador.com/blog/post-mortem-kubernetes-node-oom
- https://www.scsk.jp/sp/sysdig/blog/sysdig_monitor/kubernetes_oomcpu.html
- Goldstine研究所
   - https://blog.mosuke.tech/entry/2020/03/31/kubernetes-resource/
   - https://blog.mosuke.tech/entry/2021/03/11/kubernetes-node-down/
- 外道父の匠
    - http://blog.father.gedow.net/2019/11/28/eks-kubernetes-ouf-of-memory/

##### Impact

- MemoryがNode上限に達した場合OOM KillerがPodを強制停止する
    - 強制停止するPodはQoS Classの優先度で決まる ([参考](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/#node-out-of-memory-behavior))
    - `Note: The kubelet also sets an oom_score_adj value of -997 for containers in Pods that have system-node-critical Priority`
    - 同じQoS Class(Burstable) の場合はPodのrequestしたメモリー量とノードのキャパシティの割合によってスコア付けされる

##### Recommend

- `Requests Memory = Limits Memory` で設定する
   - Limits MemoryがRequests Memoryより多い場合、Requests MemoryがAllocatableに収まるようにスケジュールされ、Allocatableを超えた場合にNodeのOOM Killerが発動するのでシステム全体のプロセスからKill対象から選ばれる


### Node - 停止

#### 参考

- [Goldstine研究所: Kubernetesのノード障害時のPodの動きについての検証](https://blog.mosuke.tech/entry/2021/03/11/kubernetes-node-down/)

#### Impact

- ReplicaSet Podの再配置が行われる


- `node_lifecycle_controller`
    1. KubeletがNode情報を更新しなくなったことを検知してNodeのStatusを変更
    1. `key: node.kubernetes.io/unreachable` のTaintを付与

- Podの再配置
    - Podが作成される時にDefaultTolerationSeconds AdmissionControllerによって以下のtolerationsが付与されている
        - `node.kubernetes.io/not-ready:NoExecute`
        - `node.kubernetes.io/unreachable:NoExecute`
    - これらのtolerationsが `tolerationSeconds: 300` を設定されているため、300秒経過後にNodeが復旧せず`node.kubernetes.io/unreachable` Taintが外れない場合PodがEviction（強制退去）され別のノードにスケジュールされる
    - https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaulttolerationseconds
    - 別ノードへの再スケジュールまで最大5分のタイムラグが発生する場合がある
      - DefaultTolerationSeconds


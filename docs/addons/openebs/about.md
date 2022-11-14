## OpenEBS

- `OpenEBS` は、Kubernetesワーカーノードが利用できるあらゆるストレージ(ローカルや分散など) をKubernetes Persistent Volumesに変換します

### Reference

- https://openebs.io/
- https://github.com/openebs/openebs
- https://github.com/openebs/charts
- https://openebs.github.io/charts/
- https://blog.openebs.io/arming-kubernetes-with-openebs-1-b450f41e0c1f
- https://note.com/ryoma_0923/n/n7d2837212028
- https://qiita.com/ysakashita/items/8ca805cb6ac10df911be
- https://blog.cybozu.io/entry/2018/03/29/080000
- https://www.infoq.com/jp/news/2020/08/kubernetes-storage-kubera/

### OpenEBS について

- MayaDataが実装を公開し、現在はCNCF sandbox projectの[Container Storage Interface (CSI)](https://github.com/container-storage-interface/spec/blob/master/spec.md) Providerです
- `Container Attached Storage(CAS)` アーキテクチャーを採用している
- Dynamic Provisionerをサポートしている
    - Kubernetes Nativeの local PersistentVolume volume はStatic Provisioningのみをサポート

#### `Container Attached Storage(CAS)`

- https://openebs.io/docs/concepts/cas
- https://www.cncf.io/blog/2018/04/19/container-attached-storage-a-primer/
- https://www.cncf.io/blog/2020/09/22/container-attached-storage-is-cloud-native-storage-cas/
- https://www.cncf.io/online-programs/kubernetes-and-storage-kubernetes-for-storage-an-overview/
- https://blog.mayadata.io/container-attached-storage-cas-vs.-shared-storage-which-one-to-choose

    > In Kubernetes, shared storage is typically achieved by mounting volumes and connecting to an external filesystem or block storage solution. Container Attached Storage (CAS) is a relatively newer solution that allows Kubernetes administrators to deploy storage as containerized microservices in a cluster.

`Container Attached Storage(CAS)` とはPodで利用可能なストレージをコンテナ化したマイクロサービスとしてKubernetes Clusterへデプロイする仕組みです。
`CAS` ではPersistent Volumesをマイクロサービス ベースのストレージ レプリカとして構成します。その際、ストレージ レプリカを管理するためのストレージ コントローラを、独立してスケーリングおよび実行できる 構成ユニットとしてデプロイします。(つまり、OpenEBS におけるPVはVolumeをホストするreplica pod と replica podを管理するcontroller podで構成されます)

- 以下は [Deploy Jenkins with OpenEBS](../install/#deploy-jenkins-with-openebs) で作成されるストレージ レプリカとコントローラ
    ```
    $ kubectl get pods -n openebs | grep pvc-2fee1acb
    openebs                pvc-2fee1acb-2d7f-4068-b56e-777eefa35e4a-jiva-ctrl-66d449fp8jp8   2/2     Running   0                3h58m   10.200.0.39    k8s-master   <none>           <none>
    openebs                pvc-2fee1acb-2d7f-4068-b56e-777eefa35e4a-jiva-rep-0               1/1     Running   1 (3h34m ago)    4h16m   10.200.2.194   k8s-node2    <none>           <none>
    openebs                pvc-2fee1acb-2d7f-4068-b56e-777eefa35e4a-jiva-rep-1               1/1     Running   1 (3h40m ago)    4h16m   10.200.0.38    k8s-master   <none>           <none>
    openebs                pvc-2fee1acb-2d7f-4068-b56e-777eefa35e4a-jiva-rep-2               0/1     Pending   0                4h16m   <none>         <none>       <none>           <none>
    ```

### Node Disk Manager(NDM)

- https://openebs.io/docs/main/concepts/ndm
    ![](https://openebs.io/docs/assets/files/ndm-96fc51e849ddea8084d8b800d0e08975.svg)
      - CPUやMemory、Networkなどと同じようにNode上のblock deviceをkubernetes resources(CustomResource)として管理するためのコンポーネント
      - DeamonSetとして各Nodeにデプロイされる
      - Node上のblock deviceへのアクセスするのに `/dev`, `/proc`, `/sys` へのアクセス権限が必要なためPrivileged modeで動作する
      - `Local PV` と `cStor PV` で使用される
         - `JIVA PV` はNDMへアクセスしていない?
         - https://openebs.io/docs/main/user-guides/ndm
            ![](https://openebs.io/docs/assets/files/2-config-sequence-e3baee9015c5bd5936c141256de38715.svg)

### ストレージエンジン

- https://openebs.io/docs/main/concepts/casengines

    | Storage Engine | Status | Description | link |
    |:---|:---|:---|:---|
    | `Local PV` | Beta | :material-check: 単一ノードで利用可能なVolumeを提供<br>:material-check: Dynamic Provisioningをサポート(Kubernetes Nativeの local PersistentVolume volume はStatic Provisioning(事前のPVを手動作成する必要がある)) | https://openebs.io/docs/concepts/localpv<br>https://openebs.io/docs/main/user-guides/localpv-device |
    | `JIVA PV` | Stable | :material-check: 複数ノードでレプリカを構成したVolumeを提供(高可用性)<br>:material-check: iSCSI Volumeをエミュレート<br>:material-check: Dynamic Provisioningをサポート<br>:material-check: シン プロビジョニングをサポート | https://openebs.io/docs/concepts/jiva |
    | `cStor PV` | Beta | :material-check: 複数ノードでレプリカを構成したVolumeを提供(高可用性)<br>:material-check: iSCSI Volumeをエミュレート<br>:material-check: Dynamic Provisioningをサポート<br>:material-check: シン プロビジョニングをサポート<br>:material-check: Snapshotをサポート | https://openebs.io/docs/concepts/cstor |

      - ストレージエンジンの選択基準について
          - https://openebs.io/docs/2.12.x/concepts/casengines#cstor-vs-jiva-vs-localpv-features-comparison


# Container Runtime

## High Level Container Runtime

- https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/
- Container Runtime Interfaces(CRI)
    - kubeletから呼ばれるContainer Runtime
    - contanerd や cri-o などがCRIに該当する

### [cri-o](https://cri-o.io/)

- https://github.com/cri-o/cri-o
- RedHatが主導で開発
- 純粋にコンテナを起動させるための実装なので軽量かつセキュアらしい
- image の build や push は非サポート

    > What is not in scope for this project?
    >
    > - Building, signing and pushing images to various image storages

### [containerd](https://containerd.io/)

- https://github.com/containerd/containerd
- [CNCFのgraduated project](https://www.cncf.io/announcements/2019/02/28/cncf-announces-containerd-graduation/)
- dockerdからCRI Interface部分を分離(containerd v1.1でnativeに対応)
- image の build や push をサポート

## Low Level Container Runtime

### [runc](https://github.com/opencontainers/runc)

- コンテナ仕様の標準化団体である [Open Container Initiative(OCI)](https://opencontainers.org/) が公開
- `containerd` や `cri-o` から実行される

## 参考

- https://thinkit.co.jp/article/18024
- https://medium.com/nttlabs/container-runtime-d3e25189f67a
- https://qiita.com/mamomamo/items/ed5db2ab1555078f8a24
- https://www.kimullaa.com/entry/2021/05/07/204706
- https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/
- https://www.publickey1.jp/blog/20/firecrackergvisorunikernel_container_runtime_meetup_2.html

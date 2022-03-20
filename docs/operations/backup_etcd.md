# Backup etcd

## `etcdctl` インストール

```
sudo apt install etcd-client
```

## ercd 疎通確認

- https://github.com/etcd-io/etcd/tree/main/etcdctl#endpoint-health
    ```
    ETCD_ENDPOINT=https://192.168.10.50:2379
    ETCD_CACERT=/etc/etcd/ca.pem
    ETCD_CERT=/etc/etcd/kubernetes.pem
    ETCD_KEY=/etc/etcd/kubernetes-key.pem

    ETCDCTL_API=3 etcdctl endpoint health \
      --endpoints=${ETCD_ENDPOINT} \
      --cacert=${ETCD_CACERT} \
      --cert=${ETCD_CERT} \
      --key=${ETCD_KEY}
    ```
      - `https://192.168.10.50:2379 is healthy: successfully committed proposal: took = 32.557549ms` などの標準出力が確認できれば疎通できている

## etcd バックアップ

- https://github.com/etcd-io/etcd/tree/main/etcdctl#snapshot-save-filename
    ```
    ETCD_ENDPOINT=https://192.168.10.50:2379
    ETCD_CACERT=/etc/etcd/ca.pem
    ETCD_CERT=/etc/etcd/kubernetes.pem
    ETCD_KEY=/etc/etcd/kubernetes-key.pem

    ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
      --endpoints=${ETCD_ENDPOINT} \
      --cacert=${ETCD_CACERT} \
      --cert=${ETCD_CERT} \
      --key=${ETCD_KEY}
    ```

### バックアップファイルの情報を確認

- https://github.com/etcd-io/etcd/tree/main/etcdctl#snapshot-status-filename
    ```
    ETCDCTL_API=3 etcdctl snapshot status snapshot.db
    ```
      - 以下のような出力が確認できます
          ```
          +----------+----------+------------+------------+---------+
          |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE | VERSION |
          +----------+----------+------------+------------+---------+
          | c9a815d0 |     3356 |        488 |     1.1 MB |         |
          +----------+----------+------------+------------+---------+
          ```

## etcd リストア

- https://github.com/etcd-io/etcd/tree/main/etcdctl#snapshot-restore-options-filename
    ```
    ETCD_ENDPOINT=https://192.168.10.50:2379
    ETCD_CACERT=/etc/etcd/ca.pem
    ETCD_CERT=/etc/etcd/kubernetes.pem
    ETCD_KEY=/etc/etcd/kubernetes-key.pem

    ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
      --endpoints=${ETCD_ENDPOINT} \
      --cacert=${ETCD_CACERT} \
      --cert=${ETCD_CERT} \
      --key=${ETCD_KEY}
    ```


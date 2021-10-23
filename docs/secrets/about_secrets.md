# About Secrets

## 参考

- https://kubernetes.io/ja/docs/concepts/configuration/secret/
- https://kubernetes.io/ja/docs/tasks/configmap-secret/managing-secret-using-kubectl/

## Overview

- API Key、Token、SSH Keyなど機密性の高い情報を格納するためのリソース
- 参照方法
   - Volume内のファイルとしてマウント
   - 環境変数
   - imagePullSecrets
- 値の格納フォーマット
   - `date`: base64エンコーディングされた文字列
   - `stringData`: 平文などそのまま格納したい場合

## Kind of secret

### Builtin Type

- Opaque
- kubernetes.io/service-account-token
- kubernetes.io/dockercfg
- kubernetes.io/dockerconfigjson
- kubernetes.io/basic-auth
- kubernetes.io/ssh-auth
- kubernetes.io/tls
- bootstrap.kubernetes.io/token



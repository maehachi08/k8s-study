# Monitoring/Debug

## Monitoring

https://docs.fluentbit.io/manual/administration/monitoring

Fluent BitはビルトインHTTP Serverを起動することで各Pluginのメトリックス情報を提供するAPIへアクセスできるようになります。

### Integrations

#### Prometheus

- https://docs.fluentbit.io/manual/pipeline/outputs/prometheus-exporter

#### DataDog

##### 1. 公式 Datadog インテグレーション

- https://docs.datadoghq.com/ja/agent/guide/integration-management/
- https://github.com/DataDog/integrations-extras/tree/master/fluentbit

##### 2. Prometheus formatをOpenMetricsとして収集する

- https://docs.datadoghq.com/ja/integrations/openmetrics/
- https://github.com/DataDog/integrations-core/issues/7117

## Debug

https://docs.fluentbit.io/manual/administration/dump-internals-signal

fluent-bitプロセスに対して`CONT` Unix signalを送信することで`Input Plugin` と `Storage Layer` に関する内部情報をダンプしてくれる。


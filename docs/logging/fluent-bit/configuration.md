## Configuration

- https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/classic-mode/configuration-file
- https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/classic-mode/format-schema
- v1.9 で `YAML形式` をpreview support
    - https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/yaml

## fluent-bit monitoring/healthcheck

- https://docs.fluentbit.io/manual/pipeline/pipeline-monitoring

```
[SERVICE]
    HTTP_Server  On
    HTTP_Listen  0.0.0.0
    HTTP_PORT    2020
```

### Section

fluent-bitはセクションごとに設定を記述します。

セクションは複数のプロパティが定義可能です。プロパティは key/value で指定します。

| section | description |
|:---|:---|
| `SERVICE`          | fluent-bitのグローバル設定を定義するセクション |
| `INPUT`            | 転送対象となるログに付いて定義するセクション |
| `PARSE`            | `INPUT` で取り込むログを構造化データ(レコード)とするための定義を記述するセクション |
| `MULTILINE_PARSER` | `INPUT` で取り込むログを複数行で1つのまとまりとして構造化データ(レコード)とするための定義を記述するセクション |
| `FILTER`           | `INPUT` で取り込んだレコードに対して追加処理を定義するセクション |
| `OUTPUT`           | 転送先を定義するセクション |
| `PLUGIN`           | 追加のプラグインを定義するセクション |

### `SERVICE`

- https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/classic-mode/configuration-file#config_section


### `INPUT`

- https://docs.fluentbit.io/manual/pipeline/inputs
- `INPUT` セクションはどのpluginを使うか(なにを入力とするか) を `Name` プロパティで指定します
- `Tag` プロパティで環境変数を利用可能



### `PARSE`
### `MULTILINE_PARSER`

- https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/multiline-parsing


### `FILTER`
### `OUTPUT`

- https://docs.fluentbit.io/manual/pipeline/outputs
- `OUTPUT` セクションはどのpluginを使うか(どこに転送するか) を `Name` プロパティで指定します

    <details>
    <summary>e.g. S3</summary>

    ```
    [OUTPUT]
        Name                         s3
        Match                        *
        bucket                       my-bucket
        region                       us-west-2
        total_file_size              250M
        s3_key_format                /$TAG[2]/$TAG[0]/%Y/%m/%d/%H/%M/%S/$UUID.gz
        s3_key_format_tag_delimiters .-
    ```

    </details>


- `Retry_Limit`
    - https://docs.fluentbit.io/manual/administration/scheduling-and-retries
        - `output` pluginは転送処理の結果ステータスをengineに対して通知する
        - `Retry` が要求された場合、EngineはSchedulerにそのデータのフラッシュを再試行するように要求し、Schedulerはそれが起こるまでに何秒待つかを決定する

### `PLUGIN`


### config file

fluent-bitはセクションをconfig fileに記述します。

各セクションは `/etc/td-agent-bit/fluent-bit.conf` に纏めて記述することも可能です。
可読性や保守性からもセクションごとにファイル分割しておくのが良いかもしれません。

以下は一例です。

| file | description |
|:---|:---|
| `/etc/td-agent-bit/fluent-bit.conf`        | 起点となるconfig file<br>`SERVICE`セクションや `@INCLUDE` を記述する(もちろん他のセクションも記述できます)|
| `/etc/td-agent-bit/parsers.conf`           | `PARSE` セクションを記述する |
| `/etc/td-agent-bit/multiline_parsers.conf` | `MULTILINE_PARSER` セクションを記述する |
| `/etc/td-agent-bit/input.conf`             | `INPUT` セクションを記述する |
| `/etc/td-agent-bit/output.conf`            | `OUTPUT` セクションを記述する |
| `/etc/td-agent-bit/filter.conf`            | `FILTER` セクションを記述する |
| `/etc/td-agent-bit/plugin.conf`            | `PLUGIN` セクションを記述する |


### Tips

- 
https://docs.fluentbit.io/manual/pipeline/filters/grep

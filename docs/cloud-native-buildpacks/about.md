
### Cloud NativeなImage事情

従来、container runtimeはDockerがデファクトスタンダードでした。

最近のCloud Nativeなcontainer runtime事情としては、
[Container Runtime](/container-runtime/) のページで記載した通り、container runtimeは`High Level Container Runtime` と `Low Level Container Runtime` で役割を分離しています。
`Low Level Container Runtime` はdaemon serviceではなくバイナリで提供され、`High Level Container Runtime` から実行されます。`Low Level Container Runtime` は[OCI(Open Container Initiative)](https://opencontainers.org/)に準拠した [`config.json`](https://github.com/opencontainers/runtime-spec/blob/main/config.md) に生成すべきコンテナのメタ情報が書かれているのでそれを基にコンテナを生成します。

この辺りの話は以下ページなどがわかりやすかったです。

- https://www.publickey1.jp/blog/20/firecrackergvisorunikernel_container_runtime_meetup_2.html
- https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/
- https://qiita.com/mamomamo/items/ed5db2ab1555078f8a24

このcontainer runtimeの話は`config.json`に書かれている情報を基にコンテナを生成するための仕組みです。コンテナ情報をポータブルに持ち出せる状態(OCI imageと言われますが実態はtar archive)として保存する仕組みはCRI(の`High Level Container Runtime`)として標準実装ではないため別途考慮する必要があります。(例えば、cri-oではimage buildを非サポートだったりします)

OCI imageのLayoutについての公式リファレンスは以下です。

- https://github.com/opencontainers/image-spec/blob/main/image-layout.md

### Cloud Native Buildpacksとは

- https://buildpacks.io/
- https://github.com/buildpacks
- https://github.com/buildpacks/spec

`Cloud Native Buildpacks` はアプリケーションのソースコードをDockerfileなしでOCI準拠のimageとしてbuildするためのツールです。

元々は `Buildpacks` という名前でHerokuが内製ツールとして作ったものですが、2018年に [CNCF Project](https://www.cncf.io/projects/buildpacks/) として公開されました。
Herokuのツール名と見分けるため、 `Cloud Native Buildpacks(CNB)` と表現されることが多いようです。

![](https://buildpacks.io/docs/concepts/what.svg){ width="600" }

アプリケーションのソースコードが配置されているディレクトリで Builderと呼ばれるbuildpackやbuild image, run imageを組み合わせたcomponentを指定して `pack build` コマンドを実行することでOCI準拠のimageをbuildできます。build時の処理は `Detect phase` と `Build phase` があります。

(余談)
`pack build` コマンドで `--buildpack` オプションを指定することでbuilderに含まれないbuildpackを指定することも可能

1. Detect phase
    - https://buildpacks.io/docs/concepts/#detect-phase
1. Build phase
    - https://buildpacks.io/docs/concepts/#build-phase

`Builder` は独自に作成することも可能ですが、[Paketo Buildpacks](https://paketo.io/) のようなコミュニティが提供するBuilderやBuildpackを利用することも可能です。

`pack builder suggest` コマンドを実行したところ、`Google`, `Heroku`, `Paketo Buildpacks` がsuggestされました。

```
$ pack builder suggest
Suggested builders:
        Google:                gcr.io/buildpacks/builder:v1      Ubuntu 18 base image with buildpacks for .NET, Go, Java, Node.js, and Python
        Heroku:                heroku/builder:22                 Base builder for Heroku-22 stack, based on ubuntu:22.04 base image
        Heroku:                heroku/buildpacks:20              Base builder for Heroku-20 stack, based on ubuntu:20.04 base image
        Paketo Buildpacks:     paketobuildpacks/builder:base     Ubuntu bionic base image with buildpacks for Java, .NET Core, NodeJS, Go, Python, Ruby, Apache HTTPD, NGINX and Procfile
        Paketo Buildpacks:     paketobuildpacks/builder:full     Ubuntu bionic base image with buildpacks for Java, .NET Core, NodeJS, Go, Python, PHP, Ruby, Apache HTTPD, NGINX and Procfile
        Paketo Buildpacks:     paketobuildpacks/builder:tiny     Tiny base image (bionic build image, distroless-like run image) with buildpacks for Java, Java Native Image and Go

Tip: Learn more about a specific builder with:
        pack builder inspect <builder-image>
```

### Components

- https://buildpacks.io/docs/concepts/components/

#### Builder

- https://buildpacks.io/docs/concepts/components/builder/
  ![](https://buildpacks.io/docs/concepts/components/create-builder.svg){ width: 600px }
    - 1つ以上のbuildpack
    - stack
    - build image
    - run image
- `builder.toml`
    - https://buildpacks.io/docs/reference/config/builder-config/

#### Buildpack

- https://buildpacks.io/docs/concepts/components/buildpack/
    - `buildpack.toml`
    - `bin/detect`
        - 当該Buildpackを適用するかどうかをチェックする
        - e.g.
            - Ruby appの場合、 `Gemfile` の存在確認や `.ruby-version` のversion checkなど ([e.g. sample](https://github.com/buildpacks/samples/blob/main/buildpacks/ruby-bundler/bin/detect))
    - `bin/build`
        - 当該Buildpackを適用しbuildする
        - e.g.
            - Ruby appの場合、`bundle install` 実行など ([e.g. sample](https://github.com/buildpacks/samples/blob/main/buildpacks/ruby-bundler/bin/build))

- https://buildpacks.io/docs/concepts/components/buildpack/#meta-buildpack
    - `meta-buildpack` とは他の`Buildpack`を参照する順序設定を持つ `buildpack.toml` だけ持つものです

#### Stack

- https://buildpacks.io/docs/concepts/components/stack/
    - `build image` と `run image` で構成されます
        - `build image` とは[lifecycle](https://buildpacks.io/docs/concepts/components/lifecycle/)(及びそれによるbuildpack) が実行されるコンテナ環境のbase image
        - `run image` とはapplication imageをbuildするためのbase image

- https://buildpacks.io/docs/operator-guide/create-a-stack/
    - custom stackを作成するには`build image` と `run image`をcustom imageとして用意します
        1. common base imageの作成
        1. run imageの作成
        1. build imageの作成
    - Buildpack の stackで使用するimageは以下設定が必要です
        1. `io.buildpacks.stack.id` Label
        1. 環境変数
            - `CNB_USER_ID`
            - `CNB_GROUP_ID`
            - `CNB_STACK_ID`

            <details><summary>e.g. Dockerfile</summary>
            ```
            # 1. Set a common base
            FROM ubuntu:bionic as base
            
            # 2. Set required CNB information
            ENV CNB_USER_ID=1000
            ENV CNB_GROUP_ID=1000
            ENV CNB_STACK_ID="io.buildpacks.samples.stacks.bionic"
            LABEL io.buildpacks.stack.id="io.buildpacks.samples.stacks.bionic"
            
            # 3. Create the user
            RUN groupadd cnb --gid ${CNB_GROUP_ID} && \
              useradd --uid ${CNB_USER_ID} --gid ${CNB_GROUP_ID} -m -s /bin/bash cnb
            
            # 4. Install common packages
            RUN apt-get update && \
              apt-get install -y xz-utils ca-certificates && \
              rm -rf /var/lib/apt/lists/*
            
            # 5. Start a new run stage
            FROM base as run
            
            # 6. Set user and group (as declared in base image)
            USER ${CNB_USER_ID}:${CNB_GROUP_ID}
            
            # 7. Start a new build stage
            FROM base as build
            
            # 8. Install packages that we want to make available at build time
            RUN apt-get update && \
              apt-get install -y git wget jq && \
              rm -rf /var/lib/apt/lists/* && \
              wget https://github.com/sclevine/yj/releases/download/v5.0.0/yj-linux -O /usr/local/bin/yj && \
              chmod +x /usr/local/bin/yj
            
            # ========== ADDED ===========
            # 9. Set user and group (as declared in base image)
            USER ${CNB_USER_ID}:${CNB_GROUP_ID}
            ```
            </details>

            ``` bash
            docker build . -t cnbs/sample-stack-build:bionic --target build
            ```

    - `io.buildpacks.stack.id` ラベルを設定したimageをbuilderで指定します
        <details><summary>e.g. builder.toml</summary>
        ```
        [[buildpacks]]
          # ...
        
        [[order]]
          # ...
        
        [stack]
          id = "com.example.stack"
          build-image = "example/build"
          run-image = "example/run"
          run-image-mirrors = ["gcr.io/example/run", "registry.example.com/example/run"]
        ```
        </details>

#### Buildpack Group

- https://buildpacks.io/docs/concepts/components/buildpack-group/
    - 実行する順番に定義されたBuildpackのリストです。
      `Buildpack` はモジュール化されているため再利用が可能で`Buildpack Group` で複数の`Buildpack`を接続することが可能です。
    - `buildpack.toml` で他Buildpackを指定している例
        - `samples/hello-world` と `samples/hello-moon` Buildpackを持つ `samples/hello-universe` Buildpack Group
            - https://github.com/buildpacks/samples/blob/main/buildpacks/hello-universe/buildpack.toml
                <details><summary>buildpack.toml</summary>
                ```
                # Buildpack API version
                api = "0.2"
                
                # Buildpack ID and metadata
                [buildpack]
                id = "samples/hello-universe"
                version = "0.0.1"
                name = "Hello Universe Buildpack"
                homepage = "https://github.com/buildpacks/samples/tree/main/buildpacks/hello-universe"
                
                # Order used for detection
                [[order]]
                [[order.group]]
                id = "samples/hello-world"
                version = "0.0.1"
                
                [[order.group]]
                id = "samples/hello-moon"
                version = "0.0.1"
                ```
                </details>

        - `Builder` から `[[buildpacks]]` で `samples/hello-universe` Buildpack Groupを指定することで`samples/hello-world` と `samples/hello-moon` Buildpackを利用できる
            - https://buildpacks.io/docs/operator-guide/create-a-builder/#1-builder-configuration
            - `[[order]]` > `[[order.group]]` に並んでいる順番にdetect/buildが実行される
                <details><summary>builder.toml</summary>
                ```
                # Buildpacks to include in builder
                [[buildpacks]]
                uri = "samples/buildpacks/hello-processes"
                
                [[buildpacks]]
                # Packaged buildpacks to include in builder;
                # the "hello-universe" package contains the "hello-world" and "hello-moon" buildpacks
                uri = "docker://cnbs/sample-package:hello-universe"
                
                # Order used for detection
                [[order]]
                    # This buildpack will display build-time information (as a dependency)
                    [[order.group]]
                    id = "samples/hello-world"
                    version = "0.0.1"
                
                    # This buildpack will display build-time information (as a dependant)
                    [[order.group]]
                    id = "samples/hello-moon"
                    version = "0.0.1"
                
                    # This buildpack will create a process type "sys-info" to display runtime information
                    [[order.group]]
                    id = "samples/hello-processes"
                    version = "0.0.1"
                
                # Stack that will be used by the builder
                [stack]
                id = "io.buildpacks.samples.stacks.bionic"
                # This image is used at runtime
                run-image = "cnbs/sample-stack-run:bionic"
                # This image is used at build-time
                build-image = "cnbs/sample-stack-build:bionic"
                ```
                </details>

#### Lifecycle

- https://buildpacks.io/docs/concepts/components/lifecycle/
    - buildpackの実行をオーケストレーションし、application imageをbuildします
    - https://github.com/buildpacks/lifecycle
    - https://github.com/buildpacks/spec/blob/main/platform.md#lifecycle-interface

##### 1. Analyze

- https://buildpacks.io/docs/concepts/components/lifecycle/analyze/
    - BuildおよびExportフェーズを最適化するためのファイルを復元します

##### 2. Detect

- https://buildpacks.io/docs/concepts/components/lifecycle/detect/
    - Buildフェーズで実行するbuildpackの順序を検出します

##### 3. Restore

- https://buildpacks.io/docs/concepts/components/lifecycle/restore/
    - 前回のimageとimage cacheからlayerのmetadataを復元しcacheされたimage layerを復元します

##### 4. Build

- https://buildpacks.io/docs/concepts/components/lifecycle/build/
    - application source codeを実行可能なartifactへ変換しコンテナに格納します
    - https://buildpacks.io/docs/concepts/operations/build/
        - ![](https://buildpacks.io/docs/concepts/operations/build.svg){ 300px }

##### 5. Export

- https://buildpacks.io/docs/concepts/components/lifecycle/export/
    - OCI imageを作成します

##### 6. Create

- https://buildpacks.io/docs/concepts/components/lifecycle/create/
    - analyze, detect, restore, build, export を1コマンドで実行します

##### 7. Launch

- https://buildpacks.io/docs/concepts/components/lifecycle/launch/
    - OCI imageのEntrypointを設定します

##### 8. Rebase

- https://buildpacks.io/docs/concepts/components/lifecycle/rebase/
    - 前のimageからbase image(run image)を更新したい場合に、該当のimage layerだけ差し替えます
    - https://buildpacks.io/docs/concepts/operations/rebase/
    - ![](https://buildpacks.io/docs/concepts/operations/rebase.svg){ 300px }

#### Platform

- https://buildpacks.io/docs/concepts/components/platform/
    - Cloud Native Buildpacksを使用してOCI imageをbuildするプラットフォーム
        - CLI (e.g. [Pack CLI](https://github.com/buildpacks/pack))
        - CI Service (e.g. [Tekton buildpacks plugin](https://github.com/tektoncd/catalog/tree/main/task/buildpacks))
        - Cloud Service (e.g. [kpack](https://github.com/pivotal/kpack))

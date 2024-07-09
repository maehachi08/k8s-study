# Gateway API

## References

- https://gateway-api.sigs.k8s.io/
- https://github.com/kubernetes-sigs/gateway-api/releases
- https://medium.com/google-cloud-jp/gke-gateway-4150649d8c37
- https://medium.com/google-cloud-jp/gke-gateway2-317ed62f727f
- https://amsy810.hateblo.jp/entry/2023/12/01/090000
- https://qiita.com/ipppppei/items/6edb78b513c7bbe55d91
- https://thinkit.co.jp/article/19625
- https://zenn.dev/pkkudo/articles/cb1942011d81ae

## About

Gateway APIはClusterの外にいるClientからClusterの中にあるServiceへのtraffic routingを管理する目的で策定されました。

Gatewayリソース(基礎となるネットワークゲートウェイ/プロキシサーバ) を中心としたリソースのコレクションであり、シンプルな機能提供にとどまるIngressに対してプロトコルやtraffic routingなど運用面で不足している機能などが追加されたNetwork機能を提供するものです。


[2023/10/31にv1.0: GA Release](https://kubernetes.io/blog/2023/10/31/gateway-api-ga/) されたKubernetes Projectです。


> Gateway API is an official Kubernetes project focused on L4 and L7 routing in Kubernetes. This project represents the next generation of Kubernetes Ingress, Load Balancing, and Service Mesh APIs. From the outset, it has been designed to be generic, expressive, and role-oriented.
-- <cite>[Introduction](https://gateway-api.sigs.k8s.io/) から引用</cite>


Ingressにはいくつかの課題があります。<br>
refs https://blog.nginx.org/blog/5-reasons-to-try-the-kubernetes-gateway-api

1. インフラ管理者/Cluster管理者/アプリケーション開発者 間の責任分界点が明確ではなかった
1. L7しかサポートしていない
1. 細かなtraffic routingはingress controller(Nginx IngressControllerやAWS load-balancer-controllerなど) ごとに提供されるannotationで管理するため環境依存が発生する

Gateway APIはこれらの問題を解決することも考慮し策定されました。

1. 以下のようなリソースモデルにより役割が分かれている
    - ![](https://gateway-api.sigs.k8s.io/images/resource-model.png)
1. L4とL7をサポート
1. HTTPS/HTTP だけでなくTLSやTCP/UDPなど様々なtraffic routingをサポート
    - https://gateway-api.sigs.k8s.io/guides/

### Architecture

- https://gateway-api.sigs.k8s.io/concepts/api-overview/
- https://docs.nginx.com/nginx-gateway-fabric/overview/gateway-architecture/

### Resources

- https://gateway-api.sigs.k8s.io/concepts/api-overview/
    - `GatewayClass`
        - https://gateway-api.sigs.k8s.io/api-types/gatewayclass/
    - `Gateway`
        - https://gateway-api.sigs.k8s.io/api-types/gateway/
    - `xRoute`
        - `HTTPRoute`
            - https://gateway-api.sigs.k8s.io/guides/http-routing/
            - https://gateway-api.sigs.k8s.io/api-types/httproute/
            - https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.HTTPRoute
        - `TLSRoute`
            - https://gateway-api.sigs.k8s.io/guides/tls/
            - https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1alpha2.TLSRoute
        - `TCPRoute` / `UDPRoute`
            - https://gateway-api.sigs.k8s.io/guides/tcp/
            - https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1alpha2.TCPRoute
        - `GRPCRoute`
            - [v1.1.0](https://github.com/kubernetes-sigs/gateway-api/releases/tag/v1.1.0) でGA release
            - https://gateway-api.sigs.k8s.io/guides/grpc-routing/
            - https://gateway-api.sigs.k8s.io/api-types/grpcroute/
            - https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.GRPCRoute
    - `ReferenceGrant`
        - https://gateway-api.sigs.k8s.io/api-types/referencegrant/
        - https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1alpha2.ReferenceGrant
    - `BackendTLSPolicy`

### BackendTLSPolicy

[v1.1.0](https://github.com/kubernetes-sigs/gateway-api/releases/tag/v1.1.0)では `v1alpha3` の機能です。

Gateway から backend serviceにHTTPSでtraffic routingする場合のTLS設定です。Cluster外のclientからのTLS暗号をbackend serviceで終端させる `passthrough` と Cluster外のclientからのTLS暗号をGatewayで終端させてGatewayとbackend serviceで新たなTLS暗号を行う `edge` があります。

- https://gateway-api.sigs.k8s.io/guides/tls/
- https://gateway-api.sigs.k8s.io/api-types/backendtlspolicy/
- https://gateway-api.sigs.k8s.io/geps/gep-1897/

### ReferenceGrant

- https://gateway-api.sigs.k8s.io/api-types/referencegrant/
    - https://qiita.com/ipppppei/items/8e6e1f0e77de10e2446d
    - https://thinkit.co.jp/article/21458

### Progressive delivery

xRouteリソースの `BackendRef` には **`weight`** properties があり、Progressive deliveryをサポートします。

Argo RolloutsもGateway APIでのProgressive deliveryをサポートしています。

- https://gateway-api.sigs.k8s.io/implementations/#argo-rollouts
- https://argo-rollouts.readthedocs.io/en/stable/features/traffic-management/plugins/#gateway-API
- https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-gatewayapi/

### Service Mesh

Gateway APIはClusterの外にいるClientからClusterの中にあるServiceへのtraffic routingを管理する目的で策定されました。しかし時が経つにつれてService Meshに対する関心が高まってきており、Gateway APIを同一Cluster内のeast/west traffic routingに利用する方法を策定する目的で [GAMMA(Gateway API for Mesh Management and Administration) Initiative](https://gateway-api.sigs.k8s.io/mesh/gamma/) が立ち上がりました。

2024/05/9に [v1.1.0](https://github.com/kubernetes-sigs/gateway-api/releases/tag/v1.1.0) で [Gateway API for Service Mesh](https://gateway-api.sigs.k8s.io/mesh/) がGA Releaseされました。それに伴いIstio v1.22で [Gateway API Mesh Support Promoted To Stable](https://istio.io/latest/blog/2024/gateway-mesh-ga/) となりました。

- https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/
- https://developer.mamezou-tech.com/blogs/2022/07/24/k8s-gateway-api-intro/

### Implementations

https://gateway-api.sigs.k8s.io/implementations/

Gateway APIを実装した製品群

#### Argo Rollouts

- https://rollouts-plugin-trafficrouter-gatewayapi.readthedocs.io/en/latest/

## Cloud Provider

### AWS

- https://www.gateway-api-controller.eks.aws.dev/latest/
- [AWS Gateway API Controller for VPC Lattice](https://github.com/aws/aws-application-networking-k8s)


### Google Cloud

- https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api?hl=ja



## Traffic Management

### 参考

- https://argoproj.github.io/argo-rollouts/features/traffic-management/
- https://argoproj.github.io/argo-rollouts/features/traffic-management/alb/
- https://argoproj.github.io/argo-rollouts/getting-started/alb/
- https://github.com/argoproj/argo-rollouts/blob/v1.4.0/rollout/trafficrouting/
- https://aws.amazon.com/jp/blogs/news/using-aws-load-balancer-controller-for-blue-green-deployment-canary-deployment-and-a-b-testing/

### Traffic Managementとは

Argo Rolloutsでdeploy中の新version applicationへ流すtrafficの割合を制御できます。
[getting-started](https://argoproj.github.io/argo-rollouts/getting-started) の例でもある通り、defaultでは新version applicationと旧version applicationのService selecttorに紐付くReplicaSetのPod数を変動させることでtrafficの割合を制御します。
Argo Rolloutsの `Traffic Management` ではサービスメッシュ(`AWS ALB Ingress Controller` や `Nginx Ingress Controller` 含む) と連携することが可能です。

Traffic Managementは大きく3つの手法があります。

1. Raw percentages
    - 新versionにN%、旧versionにN% といった割合で指定する
    - ALB Ingress, Nginx Ingressも対応
        - https://github.com/argoproj/argo-rollouts/blob/v1.4.0/rollout/trafficrouting/nginx/nginx.go
        - https://github.com/argoproj/argo-rollouts/blob/v1.4.0/rollout/trafficrouting/alb/alb.go
1. Header-based routing
    - 任意のheaderがリクエストに含まれている場合は新versionに流す
    - ALB Ingress対応
        - https://github.com/argoproj/argo-rollouts/blob/v1.4.0/rollout/trafficrouting/nginx/nginx.go
        - https://github.com/argoproj/argo-rollouts/blob/v1.4.0/rollout/trafficrouting/alb/alb.go
1. Mirrored traffic
    - まったく同じtrafficを新versionにも流す。ただし、responseは無視される
    - ALB Ingress, Nginx Ingressも非対応
        - https://github.com/argoproj/argo-rollouts/blob/v1.4.0/rollout/trafficrouting/nginx/nginx.go
        - https://github.com/argoproj/argo-rollouts/blob/v1.4.0/rollout/trafficrouting/alb/alb.go


### 設定

#### ALBの場合

- https://argoproj.github.io/argo-rollouts/features/traffic-management/alb/#usage
    - Rollout
        - `spec.strategy.canary.canaryService`
        - `spec.strategy.canary.stableService`
        - `spec.strategy.canary.trafficRouting` (ServiceMeshによってparameterが異なる)

            ```
            spec:
              ...
              strategy:
                canary:
                  canaryService: canary-service
                  stableService: stable-service
                  trafficRouting:
                    alb:
                      ingress: ingress
                      servicePort: 443
            ```

    - Ingress
        - `spec.rules` 以下のruleが Rollout の `spec.strategy.canary.stableService` と一致するruleでdeployされる必要があります

            ```
            apiVersion: networking.k8s.io/v1beta1
            kind: Ingress
            metadata:
              name: ingress
              annotations:
                kubernetes.io/ingress.class: alb
            spec:
              rules:
              - http:
                  paths:
                  - path: /*
                    backend:
                      # serviceName must match either: canary.trafficRouting.alb.rootService (if specified),
                      # or canary.stableService (if rootService is omitted)
                      serviceName: stable-service
                      # servicePort must be the value: use-annotation
                      # This instructs AWS Load Balancer Controller to look to annotations on how to direct traffic
                      servicePort: use-annotation
            ```

        - rolloutがdeploy中
            - ALB Controllerが理解できるJSON payloadを含む `alb.ingress.kubernetes.io/actions.<SERVICE-NAME>` annotationをRollouts ControllerがIngressリソースに注入します
                ```
                apiVersion: networking.k8s.io/v1beta1
                kind: Ingress
                metadata:
                  name: ingress
                  annotations:
                    kubernetes.io/ingress.class: alb
                    alb.ingress.kubernetes.io/actions.stable-service: |
                      {
                        "Type":"forward",
                        "ForwardConfig":{
                          "TargetGroups":[
                            {
                                "Weight":10,
                                "ServiceName":"canary-service",
                                "ServicePort":"80"
                            },
                            {
                                "Weight":90,
                                "ServiceName":"stable-service",
                                "ServicePort":"80"
                            }
                          ]
                        }
                      }
                spec:
                  rules:
                  - http:
                      paths:
                      - path: /*
                        backend:
                          serviceName: stable-service
                          servicePort: use-annotation
                ```



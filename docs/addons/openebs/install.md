## Installing OpenEBS

1. helm でインストールする
    - https://openebs.io/docs/user-guides/installation#installation-through-helm
    - https://github.com/openebs/charts
    ```
    helm install openebs --namespace openebs openebs/openebs --create-namespace \
      --set cstor.enabled=true \
      --set jiva.enabled=true
    ```
1. StorageClassを確認する
    ```
    $ kubectl get sc
    NAME                       PROVISIONER           RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
    openebs-device             openebs.io/local      Delete          WaitForFirstConsumer   false                  28m
    openebs-hostpath           openebs.io/local      Delete          WaitForFirstConsumer   false                  28m
    openebs-jiva-csi-default   jiva.csi.openebs.io   Delete          Immediate              true                   28mS
    ```

1. OpenEBSの各Pod起動を確認する
    ```
    $ kubectl get pods -n openebs
    NAME                                            READY   STATUS    RESTARTS      AGE
    openebs-cstor-admission-server-b74f5487-djhxj   1/1     Running   0             22m
    openebs-cstor-csi-controller-0                  6/6     Running   0             21m
    openebs-cstor-csi-node-54prc                    2/2     Running   0             28m
    openebs-cstor-csi-node-g844x                    2/2     Running   0             22m
    openebs-cstor-csi-node-vrm2w                    2/2     Running   0             95s
    openebs-cstor-cspc-operator-84464fb479-vdh67    1/1     Running   0             22m
    openebs-cstor-cvc-operator-646f6f676b-5t79q     1/1     Running   0             22m
    openebs-jiva-csi-controller-0                   5/5     Running   3 (16m ago)   28m
    openebs-jiva-csi-node-2hz6w                     3/3     Running   0             110s
    openebs-jiva-csi-node-nt6jz                     3/3     Running   0             28m
    openebs-jiva-csi-node-zm49x                     3/3     Running   0             22m
    openebs-jiva-operator-f994f6868-w7xrl           1/1     Running   0             22m
    openebs-localpv-provisioner-55b65f8b55-2mq9z    1/1     Running   3 (16m ago)   28m
    openebs-ndm-4smgw                               1/1     Running   0             2m37s
    openebs-ndm-6fpg7                               1/1     Running   0             9m7s
    openebs-ndm-operator-6c944d87b6-5dq77           1/1     Running   0             22m
    openebs-ndm-vm255                               1/1     Running   0             28m
    ```

## Deploy Jenkins with OpenEBS

### 参考

- https://blog.openebs.io/tagged/jenkins
- https://github.com/openebs/openebs/tree/main/k8s/demo/jenkins

### 手順

1. download manifests
    ```
    wget https://raw.githubusercontent.com/openebs/openebs/master/k8s/demo/jenkins/jenkins.yml
    ```
1. modify manifests
    ```
    vim jenkins.yml
    ```

      ```
      $ diff -u <(curl -s https://raw.githubusercontent.com/openebs/openebs/master/k8s/demo/jenkins/jenkins.yml) <(cat ./jenkins.yml)
      --- /dev/fd/63  2022-10-01 14:11:35.968241073 +0000
      +++ /dev/fd/62  2022-10-01 14:11:35.980240892 +0000
      @@ -3,7 +3,7 @@
       metadata:
         name: jenkins-claim
         annotations:
      -    volume.beta.kubernetes.io/storage-class: openebs-jiva-default
      +    volume.beta.kubernetes.io/storage-class: openebs-jiva-csi-default
       spec:
         accessModes:
           - ReadWriteOnce
      ```
1. apply manifests
    ```
    kubectl apply -f jenkins.yml
    ```

1. リソース確認
    <details><summary>PersistentVolumeClaim</summary>
    
    ```
    $ kubectl describe PersistentVolumeClaim jenkins-claim
    Name:          jenkins-claim
    Namespace:     default
    StorageClass:  openebs-jiva-csi-default
    Status:        Bound
    Volume:        pvc-e69e2bbb-f945-4ec7-b61a-490a66596441
    Labels:        <none>
    Annotations:   pv.kubernetes.io/bind-completed: yes
                   pv.kubernetes.io/bound-by-controller: yes
                                  volume.beta.kubernetes.io/storage-class: openebs-jiva-csi-default
                                                 volume.beta.kubernetes.io/storage-provisioner: jiva.csi.openebs.io
                                                 Finalizers:    [kubernetes.io/pvc-protection]
                                                 Capacity:      5G
                                                 Access Modes:  RWO
                                                 VolumeMode:    Filesystem
                                                 Used By:       jenkins-7f87d6d6d8-htjx9
                                                 Events:        <none>
    ```
    
    </details>
    
    <details><summary>Service</summary>
    
    ```
    $ kubectl describe service jenkins-svc
    Name:                     jenkins-svc
    Namespace:                default
    Labels:                   <none>
    Annotations:              <none>
    Selector:                 app=jenkins-app
    Type:                     NodePort
    IP Family Policy:         SingleStack
    IP Families:              IPv4
    IP:                       10.32.0.70
    IPs:                      10.32.0.70
    Port:                     <unset>  80/TCP
    TargetPort:               8080/TCP
    NodePort:                 <unset>  30792/TCP
    Endpoints:                10.200.0.45:8080
    Session Affinity:         None
    External Traffic Policy:  Cluster
    Events:                   <none>
    ```
    
    </details>
    
    <details>><summary>Deployment</summary>
    
    ```
    $ kubectl describe deployment jenkins
    Name:                   jenkins
    Namespace:              default
    CreationTimestamp:      Fri, 30 Sep 2022 04:56:42 +0000
    Labels:                 <none>
    Annotations:            deployment.kubernetes.io/revision: 1
    Selector:               app=jenkins-app
    Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
    StrategyType:           RollingUpdate
    MinReadySeconds:        0
    RollingUpdateStrategy:  25% max unavailable, 25% max surge
    Pod Template:
      Labels:  app=jenkins-app
      Containers:
       jenkins:
        Image:        jenkins/jenkins:lts
        Port:         8080/TCP
        Host Port:    0/TCP
        Environment:  <none>
        Mounts:
          /var/jenkins_home from jenkins-home (rw)
      Volumes:
       jenkins-home:
        Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
        ClaimName:  jenkins-claim
        ReadOnly:   false
    Conditions:
      Type           Status  Reason
      ----           ------  ------
      Progressing    True    NewReplicaSetAvailable
      Available      True    MinimumReplicasAvailable
    OldReplicaSets:  <none>
    NewReplicaSet:   jenkins-7f87d6d6d8 (1/1 replicas created)
    Events:          <none>
    ```
    
    </details>
    
    <details><summary>Pod</summary>
    
    ```
    $ kubectl describe pods jenkins-7f87d6d6d8-htjx9
    Name:         jenkins-7f87d6d6d8-htjx9
    Namespace:    default
    Priority:     0
    Node:         k8s-master/192.168.3.50
    Start Time:   Sun, 02 Oct 2022 16:49:07 +0000
    Labels:       app=jenkins-app
                  pod-template-hash=7f87d6d6d8
    Annotations:  <none>
    Status:       Running
    IP:           10.200.0.45
    IPs:
      IP:           10.200.0.45
    Controlled By:  ReplicaSet/jenkins-7f87d6d6d8
    Containers:
      jenkins:
        Container ID:   containerd://9c039657f1a1dee2b4b480aaff1859c9385b9b9d2d874757ad6e0e6c47278bda
        Image:          jenkins/jenkins:lts
        Image ID:       docker.io/jenkins/jenkins@sha256:5508cb1317aa0ede06cb34767fb1ab3860d1307109ade577d5df871f62170214
        Port:           8080/TCP
        Host Port:      0/TCP
        State:          Running
          Started:      Sun, 02 Oct 2022 16:49:38 +0000
        Ready:          True
        Restart Count:  0
        Environment:    <none>
        Mounts:
          /var/jenkins_home from jenkins-home (rw)
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-gzs4t (ro)
    Conditions:
      Type              Status
      Initialized       True
      Ready             True
      ContainersReady   True
      PodScheduled      True
    Volumes:
      jenkins-home:
        Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
        ClaimName:  jenkins-claim
        ReadOnly:   false
      kube-api-access-gzs4t:
        Type:                    Projected (a volume that contains injected data from multiple sources)
        TokenExpirationSeconds:  3607
        ConfigMapName:           kube-root-ca.crt
        ConfigMapOptional:       <nil>
        DownwardAPI:             true
    QoS Class:                   BestEffort
    Node-Selectors:              <none>
    Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
    Events:                      <none>
    ```
    
    </details>

1. Webアクセス確認

    | image | descrioption |
    |:---|:---|
    | <img src="/addons/openebs/jenkins_01.png" style="width:300px;"/> | |
    | <img src="/addons/openebs/jenkins_02.png" style="width:300px;"/> | 表示にある通り`/var/jenkins_home/secrets/initialAdminPassword`の内容を貼り付ける<br>`kubectl exec -it jenkins-7f87d6d6d8-wx8nn -- cat /var/jenkins_home/secrets/initialAdminPassword` |
    | <img src="/addons/openebs/jenkins_03.png" style="width:300px;"/> | |
    | <img src="/addons/openebs/jenkins_04.png" style="width:300px;"/> | |
    | <img src="/addons/openebs/jenkins_05.png" style="width:300px;"/> | |
    | <img src="/addons/openebs/jenkins_06.png" style="width:300px;"/> | |
    | <img src="/addons/openebs/jenkins_07.png" style="width:300px;"/> | |
    | <img src="/addons/openebs/jenkins_08.png" style="width:300px;"/> | |
    | <img src="/addons/openebs/jenkins_09.png" style="width:300px;"/> | |
    | <img src="/addons/openebs/jenkins_10.png" style="width:300px;"/> | |
    | <img src="/addons/openebs/jenkins_11.png" style="width:300px;"/> | |


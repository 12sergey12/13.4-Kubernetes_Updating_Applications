### Домашнее задание к занятию «Обновление приложений» Баранов Сергей

### Цель задания

Выбрать и настроить стратегию обновления приложения.

### Чеклист готовности к домашнему заданию

1. Кластер K8s.

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Документация Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment).
2. [Статья про стратегии обновлений](https://habr.com/ru/companies/flant/articles/471620/).

-----

### Задание 1. Выбрать стратегию обновления приложения и описать ваш выбор

1. Имеется приложение, состоящее из нескольких реплик, которое требуется обновить.
2. Ресурсы, выделенные для приложения, ограничены, и нет возможности их увеличить.
3. Запас по ресурсам в менее загруженный момент времени составляет 20%.
4. Обновление мажорное, новые версии приложения не умеют работать со старыми.
5. Вам нужно объяснить свой выбор стратегии обновления приложения.

Ответ:

1. Если приложение уже протестировано, то лучшим вариантом будет использовать стратегию обновления Rolling Update, c указанием параметров maxSurge maxUnavailable для избежания ситуации с нехваткой ресурсов. Проводить обновление следует естественно в менее загруженный момент времени сервиса. При данной стратегии(Rolling Update) k8s постепенно заменит все поды без ущерба производительности. Если что-то пойдет не так, можно будет быстро откатится к предыдущему состоянию.

2. Можно использовать Canary Strategy. Аналогично указав параметры maxSurge maxUnavailable чтобы избежать нехватки ресурсов. Это позволит нам протестировать новую версию программы на реальной пользовательской базе(группа может выделяться по определенному признаку) без обязательства полного развертывания. После тестирования и собирания метрик пользователей можно постепенно переводить поды к новой версии приложения.


### Задание 2. Обновить приложение

1. Создать deployment приложения с контейнерами nginx и multitool. Версию nginx взять 1.19. Количество реплик — 5.

[deployments.yaml](https://github.com/12sergey12/13.4-Kubernetes_Updating_Applications/blob/main/deployments.yaml)

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netology-deployment
  labels:
    app: main
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: main
  template:
    metadata:
      labels:
        app: main
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80

      - name: network-multitool
        image: wbitt/network-multitool
        env:
        - name: HTTP_PORT
          value: "8080"
        - name: HTTPS_PORT
          value: "11443"
        ports:
        - containerPort: 8080
          name: http-port
        - containerPort: 11443
          name: https-port
        resources:
          requests:
            cpu: "1m"
            memory: "20Mi"
          limits:
            cpu: "10m"
            memory: "20Mi"
```

[svc.yaml](https://github.com/12sergey12/13.4-Kubernetes_Updating_Applications/blob/main/svc.yaml)

```
apiVersion: v1
kind: Service
metadata:
  name: mysvc
spec:
  ports:
    - name: web-nginx
      port: 9001
      targetPort: 80
    - name: web-mtools
      port: 9002
      targetPort: 8080
  selector:
    app: main
```

```
root@baranov:/home/baranovsa/kube_3.4# kubectl apply -f deployments.yaml
deployment.apps/netology-deployment created
root@baranov:/home/baranovsa/kube_3.4# 
root@baranov:/home/baranovsa/kube_3.4# kubectl apply -f svc.yaml 
service/mysvc created
root@baranov:/home/baranovsa/kube_3.4# 
```

```
root@baranov:/home/baranovsa/kube_3.4# kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
netology-deployment-666dcf88d-26k6j   2/2     Running   0          2m28s
netology-deployment-666dcf88d-49bkt   2/2     Running   0          2m28s
netology-deployment-666dcf88d-fv8tv   2/2     Running   0          2m29s
netology-deployment-666dcf88d-l52kd   2/2     Running   0          2m29s
netology-deployment-666dcf88d-xhrxp   2/2     Running   0          2m29s
root@baranov:/home/baranovsa/kube_3.4# 
root@baranov:/home/baranovsa/kube_3.4# kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP             14d
mysvc        ClusterIP   10.109.210.10   <none>        9001/TCP,9002/TCP   2m33s
root@baranov:/home/baranovsa/kube_3.4# 
root@baranov:/home/baranovsa/kube_3.4# kubectl get pod netology-deployment-666dcf88d-l52kd -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2024-04-28T14:22:14Z"
  generateName: netology-deployment-666dcf88d-
  labels:
    app: main
    pod-template-hash: 666dcf88d
  name: netology-deployment-666dcf88d-l52kd
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: netology-deployment-666dcf88d
    uid: 8692122a-0ccf-4add-9d7b-c492c4e37c95
  resourceVersion: "15576"
  uid: bdd5a617-bef8-470b-a758-f19f5a553a58
spec:
  containers:
  - image: nginx:1.19
    imagePullPolicy: IfNotPresent
    name: nginx
    ports:
    - containerPort: 80
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-sc827
      readOnly: true
  - env:
    - name: HTTP_PORT
      value: "8080"
    - name: HTTPS_PORT
      value: "11443"
    image: wbitt/network-multitool
    imagePullPolicy: Always
    name: network-multitool
    ports:
    - containerPort: 8080
      name: http-port
      protocol: TCP
    - containerPort: 11443
      name: https-port
      protocol: TCP
    resources:
      limits:
        cpu: 10m
        memory: 20Mi
      requests:
        cpu: 1m
        memory: 20Mi
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-sc827
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: minikube
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: kube-api-access-sc827
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2024-04-28T14:22:15Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2024-04-28T14:23:51Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2024-04-28T14:23:51Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2024-04-28T14:22:15Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://12a17e2362acecc460777a74c0570e8191486c3fd3a7512232afa35827911edb
    image: wbitt/network-multitool:latest
    imageID: docker-pullable://wbitt/network-multitool@sha256:d1137e87af76ee15cd0b3d4c7e2fcd111ffbd510ccd0af076fc98dddfc50a735
    lastState: {}
    name: network-multitool
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2024-04-28T14:23:47Z"
  - containerID: docker://0e29c4b4d0a92270fb681e6683b950b22db45c3a92ac7b034f4505ce42363d1e
    image: nginx:1.19
    imageID: docker-pullable://nginx@sha256:df13abe416e37eb3db4722840dd479b00ba193ac6606e7902331dcea50f4f1f2
    lastState: {}
    name: nginx
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2024-04-28T14:23:32Z"
  hostIP: 192.168.49.2
  phase: Running
  podIP: 10.244.0.146
  podIPs:
  - ip: 10.244.0.146
  qosClass: Burstable
  startTime: "2024-04-28T14:22:15Z"
root@baranov:/home/baranovsa/kube_3.4# 
root@baranov:/home/baranovsa/kube_3.4# 
root@baranov:/home/baranovsa/kube_3.4# kubectl exec deployment/netology-deployment -- curl mysvc:9002
Defaulted container "nginx" out of: nginx, network-multitool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   156  100   156    0     0    327      0 --:--:-WBITT Network MultiTool (with NGINX) - netology-deployment-666dcf88d-fv8tv - 10.244.0.148 - HTTP: 8080 , HTTPS: 11443 . (Formerly praqma/network-multitool)
- --:--:-- --:--:--   327
root@baranov:/home/baranovsa/kube_3.4# 

```


2. Обновить версию nginx в приложении до версии 1.20, сократив время обновления до минимума. Приложение должно быть доступно.

Обновляем. Меняем в deployment.yaml параметр image: nginx:1.19 на 1.20.

Выбираем и добавляем параметры стратегии обновления для того чтобы приложение было всегда доступно.

```
strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```

```
root@baranov:/home/baranovsa/kube_3.4# kubectl apply -f deployments.yaml
deployment.apps/netology-deployment configured
root@baranov:/home/baranovsa/kube_3.4# kubectl get pod -o wide
NAME                                   READY   STATUS              RESTARTS   AGE     IP             NODE       NOMINATED NODE   READINESS GATES
netology-deployment-566fcf8d84-7jq9f   0/2     ContainerCreating   0          3s      <none>         minikube   <none>           <none>
netology-deployment-566fcf8d84-s2x75   0/2     ContainerCreating   0          3s      <none>         minikube   <none>           <none>
netology-deployment-666dcf88d-26k6j    0/2     Terminating         0          8m32s   <none>         minikube   <none>           <none>
netology-deployment-666dcf88d-49bkt    2/2     Running             0          8m32s   10.244.0.149   minikube   <none>           <none>
netology-deployment-666dcf88d-fv8tv    2/2     Running             0          8m33s   10.244.0.148   minikube   <none>           <none>
netology-deployment-666dcf88d-l52kd    2/2     Running             0          8m33s   10.244.0.146   minikube   <none>           <none>
netology-deployment-666dcf88d-xhrxp    2/2     Running             0          8m33s   10.244.0.147   minikube   <none>           <none>
root@baranov:/home/baranovsa/kube_3.4#

root@baranov:/home/baranovsa/kube_3.4#  kubectl exec deployment/netology-deployment -- curl mysvc:9002
Defaulted container "nginx" out of: nginx, network-multitool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   15WBITT Network MultiTool (with NGINX) - netology-deployment-566fcf8d84-x6tf6 - 10.244.0.153 - HTTP: 8080 , HTTPS: 11443 . (Formerly praqma/network-multitool)
7  100   157    0     0   1891      0 --:--:-- --:--:-- --:--:--  1891
root@baranov:/home/baranovsa/kube_3.4# kubectl get pod -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP             NODE       NOMINATED NODE   READINESS GATES
netology-deployment-566fcf8d84-5mwl7   2/2     Running   0          64s   10.244.0.155   minikube   <none>           <none>
netology-deployment-566fcf8d84-72x6q   2/2     Running   0          78s   10.244.0.154   minikube   <none>           <none>
netology-deployment-566fcf8d84-7jq9f   2/2     Running   0          94s   10.244.0.151   minikube   <none>           <none>
netology-deployment-566fcf8d84-s2x75   2/2     Running   0          94s   10.244.0.152   minikube   <none>           <none>
netology-deployment-566fcf8d84-x6tf6   2/2     Running   0          80s   10.244.0.153   minikube   <none>           <none>
root@baranov:/home/baranovsa/kube_3.4# 
```

```
root@baranov:/home/baranovsa/kube_3.4# kubectl describe deployment netology-deployment
Name:                   netology-deployment
Namespace:              default
CreationTimestamp:      Sun, 28 Apr 2024 21:22:13 +0700
Labels:                 app=main
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=main
Replicas:               5 desired | 5 updated | 5 total | 5 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
Pod Template:
  Labels:  app=main
  Containers:
   nginx:
    Image:        nginx:1.20
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
   network-multitool:
    Image:       wbitt/network-multitool
    Ports:       8080/TCP, 11443/TCP
    Host Ports:  0/TCP, 0/TCP
    Limits:
      cpu:     10m
      memory:  20Mi
    Requests:
      cpu:     1m
      memory:  20Mi
    Environment:
      HTTP_PORT:   8080
      HTTPS_PORT:  11443
    Mounts:        <none>
  Volumes:         <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  netology-deployment-666dcf88d (0/0 replicas created)
NewReplicaSet:   netology-deployment-566fcf8d84 (5/5 replicas created)
Events:
  Type    Reason             Age                From                   Message
  ----    ------             ----               ----                   -------
  Normal  ScalingReplicaSet  10m                deployment-controller  Scaled up replica set netology-deployment-666dcf88d to 5
  Normal  ScalingReplicaSet  119s               deployment-controller  Scaled up replica set netology-deployment-566fcf8d84 to 1
  Normal  ScalingReplicaSet  119s               deployment-controller  Scaled down replica set netology-deployment-666dcf88d to 4 from 5
  Normal  ScalingReplicaSet  119s               deployment-controller  Scaled up replica set netology-deployment-566fcf8d84 to 2 from 1
  Normal  ScalingReplicaSet  106s               deployment-controller  Scaled down replica set netology-deployment-666dcf88d to 3 from 4
  Normal  ScalingReplicaSet  105s               deployment-controller  Scaled up replica set netology-deployment-566fcf8d84 to 3 from 2
  Normal  ScalingReplicaSet  103s               deployment-controller  Scaled down replica set netology-deployment-666dcf88d to 2 from 3
  Normal  ScalingReplicaSet  103s               deployment-controller  Scaled up replica set netology-deployment-566fcf8d84 to 4 from 3
  Normal  ScalingReplicaSet  89s                deployment-controller  Scaled down replica set netology-deployment-666dcf88d to 1 from 2
  Normal  ScalingReplicaSet  88s (x2 over 89s)  deployment-controller  (combined from similar events): Scaled down replica set netology-deployment-666dcf88d to 0 from 1
root@baranov:/home/baranovsa/kube_3.4# 
```

3. Попытаться обновить nginx до версии 1.28, приложение должно оставаться доступным.

Меняем в deployments.yaml параметр image: nginx:1.20 на 1.28.

```
kubectl apply -f deployments.yaml 
deployment.apps/netology-deployment configured
root@baranov:/home/baranovsa/kube_3.4# 
root@baranov:/home/baranovsa/kube_3.4# kubectl get pod
NAME                                   READY   STATUS             RESTARTS   AGE
netology-deployment-566fcf8d84-5mwl7   2/2     Running            0          4m28s
netology-deployment-566fcf8d84-72x6q   2/2     Running            0          4m42s
netology-deployment-566fcf8d84-s2x75   2/2     Running            0          4m58s
netology-deployment-566fcf8d84-x6tf6   2/2     Running            0          4m44s
netology-deployment-768978d8d4-dv46q   1/2     ImagePullBackOff   0          17s
netology-deployment-768978d8d4-w46dg   1/2     ImagePullBackOff   0          17s
root@baranov:/home/baranovsa/kube_3.4#
```
при этом

```
root@baranov:/home/baranovsa/kube_3.4# kubectl exec deployment/netology-deployment -- curl mysvc:9002
Defaulted container "nginx" out of: nginx, network-multitool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   157  100   157    0     0  13083      0 --:--:-- --:--:-- --:--:-- 31400
WBITT Network MultiTool (with NGINX) - netology-deployment-566fcf8d84-s2x75 - 10.244.0.152 - HTTP: 8080 , HTTPS: 11443 . (Formerly praqma/network-multitool)
root@baranov:/home/baranovsa/kube_3.4# 
```


4. Откатиться после неудачного обновления.

```
root@baranov:/home/baranovsa/kube_3.4# kubectl rollout status deployment netology-deployment
Waiting for deployment "netology-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
root@baranov:/home/baranovsa/kube_3.4# kubectl rollout undo deployment netology-deployment
deployment.apps/netology-deployment rolled back
root@baranov:/home/baranovsa/kube_3.4# 
root@baranov:/home/baranovsa/kube_3.4# 
root@baranov:/home/baranovsa/kube_3.4# kubectl get pod
NAME                                   READY   STATUS    RESTARTS   AGE
netology-deployment-566fcf8d84-5mwl7   2/2     Running   0          7m11s
netology-deployment-566fcf8d84-72x6q   2/2     Running   0          7m25s
netology-deployment-566fcf8d84-gfh5j   2/2     Running   0          25s
netology-deployment-566fcf8d84-s2x75   2/2     Running   0          7m41s
netology-deployment-566fcf8d84-x6tf6   2/2     Running   0          7m27s
root@baranov:/home/baranovsa/kube_3.4#
```

```
root@baranov:/home/baranovsa/kube_3.4# kubectl exec deployment/netology-deployment -- curl mysvc:9002
Defaulted container "nginx" out of: nginx, network-multitool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100WBITT Network MultiTool (with NGINX) - netology-deployment-566fcf8d84-gfh5j - 10.244.0.158 - HTTP: 8080 , HTTPS: 11443 . (Formerly praqma/network-multitool)
   157  100   157    0     0  22428      0 --:--:-- --:--:-- --:--:-- 22428
root@baranov:/home/baranovsa/kube_3.4# 
root@baranov:/home/baranovsa/kube_3.4# 
root@baranov:/home/baranovsa/kube_3.4# kubectl describe deployment netology-deployment
Name:                   netology-deployment
Namespace:              default
CreationTimestamp:      Sun, 28 Apr 2024 21:22:13 +0700
Labels:                 app=main
Annotations:            deployment.kubernetes.io/revision: 4
Selector:               app=main
Replicas:               5 desired | 5 updated | 5 total | 5 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
Pod Template:
  Labels:  app=main
  Containers:
   nginx:
    Image:        nginx:1.20
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
   network-multitool:
    Image:       wbitt/network-multitool
    Ports:       8080/TCP, 11443/TCP
    Host Ports:  0/TCP, 0/TCP
    Limits:
      cpu:     10m
      memory:  20Mi
    Requests:
      cpu:     1m
      memory:  20Mi
    Environment:
      HTTP_PORT:   8080
      HTTPS_PORT:  11443
    Mounts:        <none>
  Volumes:         <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  netology-deployment-666dcf88d (0/0 replicas created), netology-deployment-768978d8d4 (0/0 replicas created)
NewReplicaSet:   netology-deployment-566fcf8d84 (5/5 replicas created)
Events:
  Type    Reason             Age                  From                   Message
  ----    ------             ----                 ----                   -------
  Normal  ScalingReplicaSet  17m                  deployment-controller  Scaled up replica set netology-deployment-666dcf88d to 5
  Normal  ScalingReplicaSet  8m50s                deployment-controller  Scaled up replica set netology-deployment-566fcf8d84 to 1
  Normal  ScalingReplicaSet  8m50s                deployment-controller  Scaled down replica set netology-deployment-666dcf88d to 4 from 5
  Normal  ScalingReplicaSet  8m50s                deployment-controller  Scaled up replica set netology-deployment-566fcf8d84 to 2 from 1
  Normal  ScalingReplicaSet  8m37s                deployment-controller  Scaled down replica set netology-deployment-666dcf88d to 3 from 4
  Normal  ScalingReplicaSet  8m36s                deployment-controller  Scaled up replica set netology-deployment-566fcf8d84 to 3 from 2
  Normal  ScalingReplicaSet  8m34s                deployment-controller  Scaled down replica set netology-deployment-666dcf88d to 2 from 3
  Normal  ScalingReplicaSet  8m34s                deployment-controller  Scaled up replica set netology-deployment-566fcf8d84 to 4 from 3
  Normal  ScalingReplicaSet  8m20s                deployment-controller  Scaled down replica set netology-deployment-666dcf88d to 1 from 2
  Normal  ScalingReplicaSet  94s (x6 over 8m20s)  deployment-controller  (combined from similar events): Scaled down replica set netology-deployment-768978d8d4 to 0 from 2
  Normal  ScalingReplicaSet  94s                  deployment-controller  Scaled up replica set netology-deployment-566fcf8d84 to 5 from 4
root@baranov:/home/baranovsa/kube_3.4# 

```

## Дополнительные задания — со звёздочкой*

Задания дополнительные, необязательные к выполнению, они не повлияют на получение зачёта по домашнему заданию. **Но мы настоятельно рекомендуем вам выполнять все задания со звёздочкой.** Это поможет лучше разобраться в материале.   

### Задание 3*. Создать Canary deployment

1. Создать два deployment'а приложения nginx.
2. При помощи разных ConfigMap сделать две версии приложения — веб-страницы.
3. С помощью ingress создать канареечный деплоймент, чтобы можно было часть трафика перебросить на разные версии приложения.

### Правила приёма работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

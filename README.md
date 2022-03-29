# h-taxi

## Autoscaling

- reference: https://labs.msaez.io/#/courses/Container-full/Hanwha-ContainerContainer-Orchestration/Manual-and-AutoScaling

1. Autoscaling 테스트를 위한 k8s pod 생성

```yaml
# h-taxi-grap.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: h-taxi-grap
spec:
  selector:
    matchLabels:
      run: h-taxi-grap
  replicas: 1
  template:
    metadata:
      labels:
        run: h-taxi-grap
    spec:
      containers:
        - name: h-taxi-grap
          image: devgony/h-taxi-grap:stable
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: 500m
            requests:
              cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: h-taxi-grap
  labels:
    run: h-taxi-grap
spec:
  ports:
    - port: 80
  selector:
    run: h-taxi-grap
```

```
kubectl apply -f h-taxi-grap.yaml
kubectl get all
```

```
NAME                               READY   STATUS    RESTARTS   AGE
pod/h-taxi-grap-75f877867b-xz5cd   1/1     Running   0          8s

NAME                  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/h-taxi-grap   ClusterIP   10.40.4.25   <none>        80/TCP    8s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/h-taxi-grap   1/1     1            1           8s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/h-taxi-grap-75f877867b   1         1         1       8s
```

2. Autoscale 설정 및 horizontalpodautoscaler, hpa 확인

```
kubectl autoscale deployment h-taxi-grap --cpu-percent=50 --min=1 --max=10
kubectl get horizontalpodautoscaler
```

```
NAME          REFERENCE                TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
h-taxi-grap   Deployment/h-taxi-grap   <unknown>/50%   1         10        0          14s
```

```
kubectl get hpa
```

```
NAME          REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
h-taxi-grap   Deployment/h-taxi-grap   0%/50%    1         10        1          68s
```

3. 로드 제너레이터 설치

```
kubectl apply -f siege.yaml
kubectl get pod siege
```

```
NAME    READY   STATUS    RESTARTS   AGE
siege   1/1     Running   0          2m16s
```

4. `siege` pod 내에서 부하 발생

```
kubectl exec -it siege -- /bin/bash
siege> siege -c30 -t30S -v http://h-taxi-grap
```

5. `h-taxi-grap` pod 에 대한 모니터링 수행

- `siege` 부하 발생 전 pod 상태

```
watch -d -n 1 kubectl get pod
```

```
Every 1.0s: kubectl get pod                                    labs-1676586095: Mon Mar 28 04:32:58 2022

NAME                           READY   STATUS    RESTARTS   AGE
h-taxi-grap-75f877867b-xz5cd   1/1     Running   0          10m
siege                          1/1     Running   0          6m20s
```

- `siege` 부하 발생 후 CPU usage 50%이상 증가한 것을 grapana 통해 확인
![image](https://user-images.githubusercontent.com/51254761/160510381-c00a076e-e5f7-4476-ab45-d28d57a58a0b.png)

- `siege` 부하 발생 후 pod 상태 확인

```
Every 1.0s: kubectl get pod                                    labs-1676586095: Mon Mar 28 04:35:02 2022

NAME                           READY   STATUS              RESTARTS   AGE
h-taxi-grap-75f877867b-gw5sn   1/1     Running             0          8s
h-taxi-grap-75f877867b-lj5lv   0/1     ContainerCreating   0          8s
h-taxi-grap-75f877867b-xz5cd   1/1     Running             0          12m
h-taxi-grap-75f877867b-zpv46   0/1     ContainerCreating   0          8s
siege                          1/1     Running             0          8m24s
```

  - Autoscaling 되어 pod 의 개수가 4개로 늘어나고 있는 것이 확인 됌

## Zero-downtime deploy

- reference: https://labs.msaez.io/#/courses/cna-full/f9dfa020-8d5d-11ec-818c-6d8ef2017004/ops-readiness

1. `h-taxi-grap` deploy

```yaml
# deploy-h-taxi-grap.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: h-taxi-grap
  labels:
    app: h-taxi-grap
spec:
  replicas: 1
  selector:
    matchLabels:
      app: h-taxi-grap
  template:
    metadata:
      labels:
        app: h-taxi-grap
    spec:
      containers:
        - name: h-taxi-grap
          image: devgony/h-taxi-grap:stable
          ports:
            - containerPort: 8080
```

```
kubectl apply -f deploy-h-taxi-grap.yaml
```

2. `h-taxi-grap` service

```yaml
# service-h-taxi-grap.yaml
apiVersion: "v1"
kind: "Service"
metadata:
  name: "h-taxi-grap"
  labels:
    app: "h-taxi-grap"
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: "h-taxi-grap"
  type: "ClusterIP"
```

```
kubectl apply -f service-h-taxi-grap.yaml
```

3. 부하테스트 `siege` pod 설치

```yaml
# siege.yaml
apiVersion: v1
kind: Pod
metadata:
  name: siege
spec:
  containers:
    - name: siege
      image: apexacme/siege-nginx
```

4. `siege` 내부에서 부하 수행

```
kubectl exec -it siege -- /bin/bash
siege> siege -c1 -t60S -v http://h-taxi-grap:8080/grap --delay=1S
```

5. `stable` -> `canary` 버전으로 수정 후 재 배포

```diff
# deploy-h-taxi-grap.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: h-taxi-grap
  labels:
    app: h-taxi-grap
spec:
  replicas: 1
  selector:
    matchLabels:
      app: h-taxi-grap
  template:
    metadata:
      labels:
        app: h-taxi-grap
    spec:
      containers:
        - name: h-taxi-grap
-         image: devgony/h-taxi-grap:stable
+         image: devgony/h-taxi-grap:canary
...
```

```
kubectl apply -f deploy-h-taxi-grap.yaml
```

6. `siege` 결과 일부만 성공(80.35%)하고 나머지는 배포시 중단 된 것을 확인

```diff
siege>
Lifting the server siege...
Transactions:                   2368 hits
-Availability:                  80.35 %
Elapsed time:                  59.49 secs
Data transferred:               0.80 MB
Response time:                  0.02 secs
Transaction rate:              39.81 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                    0.79
Successful transactions:        2368
Failed transactions:             579
Longest transaction:            0.75
Shortest transaction:           0.00
```

7. readinessProbe 추가, `canary` -> `stable` 버전 변경

```diff
# deploy-h-taxi-grap.yaml
spec:
  replicas: 1
  selector:
    matchLabels:
      app: h-taxi-grap
  template:
    metadata:
      labels:
        app: h-taxi-grap
    spec:
      containers:
        - name: h-taxi-grap
-         image: devgony/h-taxi-grap:canary
+         image: devgony/h-taxi-grap:stable
          ports:
            - containerPort: 8080
+         readinessProbe:
+           httpGet:
+             path: "/grap"
+             port: 8080
+           initialDelaySeconds: 10
+           timeoutSeconds: 2
+           periodSeconds: 5
+           failureThreshold: 10
```

8. 부하 재 발생 및 무중단 배포 테스트

```
siege> siege -c1 -t60S -v http://h-taxi-grap:8080/grap --delay=1S
```

```
kubectl apply -f deploy-h-taxi-grap.yaml
```

```diff
[error] socket: unable to connect sock.c:249: Connection reset by peer
...
siege>
Lifting the server siege...
Transactions:                   2928 hits
+Availability:                  99.83 %
Elapsed time:                  59.05 secs
Data transferred:               0.99 MB
Response time:                  0.00 secs
Transaction rate:              49.59 trans/sec
Throughput:                     0.02 MB/sec
Concurrency:                    0.23
Successful transactions:        2928
Failed transactions:               5
Longest transaction:            0.04
Shortest transaction:           0.00
```

- readinessProbe 설정을 통해 99.83%에 달하는 Availability를 보여주는 것을 확인 가능
- 100%에 달하는 무중단 배포 이지만 부하 테스트의 delay가 짧아 socket에러 일부 발생하여 0.17% 낮아진 수치

## Self Healing

- reference: https://labs.msaez.io/#/courses/Container-full/Hanwha-ContainerContainer-Orchestration/Liveness-ReadinessProbe

1. livenessProbe 설정을 추가한 이미지 yaml 파일 작성

```yaml
# h-taxi-grap-liveness.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: h-taxi-grap
  labels:
    app: h-taxi-grap
spec:
  replicas: 1
  selector:
    matchLabels:
      app: h-taxi-grap
  template:
    metadata:
      labels:
        app: h-taxi-grap
    spec:
      containers:
        - name: h-taxi-grap
          image: devgony/h-taxi-grap-liveness:latest
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: "/actuator/health"
              port: 8080
            initialDelaySeconds: 15
            timeoutSeconds: 2
            successThreshold: 1
            periodSeconds: 5
            failureThreshold: 3
```

2. yaml 파일 적용 후 LoadBalancer 타입으로 배포

```
kubectl apply -f h-taxi-grap-liveness.yaml
kubectl expose deploy h-taxi-grap --type=LoadBalanc포r --port=8080
kubectl get svc
```

```
NAME          TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
h-taxi-grap   LoadBalancer   10.40.2.145   <pending>     8080:31282/TCP   6ss
```

3. 해당 서비스의 health 확인

```
http 10.40.2.145:8080/actuator/health
```

```diff
HTTP/1.1 200
Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8
Date: Mon, 28 Mar 2022 06:53:49 GMT
Transfer-Encoding: chunked

{
+    "status": "UP"
}
```

4. 서비스 down

```
http put 10.40.2.145:8080/actuator/down
```

```diff
HTTP/1.1 200
Content-Type: application/json;charset=UTF-8
Date: Mon, 28 Mar 2022 06:54:30 GMT
Transfer-Encoding: chunked

{
-    "status": "DOWN"
}
```

5. 서비스 down 이후에도 여전히 pod가 Running 중임을 확인

```
kubectl get pod
```

```
NAME                          READY   STATUS    RESTARTS   AGE
h-taxi-grap-95cb5c959-cq4th   1/1     Running   1          2m51s
```

6. 해당 pod 세부 로그에서 unhealthy 발견 후 self-killing & healing 수행한 것을 확인 가능

```
kubectl describe pod/h-taxi-grap-95cb5c959-cq4th
```

```diff
...
Name:         h-taxi-grap-95cb5c959-cq4th
Namespace:    labs-1676586095
Priority:     0
Node:         gke-cluster-2-default-pool-a1811fce-bfcl/10.146.15.213
Start Time:   Mon, 28 Mar 2022 06:52:10 +0000
Labels:       app=h-taxi-grap
              pod-template-hash=95cb5c959
Annotations:  <none>
Status:       Running
IP:           10.36.5.6
IPs:
  IP:           10.36.5.6
Controlled By:  ReplicaSet/h-taxi-grap-95cb5c959
Containers:
  h-taxi-grap:
    Container ID:   docker://0f59d6c23a68a81817b9e7d81fee3b0656c7c64c32b08db2429c37d18848dc1a
    Image:          devgony/h-taxi-grap-liveness:latest
    Image ID:       docker-pullable://devgony/h-taxi-grap-liveness@sha256:76c7e59956ad1411c08de4fb50df44ab72c01053eaf96646491eda9797dd8734
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 28 Mar 2022 06:54:48 +0000
    Last State:     Terminated
      Reason:       Error
      Exit Code:    143
      Started:      Mon, 28 Mar 2022 06:52:19 +0000
      Finished:     Mon, 28 Mar 2022 06:54:45 +0000
    Ready:          True
    Restart Count:  1
    Liveness:       http-get http://:8080/actuator/health delay=15s timeout=2s period=5s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-8nhg5 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-8nhg5:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute for 300s
                             node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason     Age                  From                                               Message
  ----     ------     ----                 ----                                               -------
  Normal   Scheduled  <unknown>                                                               Successfully assigned labs-1676586095/h-taxi-grap-95cb5c959-cq4th to gke-cluster-2-default-pool-a1811fce-bfcl
  Normal   Pulled     3m27s                kubelet, gke-cluster-2-default-pool-a1811fce-bfcl  Successfully pulled image "devgony/h-taxi-grap-liveness:latest" in 5.954773255s
  Normal   Pulling    59s (x2 over 3m33s)  kubelet, gke-cluster-2-default-pool-a1811fce-bfcl  Pulling image "devgony/h-taxi-grap-liveness:latest"
-  Warning  Unhealthy  59s (x3 over 69s)    kubelet, gke-cluster-2-default-pool-a1811fce-bfcl  Liveness probe failed: HTTP probe failed with statuscode: 503
-  Normal   Killing    59s                  kubelet, gke-cluster-2-default-pool-a1811fce-bfcl  Container h-taxi-grap failed liveness probe, will be restarted
+  Normal   Created    56s (x2 over 3m25s)  kubelet, gke-cluster-2-default-pool-a1811fce-bfcl  Created container h-taxi-grap
+  Normal   Started    56s (x2 over 3m25s)  kubelet, gke-cluster-2-default-pool-a1811fce-bfcl  Started container h-taxi-grap
+  Normal   Pulled     56s                  kubelet, gke-cluster-2-default-pool-a1811fce-bfcl  Successfully pulled image "devgony/h-taxi-grap-liveness:latest" in 2.704500252s
```

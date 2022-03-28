# h-taxi

## Autoscaling

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

```
watch -d -n 1 kubectl get pod
```

```
Every 1.0s: kubectl get pod                                    labs-1676586095: Mon Mar 28 04:32:58 2022

NAME                           READY   STATUS    RESTARTS   AGE
h-taxi-grap-75f877867b-xz5cd   1/1     Running   0          10m
siege                          1/1     Running   0          6m20s
```

```
Every 1.0s: kubectl get pod                                    labs-1676586095: Mon Mar 28 04:35:02 2022

NAME                           READY   STATUS              RESTARTS   AGE
h-taxi-grap-75f877867b-gw5sn   1/1     Running             0          8s
h-taxi-grap-75f877867b-lj5lv   0/1     ContainerCreating   0          8s
h-taxi-grap-75f877867b-xz5cd   1/1     Running             0          12m
h-taxi-grap-75f877867b-zpv46   0/1     ContainerCreating   0          8s
siege                          1/1     Running             0          8m24s
```

- Autoscaling 되어 pod 의 개수가 4개로 늘어나고 있는 것을 확인

## Zero-downtime deploy

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

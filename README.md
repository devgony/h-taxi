# h-taxi

## Autoscaling

1. Autoscaling 테스트를 위한 k8s pod 생성

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

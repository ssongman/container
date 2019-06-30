



# ...







## HorizontalPodAutoscaler(HPA, 수평적 오토 스케일링)

오토스케일링의 프로세스의 3단계

1. 스케일된 리소스 객체가 관리하는 모든 파드의 메트릭 수집
2. 지정된 목표 값에 메트릭을 가져오는데 필요한 파드 수 계산
3. 스케일된 리소스의 복제본 필드 업데이트



오토스케일링이 동작하려면 클러스터에서 힙스터가 실행 중이어야 한다.



## 2. CPU 사용률에 따른 스케일링

오토스케일링에서 가장 중요한 측정 기준은 파드내부에서 실행되는 프로세스가 소비하는 CPU 사용량이다.

CPU 사용량이 100% 에 이르면 더이상 요구에 대처할 수 없고 사용가능한 CPU 를 늘리거나, 파드의 수를 늘리는 방법의 스케일링이 필요하다.   파드 수의 증가가 수평적스케일링에 해당한다.

> 갑작스런 피크 부하를 처리할 수 있는 충분한 공간을 확봄하려면 항상 CPU 사용량을 100% 으로 설정해야 한다.  절대 90% 를 넘어서는 안된다.



### 2.1 userlist deployment

```yaml
# userlist-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: userlist
  labels:
    app: userlist
spec:
  replicas: 1
  selector:
    matchLabels:
      app: userlist
  template:
    metadata:
      labels:
        app: userlist
    spec:
      containers:
      - name: userlist
        image: docker.io/ssongman/userlist:v1
        ports:
        - containerPort: 8181
        resources:
          requests:
            cpu: 100m     ## pod당 100밀리코어의 CPU를 요구
---
```







### 2.2 오토스케일링 명령

```sh
$ kubectl autoscale deployment userlist --cpu-percent=80 --min1 --max=5
```





### 2.3 yaml 파일로 처리



```yaml
### userlist-hpa.yaml
apiVersion: "autoscaling/v2beta1"
kind: "HorizontalPodAutoscaler"
metadata:
  name: "userlist-hpa"
  namespace: "dev-song"
  labels:
    app: "userlist"
spec:
  scaleTargetRef:
    kind: "Deployment"
    name: "userlist"
    apiVersion: "apps/v1beta1"
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: "Resource"
    resource:
      name: "cpu"
      targetAverageUtilization: 80
```

- hpa 의 이름은 디플로이먼트이름과 일치할 필요는 없다.
- scaleTargetRef : 오토스케일러가 처리할 대상 리소스 지정







## 3. 메모리 소바량에 기반한 스케일링


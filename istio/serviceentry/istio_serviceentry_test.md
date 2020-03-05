# istio serviceentry test





# 1. 테스트개요

istio pod 에서 외부 Server(DB등) 로 향하는 트래픽의 모습에서 serviceEntry  존재여부에 따른 상태 점검



## 1) 사전 확인

- outboundTrafficPolicy  ALLOW_ANY mode 임을 확인

```
oc -n istio-system get cm istio -o yaml
...
outboundTrafficPolicy:
  mode: ALLOW_ANY
...
```

> ※ istio configmap 의 outboundTrafficPolicy mode 설명
>
> ALLOW_ANY : istio 트래픽에서 외부로 나가는 트래픽을 모두 허용한다.
>
> REGISTRY_ONLY : Service 또는 ServiceEntry 도 등록된 외부 트래픽만 허용한다.



- Permissive 모드인지 확인하자.

```
oc get meshpolicy -o yaml
---
apiVersion: v1
items:
- apiVersion: authentication.istio.io/v1alpha1
  kind: MeshPolicy
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"authentication.istio.io/v1alpha1","kind":"MeshPolicy","metadata":{"annotations":{},"labels":{"app":"security","chart":"security","heritage":"Tiller","release":"istio"},"name":"default","namespace":""},"spec":{"peers":[{"mtls":{"mode":"PERMISSIVE"}}]}}
    creationTimestamp: 2020-02-03T15:24:26Z
    generation: 1
    labels:
      app: security
      chart: security
      heritage: Tiller
      release: istio
    name: default
    namespace: ""
    resourceVersion: "5112347"
    selfLink: /apis/authentication.istio.io/v1alpha1/meshpolicies/default
    uid: 4396147b-4699-11ea-99bf-080027380859
  spec:
    peers:
    - mtls:
        mode: PERMISSIVE    <-- 확인
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```





# 2. 테스트시나리오

아래와 같이 3가지 시나리오로 테스트 진행한다.

1. non istio - DB 
2. istio - DB
3. istio - DB (serviceentry)





# 3. 테스트
## 3.1. 테스트를 위한 App 설치

Client 와 Server 를 각각 설치하는데 모두 Postgresql 이다. 




### 1) Server(DB) 설치

서버는 Cluster 외부에 설치되어야 하므로 특정 노드(여기서는 4번 node)에 Docker 로 설치 한다.

```
[root@okd-node04 ~]# ping okd-node04.192-168-0-229.nip.io
PING okd-node04.192-168-0-229.nip.io (192.168.0.229) 56(84) bytes of data.

[root@okd-node04 ~]#  docker run --name syj-postgres -e POSTGRES_PASSWORD=postpass -d -p5432:5432 postgres
29a2821fc9b4b2314bde84d775c38337fc88cad024f0cb1772460285965e650d
```



### 2) Client 설치

Client 는 Cluster 내부에 설치해야 하므로 Service 와 Deployment 를 설치한다. 일반적인 테스트일때는 Service 가 필요없지만 istio inject 의 경우는 Service가 필수이다.

- service
```yaml
cat 12.postgres-svc.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  labels:
    app: postgres
    service: postgres
spec:
  ports:
  - port: 5432
    name: http
  selector:
    app: postgres

```

- deployment
```yaml
cat 13.postgres-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-cli
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
      annotations
        sidecar.istio.io/inject: "false"
    spec:
      containers:
      - name: postgres
        image: postgres
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_HOST_AUTH_METHOD
          value: trust
```





# 3.2 시나리오 테스트





### 1) 시나리오 1번

non istio - DB 연결

- DB Connect Test

```bash
$ psql -h okd-node04.192-168-0-229.nip.io -p 5432 -U postgres
Password for user postgres:
psql (12.2 (Debian 12.2-2.pgdg100+1))
Type "help" for help.

postgres=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}


```

연결성공







### 2) 시나리오 2번

istio - DB 연결



- istio inject 실시

```yaml
cat 13.postgres-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-cli
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
      annotations:
        sidecar.istio.io/inject: "true"
    spec:
      containers:
      - name: postgres
        image: postgres
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_HOST_AUTH_METHOD
          value: trust
```



- 확인

```bash
oc -n song get pod
NAME                            READY     STATUS    RESTARTS   AGE
postgres-cli-5cf947dcd7-vch5v   2/2       Running   0          36s
```



- DB Connect Test

```bash
$ psql -h 192.168.0.229 -p 5432 -U postgres
psql: error: could not connect to server: server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.
```

연결불가







### 3) 시나리오 3번

istio - DB (serviceEntry) 연결



####  (1) serviceEntry 등록

```yaml
$ cat > 15.external-svc-postgres.yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-postgres
spec:
  addresses:
    - 192.168.0.229/24
  hosts:
    - external.svc.postgres
  ports:
    - name: postgresdb
      number: 5432
      protocol: TCP
---
```

- 항목설명
  - host : 필수 값이긴 하나 naming 으로 역할 수행.
    - TCP 일 경우 indicator 역할 수행(특별하게 의미 없음)
    - 호스트 필드는 VirtualServices 및 DestinationRules에서 일치하는 호스트를 선택하는 데 사용됨
    - HTTP 트래픽의 경우 HTTP Host / Authority 헤더가 hosts 필드와 일치함.
  - addresses : 실제 접속되는 ip 주소



- DB Connect Test

```bash
$ psql -h okd-node04.192-168-0-229.nip.io -p 5432 -U postgres
Password for user postgres:
psql (12.2 (Debian 12.2-2.pgdg100+1))
Type "help" for help.

postgres=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}


$ psql -h 192.168.0.229 -p 5432 -U postgres 
- 성공
```

연결성공







####  (2) serviceEntry 등록 - location

```yaml
$ cat > 15.external-svc-postgres.yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-postgres
spec:
  addresses:
    - 192.168.0.229/24
  hosts:
    - external.svc.postgres
  location: MESH_EXTERNAL
  ports:
    - name: postgresdb
      number: 5432
      protocol: TCP
---
```

- location : 서비스가 메쉬내부인지 외부인지를 명시한다. 외부일때는 mTLS 인증이 disable 되며 정책수행이 클라이언트에서 수행된다.
  - MESH_INTERNAL  / MESH_EXTERNAL
  - internal / external / 생략 해도 모두 테스트 성공임.



- DB Connect Test

```bash
$ psql -h okd-node04.192-168-0-229.nip.io -p 5432 -U postgres
Password for user postgres:
psql (12.2 (Debian 12.2-2.pgdg100+1))
Type "help" for help.

postgres=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}



$ psql -h 192.168.0.229 -p 5432 -U postgres 
- 성공
```

연결성공





####  (3) serviceEntry  등록 - endponit



```yaml
$ cat > 15.external-svc-postgres.yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-postgres
spec:
  addresses:
    - 192.168.0.229/24
  endpoints:
    - address: 192.168.0.229
  hosts:
    - external.svc.postgres
  ports:
    - name: postgresdb
      number: 5432
      protocol: TCP
  resolution: STATIC
---
```

- resolution : 라우팅방법 결정
  - STATIC : endpoint 로 static IP 를 지정



- DB Connect Test

```bash
$ psql -h okd-node04.192-168-0-229.nip.io -p 5432 -U postgres
Password for user postgres:
psql (12.2 (Debian 12.2-2.pgdg100+1))
Type "help" for help.

postgres=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}

postgres=#
```

연결성공





주의할점 : serviceEntry 가 pod 보다 먼저 실행되어야 정책이 반영됨.  즉, serviceentry 가 변경되면 pod 는 재기동되어야 함.
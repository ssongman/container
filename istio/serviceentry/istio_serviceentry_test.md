# istio ServiceEntry test





# 1. 테스트 개요

istio pod 에서 외부 Server(DB등) 로 향하는 트래픽의 모습에서 serviceEntry  존재여부에 따른 상태를 점검한다.

기본적으로 istio configmap 의 outboundTrafficPolicy mode 가 ALLOW_ANY 일 경우 별도의 serviceEntry 가 없어도 외부 연결하는데 지장이 없다.  하지만 일부 프로젝트(MMP)에서 사용하는 postgresql DB 의 경우 반드시 ServiceEntry 가 존재해야지만 연결이 되었다. 이런 사례를을 분석하고자 테스트를 실시한다.



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
        mode: PERMISSIVE                 <---- 확인
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```





# 2. Postgresql 테스트



## 2.1. 테스트시나리오

아래와 같이 3가지 시나리오로 테스트 진행한다.

1. non istio - DB 
2. istio - DB
3. istio - DB (serviceEntry)

serviceEntry 가 pod 보다 먼저 실행되어야 정책이 반영됨.  즉, serviceentry 가 변경되면 pod 는 재기동되어야 함.





## 2.2. 테스트를 위한 App 설치

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
      annotations:
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





## 2.3 시나리오 테스트



### 1) 시나리오 1번 - 성공

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



### 2) 시나리오 2번 - 실패

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

postgresql log

```
[root@okd-node04 ~]# docker logs -f syj-postgres
...
2020-03-06 00:43:29.926 UTC [1] LOG:  starting PostgreSQL 12.2 (Debian 12.2-2.pgdg100+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 8.3.0-6) 8.3.0, 64-bit
2020-03-06 00:43:29.927 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2020-03-06 00:43:29.927 UTC [1] LOG:  listening on IPv6 address "::", port 5432
2020-03-06 00:43:29.929 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2020-03-06 00:43:29.939 UTC [56] LOG:  database system was shut down at 2020-03-06 00:43:29 UTC
2020-03-06 00:43:29.942 UTC [1] LOG:  database system is ready to accept connections
2020-03-06 01:40:45.313 UTC [65] LOG:  invalid length of startup packet
2020-03-06 01:40:59.416 UTC [66] LOG:  invalid length of startup packet
2020-03-06 01:43:04.638 UTC [67] LOG:  invalid length of startup packet
2020-03-06 02:31:16.934 UTC [71] LOG:  invalid length of startup packet
2020-03-06 02:31:25.051 UTC [72] LOG:  invalid length of startup packet
```

invalid length of startup packet  로 반응함.



- tcpdump

```

14:10:10.103721 IP 192.168.0.217.45744 > okd-node04.192-168-0-229.nip.io.postgres: Flags [S], seq 2822401932, win 28200, options [mss 1410,sackOK,TS val 2728076349 ecr 0,nop,wscale 7], length 0
14:10:10.103794 IP 192.168.0.217.45744 > 172.17.0.2.postgres: Flags [S], seq 2822401932, win 28200, options [mss 1410,sackOK,TS val 2728076349 ecr 0,nop,wscale 7], length 0
14:10:10.103799 IP 192.168.0.217.45744 > 172.17.0.2.postgres: Flags [S], seq 2822401932, win 28200, options [mss 1410,sackOK,TS val 2728076349 ecr 0,nop,wscale 7], length 0

14:10:10.103880 IP 172.17.0.2.postgres > 192.168.0.217.45744: Flags [S.], seq 1091572995, ack 2822401933, win 27960, options [mss 1410,sackOK,TS val 2728092033 ecr 2728076349,nop,wscale 7], length 0
14:10:10.103880 IP 172.17.0.2.postgres > 192.168.0.217.45744: Flags [S.], seq 1091572995, ack 2822401933, win 27960, options [mss 1410,sackOK,TS val 2728092033 ecr 2728076349,nop,wscale 7], length 0

14:10:10.103918 IP okd-node04.192-168-0-229.nip.io.postgres > 192.168.0.217.45744: Flags [S.], seq 1091572995, ack 2822401933, win 27960, options [mss 1410,sackOK,TS val 2728092033 ecr 2728076349,nop,wscale 7], length 0
14:10:10.108037 IP 192.168.0.217.45744 > okd-node04.192-168-0-229.nip.io.postgres: Flags [.], ack 1, win 221, options [nop,nop,TS val 2728076452 ecr 2728092033], length 0
14:10:10.108061 IP 192.168.0.217.45744 > 172.17.0.2.postgres: Flags [.], ack 1, win 221, options [nop,nop,TS val 2728076452 ecr 2728092033], length 0
14:10:10.108065 IP 192.168.0.217.45744 > 172.17.0.2.postgres: Flags [.], ack 1, win 221, options [nop,nop,TS val 2728076452 ecr 2728092033], length 0

14:10:10.108068 IP 192.168.0.217.45744 > okd-node04.192-168-0-229.nip.io.postgres: Flags [P.], seq 1:240, ack 1, win 221, options [nop,nop,TS val 2728076452 ecr 2728092033], length 239
14:10:10.108071 IP 192.168.0.217.45744 > 172.17.0.2.postgres: Flags [P.], seq 1:240, ack 1, win 221, options [nop,nop,TS val 2728076452 ecr 2728092033], length 239
14:10:10.108072 IP 192.168.0.217.45744 > 172.17.0.2.postgres: Flags [P.], seq 1:240, ack 1, win 221, options [nop,nop,TS val 2728076452 ecr 2728092033], length 239

14:10:10.108119 IP 172.17.0.2.postgres > 192.168.0.217.45744: Flags [.], ack 240, win 227, options [nop,nop,TS val 2728092037 ecr 2728076452], length 0
14:10:10.108119 IP 172.17.0.2.postgres > 192.168.0.217.45744: Flags [.], ack 240, win 227, options [nop,nop,TS val 2728092037 ecr 2728076452], length 0

14:10:10.108127 IP okd-node04.192-168-0-229.nip.io.postgres > 192.168.0.217.45744: Flags [.], ack 240, win 227, options [nop,nop,TS val 2728092037 ecr 2728076452], length 0

14:10:10.109897 IP 172.17.0.2.postgres > 192.168.0.217.45744: Flags [F.], seq 1, ack 240, win 227, options [nop,nop,TS val 2728092039 ecr 2728076452], length 0
14:10:10.109897 IP 172.17.0.2.postgres > 192.168.0.217.45744: Flags [F.], seq 1, ack 240, win 227, options [nop,nop,TS val 2728092039 ecr 2728076452], length 0
14:10:10.109913 IP okd-node04.192-168-0-229.nip.io.postgres > 192.168.0.217.45744: Flags [F.], seq 1, ack 240, win 227, options [nop,nop,TS val 2728092039 ecr 2728076452], length 0
14:10:10.113302 IP 192.168.0.217.45744 > okd-node04.192-168-0-229.nip.io.postgres: Flags [F.], seq 240, ack 2, win 221, options [nop,nop,TS val 2728076457 ecr 2728092039], length 0
14:10:10.113312 IP 192.168.0.217.45744 > 172.17.0.2.postgres: Flags [F.], seq 240, ack 2, win 221, options [nop,nop,TS val 2728076457 ecr 2728092039], length 0
14:10:10.113314 IP 192.168.0.217.45744 > 172.17.0.2.postgres: Flags [F.], seq 240, ack 2, win 221, options [nop,nop,TS val 2728076457 ecr 2728092039], length 0
14:10:10.113333 IP 172.17.0.2.postgres > 192.168.0.217.45744: Flags [.], ack 241, win 227, options [nop,nop,TS val 2728092042 ecr 2728076457], length 0
14:10:10.113333 IP 172.17.0.2.postgres > 192.168.0.217.45744: Flags [.], ack 241, win 227, options [nop,nop,TS val 2728092042 ecr 2728076457], length 0
14:10:10.113342 IP okd-node04.192-168-0-229.nip.io.postgres > 192.168.0.217.45744: Flags [.], ack 241, win 227, options [nop,nop,TS val 2728092042 ecr 2728076457], length 0

```

실패의 경우 length 239 이다.





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



- tcpdump

```

14:08:45.356884 IP 192.168.0.217.44060 > okd-node04.192-168-0-229.nip.io.postgres: Flags [S], seq 3870950725, win 28200, options [mss 1410,sackOK,TS val 2727991670 ecr 0,nop,wscale 7], length 0
14:08:45.356923 IP 192.168.0.217.44060 > 172.17.0.2.postgres: Flags [S], seq 3870950725, win 28200, options [mss 1410,sackOK,TS val 2727991670 ecr 0,nop,wscale 7], length 0
14:08:45.356928 IP 192.168.0.217.44060 > 172.17.0.2.postgres: Flags [S], seq 3870950725, win 28200, options [mss 1410,sackOK,TS val 2727991670 ecr 0,nop,wscale 7], length 0

14:08:45.356930 IP 192.168.0.217.44060 > 172.17.0.2.postgres: Flags [S], seq 3870950725, win 28200, options [mss 1410,sackOK,TS val 2727991670 ecr 0,nop,wscale 7], length 0

14:08:45.356973 IP 172.17.0.2.postgres > 192.168.0.217.44060: Flags [S.], seq 4162906650, ack 3870950726, win 27960, options [mss 1410,sackOK,TS val 2728007286 ecr 2727991670,nop,wscale 7], length 0
14:08:45.356973 IP 172.17.0.2.postgres > 192.168.0.217.44060: Flags [S.], seq 4162906650, ack 3870950726, win 27960, options [mss 1410,sackOK,TS val 2728007286 ecr 2727991670,nop,wscale 7], length 0

14:08:45.356985 IP okd-node04.192-168-0-229.nip.io.postgres > 192.168.0.217.44060: Flags [S.], seq 4162906650, ack 3870950726, win 27960, options [mss 1410,sackOK,TS val 2728007286 ecr 2727991670,nop,wscale 7], length 0
14:08:45.360725 IP 192.168.0.217.44060 > okd-node04.192-168-0-229.nip.io.postgres: Flags [.], ack 1, win 221, options [nop,nop,TS val 2727991705 ecr 2728007286], length 0
14:08:45.360734 IP 192.168.0.217.44060 > 172.17.0.2.postgres: Flags [.], ack 1, win 221, options [nop,nop,TS val 2727991705 ecr 2728007286], length 0
14:08:45.360736 IP 192.168.0.217.44060 > 172.17.0.2.postgres: Flags [.], ack 1, win 221, options [nop,nop,TS val 2727991705 ecr 2728007286], length 0

14:08:45.360737 IP 192.168.0.217.44060 > okd-node04.192-168-0-229.nip.io.postgres: Flags [P.], seq 1:9, ack 1, win 221, options [nop,nop,TS val 2727991705 ecr 2728007286], length 8
14:08:45.360739 IP 192.168.0.217.44060 > 172.17.0.2.postgres: Flags [P.], seq 1:9, ack 1, win 221, options [nop,nop,TS val 2727991705 ecr 2728007286], length 8
14:08:45.360740 IP 192.168.0.217.44060 > 172.17.0.2.postgres: Flags [P.], seq 1:9, ack 1, win 221, options [nop,nop,TS val 2727991705 ecr 2728007286], length 8

14:08:45.360771 IP 172.17.0.2.postgres > 192.168.0.217.44060: Flags [.], ack 9, win 219, options [nop,nop,TS val 2728007290 ecr 2727991705], length 0
14:08:45.360771 IP 172.17.0.2.postgres > 192.168.0.217.44060: Flags [.], ack 9, win 219, options [nop,nop,TS val 2728007290 ecr 2727991705], length 0

14:08:45.372140 IP 192.168.0.217.44060 > okd-node04.192-168-0-229.nip.io.postgres: Flags [.], ack 16, win 221, options [nop,nop,TS val 2727991716 ecr 2728007298], length 0

```

성공의 경우 length 8 이다.





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





####  (4) serviceEntry Protocol: HTTP - 실패

protocol 을tcp 대신 http 로 변경했을때 tcpdump 에는 어떤 모습이 나오는지 확인해 보자.

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
      protocol: HTTP             <------ 변경
---
```



- DB Connect Test

```bash
$ psql -h 192.168.0.229 -p 5432 -U postgres
psql: error: could not connect to server: server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.

```

역시나 연결 불가



- tcpdump

```

14:20:59.930731 IP 192.168.0.228.50732 > okd-node04.192-168-0-229.nip.io.postgres: Flags [S], seq 1180260197, win 28200, options [mss 1410,sackOK,TS val 2728733729 ecr 0,nop,wscale 7], length 0
14:20:59.930784 IP 192.168.0.228.50732 > 172.17.0.2.postgres: Flags [S], seq 1180260197, win 28200, options [mss 1410,sackOK,TS val 2728733729 ecr 0,nop,wscale 7], length 0
14:20:59.930787 IP 192.168.0.228.50732 > 172.17.0.2.postgres: Flags [S], seq 1180260197, win 28200, options [mss 1410,sackOK,TS val 2728733729 ecr 0,nop,wscale 7], length 0
14:20:59.930834 IP 172.17.0.2.postgres > 192.168.0.228.50732: Flags [S.], seq 1101595117, ack 1180260198, win 27960, options [mss 1410,sackOK,TS val 2728741860 ecr 2728733729,nop,wscale 7], length 0
14:20:59.930834 IP 172.17.0.2.postgres > 192.168.0.228.50732: Flags [S.], seq 1101595117, ack 1180260198, win 27960, options [mss 1410,sackOK,TS val 2728741860 ecr 2728733729,nop,wscale 7], length 0
14:20:59.930845 IP okd-node04.192-168-0-229.nip.io.postgres > 192.168.0.228.50732: Flags [S.], seq 1101595117, ack 1180260198, win 27960, options [mss 1410,sackOK,TS val 2728741860 ecr 2728733729,nop,wscale 7], length 0
14:20:59.931098 IP 192.168.0.228.50732 > okd-node04.192-168-0-229.nip.io.postgres: Flags [.], ack 1, win 221, options [nop,nop,TS val 2728733729 ecr 2728741860], length 0
14:20:59.931105 IP 192.168.0.228.50732 > 172.17.0.2.postgres: Flags [.], ack 1, win 221, options [nop,nop,TS val 2728733729 ecr 2728741860], length 0
14:20:59.931107 IP 192.168.0.228.50732 > 172.17.0.2.postgres: Flags [.], ack 1, win 221, options [nop,nop,TS val 2728733729 ecr 2728741860], length 0
14:20:59.931173 IP 192.168.0.228.50732 > okd-node04.192-168-0-229.nip.io.postgres: Flags [P.], seq 1:240, ack 1, win 221, options [nop,nop,TS val 2728733729 ecr 2728741860], length 239
14:20:59.931178 IP 192.168.0.228.50732 > 172.17.0.2.postgres: Flags [P.], seq 1:240, ack 1, win 221, options [nop,nop,TS val 2728733729 ecr 2728741860], length 239
14:20:59.931179 IP 192.168.0.228.50732 > 172.17.0.2.postgres: Flags [P.], seq 1:240, ack 1, win 221, options [nop,nop,TS val 2728733729 ecr 2728741860], length 239
14:20:59.931191 IP 172.17.0.2.postgres > 192.168.0.228.50732: Flags [.], ack 240, win 227, options [nop,nop,TS val 2728741860 ecr 2728733729], length 0
14:20:59.931191 IP 172.17.0.2.postgres > 192.168.0.228.50732: Flags [.], ack 240, win 227, options [nop,nop,TS val 2728741860 ecr 2728733729], length 0
14:20:59.931196 IP okd-node04.192-168-0-229.nip.io.postgres > 192.168.0.228.50732: Flags [.], ack 240, win 227, options [nop,nop,TS val 2728741860 ecr 2728733729], length 0

```

실패했을때와 동일한 length 239 이다. 

혹시 serviceEntry 가 없을때  http 통신을 하는건가? -- 확실치 않음.





## 2.3 Clean up

테스트시 사용했던 리소스들 모두 초기화 하자.

```bash
oc -n song delete deployment postgres-cli

oc -n song delete svc postgres-svc

oc -n song delete serviceentry external-svc-postgres

[root@okd-node04 ~]#  docker rm -f syj-postgres
```





## 2.4 테스트 결론

- outboundTrafficPolicy mode 가 Allow_any 로 되어 있으면 ServieEntry 가 없어도 외부 트래픽을 허용해야 하지만 Postgresql DB 의 경우 serviceEntry 가 있어야지만 정상 통신이 됨.

- 실패 상태의 tcpdump detail 로 로그 분석 결과

  tcpdump -i any -XX  port 5433 and host 192.168.0.216

```
16:57:35.540929 IP 192.168.0.228.42640 > okd-node04.192-168-0-229.nip.io.postgres: Flags [P.], seq 1:241, ack 1, win 221, options [nop,nop,TS val 2738129338 ecr 2738137469], length 240
        0x0000:  0000 0001 0006 0800 2798 4b23 0000 0800  ........'.K#....
        0x0010:  4500 0124 ca45 4000 3f06 ed74 c0a8 00e4  E..$.E@.?..t....
        0x0020:  c0a8 00e5 a690 1538 7cbc 43fa 5419 8d5b  .......8|.C.T..[
        0x0030:  8018 00dd 30ef 0000 0101 080a a334 89ba  ....0........4..
        0x0040:  a334 a97d 1603 0100 eb01 0000 e703 0377  .4.}...........w
        0x0050:  64ec f8e1 f2f8 3df5 5eec 4726 5a9a 4726  d.....=.^.G&Z.G&
        0x0060:  05e1 ce4f ccdb 88d9 cb40 5660 5058 b900  ...O.....@V`PX..
        0x0070:  001c c02b cca9 c02f cca8 c009 c013 009c  ...+.../........
        0x0080:  002f c02c c030 c00a c014 009d 0035 0100  ./.,.0.......5..
        0x0090:  00a2 0000 005d 005b 0000 586f 7574 626f  .....].[..Xoutbo
        0x00a0:  756e 645f 2e35 3433 325f 2e5f 2e70 7974  und_.5432_._.pyt
        0x00b0:  686f 6e2d 7465 7374 362d 706f 7374 6772  hon-test6-postgr
        0x00c0:  6573 716c 2d61 7273 656e 616c 2d68 6561  esql-arsenal-hea
        0x00d0:  646c 6573 732e 7079 7468 6f6e 7465 7374  dless.pythontest
        0x00e0:  322e 7376 632e 636c 7573 7465 722e 6c6f  2.svc.cluster.lo
        0x00f0:  6361 6c00 1700 00ff 0100 0100 000a 0006  cal.............
        0x0100:  0004 001d 0017 000b 0002 0100 0023 0000  .............#..
        0x0110:  0010 0008 0006 0569 7374 696f 000d 0014  .......istio....
        0x0120:  0012 0403 0804 0401 0503 0805 0501 0806  ................
        0x0130:  0601 0201 0000 0000 0000 0000 0000 0000  ................
        0x0140:  0000 0000                                ....

위 내용을 분석해 보면 아래와 같다.
.....].[..Xoutbound_.5432_._.python-test6-postgresql-arsenal-headless.pythontest2.svc.cluster.local

```

위 내용처럼 본 서비스와 전혀 무관한 python-test6-postgresql-arsenal-headless.pythontest2.svc.cluster.local 의 주소가 찍혔다.  pythontest2 namespace 에 존재하는 postgresql 서버를 가르키고 있는데 아마도 5432 port 를 이용해서 서비스를 찾은것 같다.

확실치는 않지만 아래와 같이 가설을 세워본다.

istio 에서 외부통신 시도시 serviceEntry 가 없을때 istio 내부통신으로 간주하고 내부에서 목적지 Service 를 찾음.

실제로 외부 DB 서버의 port를 5432 가 아닌 5433 으로 실행 후 connect 시도하니  잘 수행됨.

이 가설이 맞다면 istio bug 로 보임.



### 최종결론

- istio pod 에서 outbound traffic 이 있을때는 서비스를 제대로 찾지 못하는 bug 존재함.

- 해결책은 serviceEntry 를 등록해 줘야 한다.







# 3. Mysql 테스트



## 3.1 테스트 시나리오

아래와 같이 3가지 시나리오로 테스트 진행한다.

1. non istio - DB 
2. istio - DB
3. istio - DB (serviceEntry)



## 3.2. 테스트를 위한 App 설치

Client 와 Server 를 각각 mysql  설치한다.




### 1) Server(DB) 설치

서버는 Cluster 외부에 설치되어야 하므로 특정 노드(여기서는 4번 node)에 Docker 로 설치 한다.

```
[root@okd-node04 ~]#  docker run -d -p 3307:3306 \
  -e MYSQL_ALLOW_EMPTY_PASSWORD=true \
  --name mysql_syj \
  mysql:5.7
33bb490d37b67e4db4bf4245f18531168c955991da89765270305250bba90312
```



### 2) Client 설치

Client 는 Cluster 내부에 설치해야 하므로 Service 와 Deployment 를 설치한다. 일반적인 테스트일때는 Service 가 필요없지만 istio inject 의 경우는 Service가 필수이다.

- service

```yaml
cat > 22.mysql-svc.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
  labels:
    app: mysql
    service: mysql
spec:
  ports:
  - port: 3306
    name: http
  selector:
    app: mysql

```

- deployment

```yaml
cat > 23.mysql-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-cli
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
      annotations:
        sidecar.istio.io/inject: "false"
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: songsong
---
          
env 명시하지 않으면 나오는 에러들...
2020-03-06 02:09:49+00:00 [ERROR] [Entrypoint]: Database is uninitialized and password option is not specified
	You need to specify one of MYSQL_ROOT_PASSWORD, MYSQL_ALLOW_EMPTY_PASSWORD and MYSQL_RANDOM_ROOT_PASSWORD
          
```







## 3.3 시나리오 테스트



### 1) 시나리오 1번 - 성공

non istio - DB 연결

- DB Connect Test

```bash
$ mysql -h okd-node04.192-168-0-229.nip.io -P 3307 -u root
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 10
Server version: 5.7.29 MySQL Community Server (GPL)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.01 sec)


$ mysql -h 192.168.0.229 -P 3307  -u root
역시 성공
```

연결성공



- tcpdump

tcpdump -i any port 3307

```
14:26:45.723247 IP 192.168.0.217.44584 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [S], seq 3373573094, win 28200, options [mss 1410,sackOK,TS val 2729072065 ecr 0,nop,wscale 7], length 0
14:26:45.723377 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.217.44584: Flags [S.], seq 1488617022, ack 3373573095, win 27960, options [mss 1410,sackOK,TS val 2729087652 ecr 2729072065,nop,wscale 7], length 0
14:26:45.726899 IP 192.168.0.217.44584 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [.], ack 1, win 221, options [nop,nop,TS val 2729072071 ecr 2729087652], length 0

14:26:45.727239 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.217.44584: Flags [P.], seq 1:79, ack 1, win 219, options [nop,nop,TS val 2729087656 ecr 2729072071], length 78
14:26:45.731766 IP 192.168.0.217.44584 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [.], ack 79, win 221, options [nop,nop,TS val 2729072075 ecr 2729087656], length 0
14:26:45.732635 IP 192.168.0.217.44584 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [P.], seq 1:37, ack 79, win 221, options [nop,nop,TS val 2729072075 ecr 2729087656], length 36

14:26:45.732654 IP 192.168.0.217.44584 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [P.], seq 37:235, ack 79, win 221, options [nop,nop,TS val 2729072075 ecr 2729087656], length 198
14:26:45.732686 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.217.44584: Flags [.], ack 37, win 219, options [nop,nop,TS val 2729087662 ecr 2729072075], length 0
14:26:45.732698 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.217.44584: Flags [.], ack 235, win 227, options [nop,nop,TS val 2729087662 ecr 2729072075], length 0
14:26:45.738643 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.217.44584: Flags [P.], seq 79:2101, ack 235, win 227, options [nop,nop,TS val 2729087667 ecr 2729072075], length 2022
14:26:45.748000 IP 192.168.0.217.44584 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [.], ack 2101, win 252, options [nop,nop,TS val 2729072088 ecr 2729087667], length 0
14:26:45.748063 IP 192.168.0.217.44584 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [P.], seq 235:340, ack 2101, win 252, options [nop,nop,TS val 2729072088 ecr 2729087667], length 105
14:26:45.749587 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.217.44584: Flags [P.], seq 2101:2343, ack 340, win 227, options [nop,nop,TS val 2729087678 ecr 2729072088], length 242
14:26:45.757172 IP 192.168.0.217.44584 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [P.], seq 340:534, ack 2343, win 274, options [nop,nop,TS val 2729072101 ecr 2729087678], length 194
14:26:45.757402 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.217.44584: Flags [P.], seq 2343:2383, ack 534, win 236, options [nop,nop,TS val 2729087686 ecr 2729072101], length 40
14:26:45.762189 IP 192.168.0.217.44584 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [P.], seq 534:600, ack 2383, win 274, options [nop,nop,TS val 2729072106 ecr 2729087686], length 66
14:26:45.762649 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.217.44584: Flags [P.], seq 2383:2504, ack 600, win 236, options [nop,nop,TS val 2729087691 ecr 2729072106], length 121
14:26:45.808436 IP 192.168.0.217.44584 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [.], ack 2504, win 274, options [nop,nop,TS val 2729072152 ecr 2729087691], length 0

```



### 2) 시나리오 2번 - 성공

istio - DB 연결



- istio inject 실시

```yaml
cat > 23.mysql-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-cli
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
      annotations:
        sidecar.istio.io/inject: "true"
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: songsong
```



- 확인

```bash
oc -n song get pod
NAME                            READY     STATUS    RESTARTS   AGE
mysql-cli-56959cbd66-srrrl      2/2       Running   0          11m

```



- DB Connect Test

```bash
$ mysql -h okd-node04.192-168-0-229.nip.io -P 3307 -u root
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 11
Server version: 5.7.29 MySQL Community Server (GPL)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

```

연결 성공



- tcpdump

tcpdump -i any port 3307

```
14:23:14.316489 IP 192.168.0.228.54352 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [S], seq 778020401, win 28200, options [mss 1410,sackOK,TS val 2728868114 ecr 0,nop,wscale 7], length 0
14:23:14.316593 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.54352: Flags [S.], seq 437886330, ack 778020402, win 27960, options [mss 1410,sackOK,TS val 2728876245 ecr 2728868114,nop,wscale 7], length 0
14:23:14.316755 IP 192.168.0.228.54352 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [.], ack 1, win 221, options [nop,nop,TS val 2728868115 ecr 2728876245], length 0

14:23:14.316949 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.54352: Flags [P.], seq 1:79, ack 1, win 219, options [nop,nop,TS val 2728876246 ecr 2728868115], length 78

14:23:14.317093 IP 192.168.0.228.54352 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [.], ack 79, win 221, options [nop,nop,TS val 2728868115 ecr 2728876246], length 0
14:23:14.317597 IP 192.168.0.228.54352 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [P.], seq 1:37, ack 79, win 221, options [nop,nop,TS val 2728868116 ecr 2728876246], length 36

14:23:14.317647 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.54352: Flags [.], ack 37, win 219, options [nop,nop,TS val 2728876247 ecr 2728868116], length 0

14:23:14.317738 IP 192.168.0.228.54352 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [P.], seq 37:235, ack 79, win 221, options [nop,nop,TS val 2728868116 ecr 2728876246], length 198
14:23:14.317804 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.54352: Flags [.], ack 235, win 227, options [nop,nop,TS val 2728876247 ecr 2728868116], length 0
14:23:14.323657 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.54352: Flags [P.], seq 79:2101, ack 235, win 227, options [nop,nop,TS val 2728876253 ecr 2728868116], length 2022
14:23:14.323860 IP 192.168.0.228.54352 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [.], ack 2101, win 252, options [nop,nop,TS val 2728868122 ecr 2728876253], length 0
14:23:14.324583 IP 192.168.0.228.54352 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [P.], seq 235:340, ack 2101, win 252, options [nop,nop,TS val 2728868123 ecr 2728876253], length 105
14:23:14.324826 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.54352: Flags [P.], seq 2101:2343, ack 340, win 227, options [nop,nop,TS val 2728876254 ecr 2728868123], length 242
14:23:14.325146 IP 192.168.0.228.54352 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [P.], seq 340:534, ack 2343, win 274, options [nop,nop,TS val 2728868123 ecr 2728876254], length 194
14:23:14.325260 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.54352: Flags [P.], seq 2343:2383, ack 534, win 236, options [nop,nop,TS val 2728876254 ecr 2728868123], length 40
14:23:14.325563 IP 192.168.0.228.54352 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [P.], seq 534:600, ack 2383, win 274, options [nop,nop,TS val 2728868124 ecr 2728876254], length 66
14:23:14.325716 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.54352: Flags [P.], seq 2383:2504, ack 600, win 236, options [nop,nop,TS val 2728876255 ecr 2728868124], length 121
14:23:14.365616 IP 192.168.0.228.54352 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [.], ack 2504, win 274, options [nop,nop,TS val 2728868164 ecr 2728876255], length 0

```





### 3) 시나리오 3번

istio - DB (serviceEntry) 연결

#### (1) protocol: TCP - 성공

```yaml
$ cat > 25.external-svc-mysql.yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-mysql
spec:
  addresses:
    - 192.168.0.229/24
  hosts:
    - external.svc.mysql
  ports:
    - name: mysqldb
      number: 3307
      protocol: TCP
---
```



- DB Connect Test

```bash
$ mysql -h okd-node04.192-168-0-229.nip.io -P 3307 -u root
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 22
Server version: 5.7.29 MySQL Community Server (GPL)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

연결성공



- tcpdump

tcpdump -i any port 3307

```
14:47:12.554297 IP 192.168.0.228.48430 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [S], seq 1798557758, win 28200, options [mss 1410,sackOK,TS val 2730306352 ecr 0,nop,wscale 7], length 0
14:47:12.554409 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.48430: Flags [S.], seq 850557955, ack 1798557759, win 27960, options [mss 1410,sackOK,TS val 2730314483 ecr 2730306352,nop,wscale 7], length 0
14:47:12.554636 IP 192.168.0.228.48430 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [.], ack 1, win 221, options [nop,nop,TS val 2730306353 ecr 2730314483], length 0
14:47:12.554929 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.48430: Flags [P.], seq 1:79, ack 1, win 219, options [nop,nop,TS val 2730314484 ecr 2730306353], length 78
14:47:12.555094 IP 192.168.0.228.48430 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [.], ack 79, win 221, options [nop,nop,TS val 2730306353 ecr 2730314484], length 0
14:47:12.560053 IP 192.168.0.228.48430 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [P.], seq 1:235, ack 79, win 221, options [nop,nop,TS val 2730306358 ecr 2730314484], length 234
14:47:12.560093 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.48430: Flags [.], ack 235, win 227, options [nop,nop,TS val 2730314489 ecr 2730306358], length 0
14:47:12.566051 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.48430: Flags [P.], seq 79:2101, ack 235, win 227, options [nop,nop,TS val 2730314493 ecr 2730306358], length 2022
14:47:12.566300 IP 192.168.0.228.48430 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [.], ack 2101, win 252, options [nop,nop,TS val 2730306365 ecr 2730314493], length 0
14:47:12.567151 IP 192.168.0.228.48430 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [P.], seq 235:340, ack 2101, win 252, options [nop,nop,TS val 2730306365 ecr 2730314493], length 105
14:47:12.567441 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.48430: Flags [P.], seq 2101:2343, ack 340, win 227, options [nop,nop,TS val 2730314496 ecr 2730306365], length 242
14:47:12.567815 IP 192.168.0.228.48430 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [P.], seq 340:534, ack 2343, win 274, options [nop,nop,TS val 2730306366 ecr 2730314496], length 194
14:47:12.567909 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.48430: Flags [P.], seq 2343:2383, ack 534, win 236, options [nop,nop,TS val 2730314497 ecr 2730306366], length 40
14:47:12.568273 IP 192.168.0.228.48430 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [P.], seq 534:600, ack 2383, win 274, options [nop,nop,TS val 2730306366 ecr 2730314497], length 66
14:47:12.568688 IP okd-node04.192-168-0-229.nip.io.opsession-prxy > 192.168.0.228.48430: Flags [P.], seq 2383:2504, ack 600, win 236, options [nop,nop,TS val 2730314498 ecr 2730306366], length 121
14:47:12.609772 IP 192.168.0.228.48430 > okd-node04.192-168-0-229.nip.io.opsession-prxy: Flags [.], ack 2504, win 274, options [nop,nop,TS val 2730306408 ecr 2730314498], length 0

```





#### (2) protocol: HTTP - 실패

http 통신일때 tcpdump 가 어떻게 나오는지 확인해 보자.

```yaml
$ cat > 25.external-svc-mysql.yaml
---
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-mysql
spec:
  addresses:
    - 192.168.0.229/24
  hosts:
    - external.svc.mysql
  ports:
    - name: mysqldb
      number: 3307
      protocol: HTTP
---
```



- DB Connect Test

```bash
$ mysql -h okd-node04.192-168-0-229.nip.io -P 3307 -u root
반응없음
```

역시나 실패

반응없으므로 tcpdump 도 없음



## 3.3 Clean up

테스트시 사용했던 리소스들 모두 초기화 하자.

```bash
oc -n song delete deployment mysql-cli

oc -n song delete svc mysql-svc

oc -n song delete serviceentry external-svc-mysql

[root@okd-node04 ~]#  docker rm -f mysql_syj
```





## 3.4 테스트 결론

- outboundTrafficPolicy mode 가 Allow_any 로 되어 있으면 ServieEntry 가 없어도 외부 트래픽을 허용함.
- 역시 mysql DB 도 ServiceEntry 없이도 잘 됨.
- 만약 동일한 port 를 사용하는 mysql 서버가 클러스터 내에 존재한다면 동일한 오류를 일으킬 가능성 존재할 것으로 추정됨.




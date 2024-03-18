# NFS Hostname Test

천안 재해시 탄방으로 NFS 방향을 돌리기 위한 적절한 방안을 찾기 위해 분석해 본다.





# 1. NFS 저장공간 확인



## 1) 테스트 공간 확인



```

sa-devpilot-beta2-pv

 nfs:
  server: 10.217.139.23
  path: /SA_COMMON/devpilot_beta2


```





## 2) 샘플 PV



```sh

apiVersion: v1
kind: PersistentVolume
metadata:
  name: sa-song-nfs-test-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 20Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: sa-devpilot-beta2-pvc
    namespace: devpilot
  nfs:
    path: /SA_COMMON/songpvtest
    server: 10.217.139.23
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem
status:
  phase: Bound


```





# 2. NFS Mount



## 1) Service



### (1) Headless service

```sh

$ cd ~/song/del/sa-song-nfs-test

$ cat > 10.sa-song-nfs-svc.yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: sa-song-nfs-test
  name: songpvtest
  namespace: sa-test
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: nfs
    port: 2049
    protocol: TCP
    targetPort: 2049
  - name: mountd
    port: 20048
    protocol: TCP
    targetPort: 20048
  - name: rpcbind
    port: 111
    protocol: TCP
    targetPort: 111
---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    app: sa-song-nfs-test
  name: songpvtest
  namespace: sa-test
subsets:
- addresses:
  - ip: 10.217.139.23
  ports:
  - name: nfs
    port: 2049
    protocol: TCP
  - name: mountd
    port: 20048
    protocol: TCP
  - name: rpcbind
    port: 111
    protocol: TCP
---



$ kubectl -n sa-test apply -f 10.sa-song-nfs-svc.yaml

# 삭제시...
$ kubectl -n sa-test delete -f 10.sa-song-nfs-svc.yaml


```







### (2) externalName

```sh

$ cd ~/song/del/sa-song-nfs-test

$ cat > 10.sa-song-nfs-svc.yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: sa-song-nfs-test
  name: songpvtest
  namespace: sa-test
spec:
  type: ExternalName
  #externalName: my.database.example.com
  externalName: 10.217.139.23
---
  
  

$ kubectl -n sa-test apply -f 10.sa-song-nfs-svc.yaml

# 삭제시...
$ kubectl -n sa-test delete -f 10.sa-song-nfs-svc.yaml


```















### 확인

```sh

$ nc -zv songpvtest.sa-test.svc.cluster.local 2049
$ nc -zv songpvtest.sa-test.svc.cluster.local 20048
$ nc -zv songpvtest.sa-test.svc.cluster.local 111

```







## 2) deploy

```sh
$ cd ~/song/del/sa-song-nfs-test

$ cat > 11.sa-song-nfs-test.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sa-song-nfs-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sa-song-nfs-test
      elogging: enabled
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: "false"
      creationTimestamp: null
      labels:
        app: sa-song-nfs-test
        elogging: enabled
    spec:
      containers:
      - image: nexus.dspace.kt.co.kr/icis/userlist:v1
        imagePullPolicy: IfNotPresent
        name: userlist
        ports:
        - containerPort: 8181
          protocol: TCP
        volumeMounts:
        - mountPath: /app/songpvtest
          name: sa-song-nfs-test          
        securityContext:
          runAsUser: 0
          privileged: true
      volumes:
      - name: sa-song-nfs-test
        nfs:
          path: /SA_COMMON/songpvtest/
          #server: 10.217.139.23
          server: songpvtest.sa-test.svc.cluster.local
---


$ kubectl -n sa-test apply -f 11.sa-song-nfs-test.yaml

# 삭제시...
$ kubectl -n sa-test delete -f 11.sa-song-nfs-test.yaml

```






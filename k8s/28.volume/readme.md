



# volume





## volume 의 종류

| Temp     | Local    | Network                                                      |
| -------- | -------- | ------------------------------------------------------------ |
| emptyDir | hostPath | GlusterFS      gitRepo      NFS      iSCSI      gcePersistentDisk      AWS EBS      azureDisk      Fiber Channel      Secret      VshereVolume |





## emptyDir

emptyDir은 Pod가 생성될때 생성되고, Pod가 삭제 될때 같이 삭제되는 임시 볼륨이다. 

단 Pod 내의 컨테이너 크래쉬되어 삭제되거나 재시작 되더라도 emptyDir의 생명주기는 컨테이너 단위가 아니라, Pod 단위이기 때문에, emptyDir은 삭제 되지 않고 계속해서 사용이 가능하다. 

생성 당시에는 디스크에 아무 내용이 없기 때문에, emptyDir  이라고 한다.

다음은 하나의 Pod에 nginx와 redis 컨테이너를 기동 시키고, emptyDir 볼륨을 생성하여 이를 공유하는 설정이다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volumes 
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - name: shared-storage
      mountPath: /data/shared
  - name: nginx
    image: nginx
    volumeMounts:
    - name: shared-storage
      mountPath: /data/shared
  volumes:
  - name : shared-storage
    emptyDir: {}
```



## hostpath

 hostPath는 노드의 로컬 디스크의 경로를 Pod에서 마운트해서 사용한다. 같은 hostPath에 있는 볼륨은 여러 Pod 사이에서 공유되어 사용된다. 

또한  Pod가 삭제 되더라도 hostPath에 있는 파일들은 삭제되지 않고 다른 Pod가 같은 hostPath를 마운트하게 되면, 남아 있는 파일을 액세스할 수 있다. 

![img](https://t1.daumcdn.net/cfile/tistory/99BAD7375B1D3F100B)

주의할점 중의 하나는 Pod가 재시작되서 다른 노드에서 기동될 경우, 그 노드의 hostPath를 사용하기 때문에, 이전에 다른 노드에서 사용한 hostPath의 파일 내용은 액세스가 불가능하다. 

hostPath는 노드의 파일 시스템을 접근하는데 유용한데, 예를 들어 노드의 로그 파일을 읽어서 수집하는 로그 에이전트를 Pod로 배포하였을 경우, 이 Pod에서 노드의 파일 시스템을 접근해야 한다. 이러한 경우에 유용하게 사용할 수 있다. 

아래는 노드의 /tmp 디렉토리를 hostPath를 이용하여 /data/shared 디렉토리에 마운트 하여 사용하는 예제이다.



```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - name: terrypath
      mountPath: /data/shared
  volumes:
  - name : terrypath
    hostPath:
      path: /tmp
      type: Directory
```



## pv

### pv & pvc

### GCE

### amazone s3






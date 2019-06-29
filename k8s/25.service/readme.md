



# service





## service

## expose



**expose 명령을 이용한 서비스 생성 (외부IP 노출)**

아래 명령으로 LoadBalancer 타입의 서비스를 생성하여 nginx 컨테이너를 외부에서 접근할 수 있게 한다. 생성된 External-IP 와 port 는 서비스가 유지되는 동안만 유효하다. (External-IP 생성에 일정 시간이 소요될 수 있음)

```sh
$ kubectl expose deployment nginx --port 80 --type LoadBalancer
service/nginx exposed

$ kubectl get services
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
kubernetes   ClusterIP      10.51.240.1     <none>          443/TCP        18m
nginx        LoadBalancer   10.51.250.242   34.66.113.107   80:32693/TCP   1m
```



LoadBalancer type 은 클러스터에서 lo









## nodeport

## ingress




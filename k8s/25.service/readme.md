



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

참고 : https://arisu1000.tistory.com/27840 

인그레스(ingress)는 클러스터 외부에서 내부로 접근하는 요청들을 어떻게 처리할지 정의해둔 규칙들의 모음입니다. 외부에서 접근가능한 URL을 사용할 수 있게 하고, 트래픽 로드밸런싱도 해주고, SSL 인증서 처리도 해주고, 도메인 기반으로 가상 호스팅을 제공하기도 합니다. 인그레스 자체는 이런 규칙들을 정의해둔 자원이고 이런 규칙들을 실제로 동작하게 해주는게 인그레스 컨트롤러(ingress controller)입니다. 

```yaml
# 14.userlist-ingress.yaml
apiVersion: extensions/v1beta1
#apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: userlist-ingress
spec:
  rules:
  - host: userlist.gcp.com
    http:
      paths:
      #- path: /users/1
      #  backend:
      #    serviceName: userlist2-svc
      #    servicePort: 80
      - path: /
        backend:
          serviceName: userlist-svc
          servicePort: 80

```



```
$ ks get ingress
NAME               HOSTS              ADDRESS          PORTS   AGE
userlist-ingress   userlist.gcp.com   104.196.35.210   80      12m
```





## ingress controller

참고 :  <https://kubernetes.github.io/ingress-nginx/deploy/#gce-gke>

인그레스는 설정일 뿐이고 이 설정내용대로 동작하는 실제 행위자가 인그레스 컨트롤러임. 즉, 인그레스는 인그레스 컨트롤러에 정책을 적용시키기 위해 존재한다.  또한 인그레스 컨트롤러도 로드발란스 형태의 제공되는 일반적인 서비스와 pod 로 구성된다.

 인그레스 컨트롤러는 여러가지가 있지만 쿠버네티스가 공식적으로 제공하고 있는건 gce용인 ingress-gce와 nginx용인 ingress-nginx 2가지가 있다. ingress-gce는 gce를 이용하면 자동으로 사용할 수 있다. (하지만 직접 설치해서도 사용 가능하다.) 

 ingress-nginx는 github https://github.com/kubernetes/ingress-nginx 에서 확인가능함.

다음 명령으로 ingress-nginx 실행

```sh
# ingress-controller deployment
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml

# service
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud-generic.yaml
```



 특이한점은 인그레스컨트롤러서비스와 인그레스의 외부 아이피(로드발란서)가 동일하다.

```sh
# ingress 확인
$ kubectl -n dev-song get ingress
NAME               HOSTS              ADDRESS          PORTS   AGE
userlist-ingress   userlist.gcp.com   104.196.35.210   80      12m

# ingress controller svc 확인
$ kubectl -n ingress-nginx get svc
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx   LoadBalancer   10.11.240.22   104.196.35.210   80:32685/TCP,443:30592/TCP   33m


```

104.196.35.210 로 동일한것을 확인 할 수 있다.



ingress controller log 확인

```
$ kubectl -n ingress-nginx get pod
NAME                                        READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-76c86d76c4-29cv2   1/1     Running   0          38m

$ kubectl -n ingress-nginx logs -f nginx-ingress-controller-76c86d76c4-29cv2
```






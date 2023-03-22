
# headless service를 활용한 POD들의 IP 확인


# 1. 개요
모든 POD에 Message 를 보내는 방법에 대해서 분석한다.
일반적인 Service는 Load Balancer 목적으로 사용한다. 즉, 한 번에 하나의 POD로만 전달되며 트래픽이 반복되면 RoundRobbin 방식으로 트래픽이 순환된다.
그러므로 일반적인 Service 를 통해 모든 POD 에 신호를 보내는 것은 적절치 않다.
어떤 방식을 사용해야 하는지 방안을 고민해 본다.


# 2 방안 분석

## 방안1 - Kafka 활용
Pod 마다 다른 Consumer Group 으로 동일 메시지를 Consum 하는 방안이다.

## 방안2 - Redis Pub/Sub

## 방안2 - K8S API 활용
k8s API 를 활용하여 POD 들의 endpoint를 알아내어 직접 메세지를 전송한다.
> Cluster 내에서만 통신 가능


참고 
```sh
$ kubectl get endpoints service-name -o yaml
```


## 방안3 - headless service 활용
headless service 를 이용하여 POD들의 IP 를 취득한후 메세지 전송한다.
> Cluster 내에서만 통신 가능





# 3. Headless Service 활용한 IP 획득 분석
일반 서비스는 서비스 고유의 IP를 가지고 있으며 Load Balancer(Proxy) 역할을 수행한다.
하지만 Headless Servcie 는 IP 가 없으며 직접 POD 로 연결된다. 그러므로 Load Balancer 역할을 수행하지 못한다.

## 1) headless service
아래와 같이 ClusterIP 가 None 기록하는 Service 가 Headless service 이다.
```yaml
kind: Service
apiVersion: v1
metadata:
  name: userlist-svc-hl
  namespace: song
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8181
  selector:
    app: userlist
  clusterIP: None
  clusterIPs:
    - None
  type: ClusterIP
  sessionAffinity: None
```


## 2) nslookup 를 활용하여 IP 획득 사례
* 일반 서비스 IP

```sh
$  nslookup userlist-svc.song.svc.cluster.local
Server:         172.30.0.10
Address:        172.30.0.10:53

Name:   userlist-svc.song.svc.cluster.local
Address: 172.30.99.19
```
일반 서비스는 Service 의 IP가 리턴된다.


* Headless Service IP
```sh
$ nslookup userlist-svc-hl.song.svc.cluster.local
Server:         172.30.0.10
Address:        172.30.0.10:53


Name:   userlist-svc-hl.song.svc.cluster.local
Address: 10.129.6.202
Name:   userlist-svc-hl.song.svc.cluster.local
Address: 10.129.6.207
```


## 3) java Sample

* 도메인주소를 이용하여 IP 알아내기(java InetAddress 활용)

https://ery221.tistory.com/4

https://needneo.tistory.com/205




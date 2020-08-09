# istio install - v1.1.17



# 1. download 및 CRD 설치



## 1.1 download

적당한 위치에 아래와 같이 Download 받는다.

참고 : https://github.com/istio/istio/releases/tag/1.1.17

```shell
wget https://github.com/istio/istio/releases/tag/1.1.17

tar -xzvf istio-1.1.17-linux.tar.gz
cd istio-1.1.17

```



## 1.2 CRD 먼저 설치

```shell
cd istio-1.1.17

for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done
```



## 1.3 NS 생성

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: istio-system
  labels:
    istio-injection: disabled
EOF
```





# 2.  설치

사이드카 주입/미주입등 옵션이 있지만 사이드카 주입없이 istio 설치를 권장한다.

## 2.1 사이드카 주입없이 istio 설치

```shell
# A lighter template, with just pilot/gateway.
# Based on install/kubernetes/helm/istio/values-istio-minimal.yaml
helm template --namespace=istio-system \
  --set prometheus.enabled=false \
  --set mixer.enabled=false \
  --set mixer.policy.enabled=false \
  --set mixer.telemetry.enabled=false \
  `# Pilot doesn't need a sidecar.` \
  --set pilot.sidecar=false \
  --set pilot.resources.requests.memory=128Mi \
  `# Disable galley (and things requiring galley).` \
  --set galley.enabled=false \
  --set global.useMCP=false \
  `# Disable security / policy.` \
  --set security.enabled=false \
  --set global.disablePolicyChecks=true \
  `# Disable sidecar injection.` \
  --set sidecarInjectorWebhook.enabled=false \
  --set global.proxy.autoInject=disabled \
  --set global.omitSidecarInjectorConfigMap=true \
  `# Set gateway pods to 1 to sidestep eventual consistency / readiness problems.` \
  --set gateways.istio-ingressgateway.autoscaleMin=1 \
  --set gateways.istio-ingressgateway.autoscaleMax=1 \
  `# Set pilot trace sampling to 100%` \
  --set pilot.traceSampling=100 \
  install/kubernetes/helm/istio \
  > ./istio-lean.yaml

kubectl apply -f istio-lean.yaml
```





# 3. cluster local gateway

클러스터 내부에서만 통신되도록(backend-service) 설정하려면 클러스터 로컬 경로를 활성화 해야 한다.

아래와 설치하게 되면 istio-system 내에 pod 와 svc 가 설치된다.



```shell
# Add the extra gateway.
helm template --namespace=istio-system \
  --set gateways.custom-gateway.autoscaleMin=1 \
  --set gateways.custom-gateway.autoscaleMax=1 \
  --set gateways.custom-gateway.cpu.targetAverageUtilization=60 \
  --set gateways.custom-gateway.labels.app='cluster-local-gateway' \
  --set gateways.custom-gateway.labels.istio='cluster-local-gateway' \
  --set gateways.custom-gateway.type='ClusterIP' \
  --set gateways.istio-ingressgateway.enabled=false \
  --set gateways.istio-egressgateway.enabled=false \
  --set gateways.istio-ilbgateway.enabled=false \
  install/kubernetes/helm/istio \
  -f install/kubernetes/helm/istio/example-values/values-istio-gateways.yaml \
  | sed -e "s/custom-gateway/cluster-local-gateway/g" -e "s/customgateway/clusterlocalgateway/g" \
  > ./istio-local-gateway.yaml

kubectl apply -f istio-local-gateway.yaml
```



# 4. istio 설치확인

## 4.1 service 확인

```
$ kubectl get svc -nistio-system
NAME                    TYPE           CLUSTER-IP   EXTERNAL-IP    PORT(S)                                      AGE
cluster-local-gateway   ClusterIP      10.0.2.216   <none>         15020/TCP,80/TCP,443/TCP                     2m14s
istio-ingressgateway    LoadBalancer   10.0.2.24    34.83.80.117   15020:32206/TCP,80:30742/TCP,443:30996/TCP   2m14s
istio-pilot             ClusterIP      10.0.3.27    <none>         15010/TCP,15011/TCP,8080/TCP,15014/TCP       2m14s
```

- istio-ingressgateway 가 LoadBalancer Type 으로 설치됨.
- EXTERNAL-IP( 34.83.80.117 ) 을 통해서 외부에서 접근가능할 수 있다.



## 4.2 DNS 

DNS가 필요하다면 아래와 같이 xip.io 를 이용하면 편하다.

```
# 설정
34.83.80.117.xip.io

# ping test
ping aaa.34.83.80.117.xip.io
34.83.80.117
```

참고 : http://xip.io/
























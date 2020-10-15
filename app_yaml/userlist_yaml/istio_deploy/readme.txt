#### istio-system ####
alias oi='oc -n istio-system'

#### dev-song ####
alias os='oc -n song'

oc label namespace song istio-injection=enabled


os create -f 11.userlist-deployment.yaml
os create -f 12.userlist-svc.yaml
os create -f 13.userlist-gateway-vs.yaml
oi create -f 14.userlist-route.yaml

os delete -f 11.userlist-deployment.yaml
os delete -f 12.userlist-service.yaml
os delete -f 13.userlist-gateway-vs.yaml
oi delete -f 14.userlist-route.yaml

os apply -f 11.userlist-deployment.yaml
os apply -f 12.userlist-service.yaml
os apply -f 13.userlist-gateway-vs.yaml
oi apply -f 14.userlist-route.yaml

# service test
curl userlist-svc.song.svc.cluster.local/users/1
while true; do curl userlist-svc.song.svc.cluster.local/users/1; echo ""; sleep 1; done

# routing test
curl http://userlist-song.ktdscoe.myds.me/users/1
while true; do curl userlist.ipc.kt.com/users/1; echo ""; sleep 1; done

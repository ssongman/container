

## 1) external service

externalName 에는 CName(Canonical Name) 만 입력 가능하다.

IP address 는 불가하다.



```sh


$ cd ~/song/externalName
  
  
$ cat > 11.externalname1.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ExternalName
  externalName: google.com
---


$ kubectl -n yjsong apply -f 11.externalname1.yaml


# 삭제시...
$ kubectl -n yjsong delete -f 11.externalname1.yaml

```



### net-tools 

```sh


$ kubectl -n yjsong create deploy net-tools --image=ssongman/net-tools -- sleep 365d


$ kubectl -n yjsong exec -it  deploy/net-tools -- bash

# 삭제시...
$ kubectl -n yjsong delete deploy/net-tools

```



#### nslookup

```sh


$ nslookup google.com
Server:         10.43.0.10
Address:        10.43.0.10#53

Non-authoritative answer:
Name:   google.com
Address: 142.250.76.142
Name:   google.com
Address: 2404:6800:400a:804::200e



$ nslookup my-service

Server:         10.43.0.10
Address:        10.43.0.10#53

my-service.yjsong.svc.cluster.local     canonical name = google.com.
Name:   google.com
Address: 142.250.76.142
Name:   google.com
Address: 2404:6800:400a:804::200e


```



#### curl 테스트

```sh

$ curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>


$ curl my-service
<!DOCTYPE html>
<html lang=en>
  <meta charset=utf-8>
  <meta name=viewport content="initial-scale=1, minimum-scale=1, width=device-width">
  <title>Error 404 (Not Found)!!1</title>
  <style>
    *{margin:0;padding:0}html,code{font:15px/22px arial,sans-serif}html{background:#fff;color:#222;padding:15px}body{margin:7% auto 0;max-width:390px;min-height:180px;padding:30px 0 15px}* > body{background:url(//www.google.com/images/errors/robot.png) 100% 5px no-repeat;padding-right:205px}p{margin:11px 0 22px;overflow:hidden}ins{color:#777;text-decoration:none}a img{border:0}@media screen and (max-width:772px){body{background:none;margin-top:0;max-width:none;padding-right:0}}#logo{background:url(//www.google.com/images/branding/googlelogo/1x/googlelogo_color_150x54dp.png) no-repeat;margin-left:-5px}@media only screen and (min-resolution:192dpi){#logo{background:url(//www.google.com/images/branding/googlelogo/2x/googlelogo_color_150x54dp.png) no-repeat 0% 0%/100% 100%;-moz-border-image:url(//www.google.com/images/branding/googlelogo/2x/googlelogo_color_150x54dp.png) 0}}@media only screen and (-webkit-min-device-pixel-ratio:2){#logo{background:url(//www.google.com/images/branding/googlelogo/2x/googlelogo_color_150x54dp.png) no-repeat;-webkit-background-size:100% 100%}}#logo{display:inline-block;height:54px;width:150px}
  </style>
  <a href=//www.google.com/><span id=logo aria-label=Google></span></a>
  <p><b>404.</b> <ins>That’s an error.</ins>
  <p>The requested URL <code>/</code> was not found on this server.  <ins>That’s all we know.</ins>


```


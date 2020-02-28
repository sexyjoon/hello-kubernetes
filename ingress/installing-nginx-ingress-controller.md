# Nginx Ingress Controller 설치하기
### 들어가며
AWS EKS 환경에서 [nginx-ingress-controller](https://github.com/kubernetes/ingress-nginx) 를 설치할것이다.
### 설치하기
인그레스 컨트롤러 및 그에 필요한 리소스를 설치한다.
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
```
nginx 의 설정을 커스터마이징 한다. nginx-ingress-controller 의 Dockerfile 을 직접 수정하기보다는 설정 파일만 ConfigMap 형태로 제공하면 간단함.


```
$ cat configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
data:
  log-format-upstream: '{"time": "$time_iso8601", "remote_addr": "$proxy_protocol_addr", "request_id": "$req_id","status":$status, "path": "$uri", "request_query": "$args", "request_body": "$request_body", "duration": $request_time,"method": "$request_method", "http_user_agent": "$http_user_agent" , "service_name": "$service_name"}'
```
```
$ kubectl apply -f my-configmap.yaml
```

인그레스 컨트롤러와 클라이언트를 연결해 줄 로드밸런서를 배포한다. (시간이 조금 걸리니 기다릴 것)
```
$ kubectl apply -f https://raw.githubusercontent.com/cornellanthony/nlb-nginxIngress-eks/master/nlb-service.yaml
```


로드밸런서의 상태가 active 가 되었다면 간단한 에코서버를 Pod + Service 형태로 배포한다.
```
$ kubectl apply -f https://raw.githubusercontent.com/cornellanthony/nlb-nginxIngress-eks/master/apple.yaml
```

배포한 서비스와 인그레스 컨트롤러를 연결해 줄 인그레스를 배포한다.
```
$ cat ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
    - http:
        paths:
          - path: /apple
            backend:
              serviceName: apple-service
              servicePort: 5678
```
```
$ kubectl apply -f ingress.yaml
```

curl 을 사용해 로드밸런서에 요청을 보내본다.
```
$ curl XXXXXXX-XXXXXXXXXX.my-region.elb.amazonaws.com/apple
apple
```

ConfigMap 을 통해 커스터마이징 한 설정파일이 제대로 동작하는지 access log 를 확인해본다.
기본적으로 nginx-ingress-controller 의 access_log 는 stdout 에 심볼릭 링크로 연결되고, 노드의 /var/log/containers/nginx-ingress-controller-xxxxx.log 파일로 저장된다. (이후에 이 파일을 fluentd 의 in-tail 플러그인과 연결하여 인그레스로 통하는 모든 서비스의 요청을 로깅할 것이다.)
```
$ kubectl logs nginx-ingress-controller-xxxx -n ingress-nginx
```
```
{"time": "2020-02-28T07:45:51+00:00", "remote_addr": "-", "request_id": "e2417148a2f7b8871fae7b42abced5d2","status":400, "path": "/", "request_query": "-", "request_body": "-", "duration": 0.000,"method": "GET", "http_user_agent": "-" , "service_name": ""}
```

## 참고
* [NGINX Custom Configuration](https://kubernetes.github.io/ingress-nginx/examples/customization/custom-configuration/)
* [Using a Network Load Balancer with the NGINX Ingress Controller on Amazon EKS](https://aws.amazon.com/blogs/opensource/network-load-balancer-nginx-ingress-controller-eks/)

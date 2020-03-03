# Logging Architecture
컨테이너의 충돌, 파드의 축출, 노드의 제거 등 여러 이유로 인해 노드 안에 저장된 컨테이너의 로그들이 유실될 수 있다. 클러스터-레벨 로깅을 통해 이를 해결할 수 있고 클러스터-레벨 로깅은 원격 저장소, 분석, 쿼리 등을 요구한다. 쿠버네티스는 네이티브한 클러스터-레벨 로깅 솔루션을 구축하지 않았기 때문에 직접 구현해야한다.

### Kubernetes Logging 기초
1초에 한 번씩 stdout 으로 메시지를 출력하는 파드를 배포해보자.
```
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    args: [/bin/sh, -c,
            'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done']
```
```
$ kubectl apply -f https://k8s.io/examples/debug/counter-pod.yaml
```
kubectl logs 명령어를 사용해 해당 파드의 로그를 확인해보자.
```
$ kubectl logs counter
0: Mon Jan  1 00:00:00 UTC 2001
1: Mon Jan  1 00:00:01 UTC 2001
2: Mon Jan  1 00:00:02 UTC 2001
...
```
kubectl logs 명령어를 사용하면 kublet 에 요청을 하고 kublet 이 컨테이너의 로그를 읽어서 그 결과를 반환한다. 이때 로그파일은 /var/logs/containers/ 디렉토리에 저장이 되는데 이는 docker engine 의 [logging driver](https://docs.docker.com/config/containers/logging/configure/) 가 컨테이너 내의 stdout 과 stderr 를 해당 디렉토리에 로그파일로 기록한다. logging driver 는 /etc/docker/daemon.json 파일에서 설정값을 확인할 수 있고 logging driver 의 종류, 로테이션 주기(로그파일이 무한정 커지는것을 방지) 등을 설정할 수 있다.

### 노드-레벨 로깅

### 클러스터-레벨 로깅


## 참고
* [Docker Logging Driver](https://docs.docker.com/config/containers/logging/configure/)
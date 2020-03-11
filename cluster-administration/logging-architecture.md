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
모든 컨테이너화 된 애플리케이션들이 stdout 과 stderr 에 쓰는 로그들은 컨테이너 엔진에 의해 처리되고 어딘가로 전달된다. 예를들어 Docker 엔진같은 경우 두 스트림이 logging driver 로 전달되고, 쿠버네티스에서는 기본적으로 json 포맷의 파일로 저장되도록 설정되어있다.
컨테이너가 재시작되면 kublet 은 종료된 컨테이너의 로그를 저장하고 있지만, 파드가 축출되면 그 안의 모든 컨테이너와 해당 로그가 축출된다. 로그가 머신의 사용가능한 용량을 다 써버리지 않기 위해 logrotate 도구를 사용해 주기적으로 로그파일을 지우고 새로운 파일로 작성한다. 단 kubectl log 명령어를 사용할때는 가장 최신의 로그 파일만을 읽기 때문에 만약 한계용량에 도달한 로그파일과 0kb 의 두가지 로그파일이 있다면 kubectl log 의 결과는 빈 값일 것이다.

#### System component logs
* Kubernetes Scheduler, kube-proxy (컨테이너 안에서 동작)
* kublet, container runtime (컨테이너 안에서 동작 X)

System component 에는 두가지 타입이 있는데 하나는 컨테이너 안에서 동작하는 것이고 다른 하나는 그렇지 않은 것이다. 전자의 경우 일반 컨테이너들과 마찬가지로 /var/log/ 디렉토리에 로그가 저장된다. 후자의 경우 머신에 systemd 의 여부에 따라 있을 경우 journald 로, 없을 경우 /var/log/ 디텍토리에 .log 형태로 로그가 저장된다.

### 클러스터-레벨 로깅

클러스터-레벨 로깅에는 세가지 종류가 있다.

* 모든 노드에서 실행되는 노드-레벨 에이전트 사용
* 애플리케이션 파드에 로깅 전용 사이드카 컨테이너를 사용 + 모든 노드에서 실행되는 노드-레벨 에이전트 사용
* 애플리케이션 파드에 로깅 전용 사이드카 컨테이너를 사용 + 사이드카 컨테이너에서 원격 저장소로 직접 로깅

#### 모든 노드에서 실행되는 노드-레벨 에이전트 사용
![logging-with-node-agent](https://d33wubrfki0l68.cloudfront.net/2585cf9757d316b9030cf36d6a4e6b8ea7eedf5a/1509f/images/docs/user-guide/logging/logging-with-node-agent.png)

노드-레벨 에이전트를 각 노드에 배치함으로써 클러스터-레벨 로깅을 구현할 수 있다. 이 로깅 에이전트는 로그를 추출해 원격 저장소로 보내는 역할을 한다. 따라서 로깅 에이전트는 모든 애플리케이션 파드로부터 생성된 로그가 저장되는 디렉토리에 접근한다. 보통 데몬셋, 파드, 노드의 전용 프로세스 형태로 사용할 수 있지만 뒤의 두 개는 추천하지 않는다. 보통은 데몬셋 형태의 fluentd 를 배포하여 사용한다.

참고 : [fluentd-kubernetes-daemonset 을 사용하여 로깅 시스템 구축하기]()

#### 애플리케이션 파드에 로깅 전용 사이드카 컨테이너를 사용 + 모든 노드에서 실행되는 노드-레벨 에이전트 사용
![logging-with-streaming-sidecar](https://d33wubrfki0l68.cloudfront.net/5bde4953b3b232c97a744496aa92e3bbfadda9ce/39767/images/docs/user-guide/logging/logging-with-streaming-sidecar.png)

#### 애플리케이션 파드에 로깅 전용 사이드카 컨테이너를 사용 + 사이드카 컨테이너에서 원격 저장소로 직접 로깅
![logging-with-sidecar-agent](https://d33wubrfki0l68.cloudfront.net/d55c404912a21223392e7d1a5a1741bda283f3df/c0397/images/docs/user-guide/logging/logging-with-sidecar-agent.png)

## 참고
* [Docker Logging Driver](https://docs.docker.com/config/containers/logging/configure/)
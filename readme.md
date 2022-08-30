# 쿠버네티스 명령어

파일 실행
> kubectl apply -f [uri]

pods 가져오기
> kubectl get pods

pods 삭제
> kubectl delete pod [name]

deployment pods 삭제
> kubectl delete deployment [deploy name]

배포된 pods 상세확인
> kubectl get pods -o wide -w

kubelet 종료
> systemctl stop kubelet

kube scale
> kubectl scale deployment [deploy name] --replicas=[i]

# 쿠버네티스는 어떻게 구성되어 있을까?

- api
- etcd
- c-m (컨트롤 매니저)
- sched
- CoreDNS
- k-proxy
- cni
- kubelet

## 구역을 나누는 네임스페이스(Namespace)

> kubectl get pods -n kube-system

명령어를 실행하면 구성하는 구역의 목록을 확인하실 수 있습니다.

### 다른 클라우드 에서도 동일하게 구성요소가 세팅되어 있을까?

- AWS EKS (Elastic Kubernetes Service)
  - aws-node
  - coredns
  - kube-proxy
- Azure AKS
  - coredns
  - kube-proxy
  - metrics-server
  - omsagent
  - tunnelfront
- GCP GKE (Google Kubernetes Engine)
  - event-exporter-gke
  - fluentbit-gke
  - gke-metrics
  - kube-dns
  - kube-proxy
  - metrics-server
  - pdcsi-node

3사의 관리형 쿠버네티스 구성 요소가 차이가 많이 있고 이런 구성 요소들을 보여주지 않습니다.

# 쿠버네티스의 기본 철학 (MSA)

마이크로서비스 아키텍처 = 하는 일들이 개별적으로 나누어있음  
이의 반대되는 것은 모놀리식 아키텍처 = 하나에 모두가 참여되어 있음

## 파드가 배포되면?

- 사용자 kubectl create pod(s) 파드 생성 요청
  - API 서버 & etcd 로 넘어갑니다.
- API 서버가 모든것의 중심에 있습니다.
  - pod(s) 생성 요청을 받으면 c-m (컨트롤러 매니저) 로 파드의 생성되었는지 감시하도록 합니다.
  - 그리하여 API 서버가 모든 상태값 모든 중심에 있습니다.
  - API 는 감시만하고 선언한 값만 가지고 있습니다
- 스케줄러가 각 pod 들을 배포를 하도록 스케줄 하는것 까지만 담당
  - API는 새로운 pod 가 워커 노드가 들어갔는지 감시
  - 스케줄러는 새로운 파드가 워크 노드에 넣도록 스케줄 함
- API 는 Kubelet 에게 새로운 파드가 노드에 잘 소속되어 있는지 감시
- Kubelet -> docker 파드의 동작관리
  - docker 컨테이너 런타임이 컨테이너 생성

API 서버는 중앙에서 상태값을 가지고 있으며 그 상태를 c-m, sched, Kubelet 등이 계속 맞추려고 합니다.  
이렇게 API 서버가 선언한다는 방식을 `선언적인 시스템` 이라고 합니다.

추구하는 상태와 현재 상태를 동일하게 맞추려고함  
이를 [ 상태변경 -> 감시 -> 차이발견 ] 을 무한 반복

API 서버와 etcd 서버의 방식은 조금 다릅니다.  
API 서버는 가지고 있는 현재상태 추구하는 값을 가지고 있는데 중간에 사라질 수 있으니  
매번 클러스터의 업데이트된 정보를 etcd 공간에 저장을 합니다.

# 실제 쿠버네티스의 파드 배포 흐름

- 마스터 노드 (m-k8s) - 관리자, 개발자
  - 워크 노드#1 - 사용자
  - 워크 노드#2 - 사용자
  - 워크 노드#3 - 사용자

마스터 노드에서 개발자가 kubectl 명령어로 API 서버에 요청합니다.  
API 서버는 etcd 에 해당 요청 정보를 업데이트를 진행합니다.
문제가 생기면 복구하는 방법은 etcd 에 있습니다.

다음 c-m 으로 통신합니다.  
API 서버의 값을 c-m 가 보고 자신의 값을 바꾸고 API 서버에 업데이트를 진행합니다.

스케줄러는 워크로드에 파드를 로드밸런싱 하게 도와줍니다.  
파드가 각각의 워커 노드에 밸러드 되게 들어가도록 도와주는 역할을 합니다.

API 서버를 보고 Kubelet 이 각각의 워커 노드에게 컨테이너 런타임에게 파드를 생성하도록 요청합니다.

각각의 워커 노드에서 컨테이너 런타임이 파드를 생성합니다.

이렇게 생성된 파드들은 사용자와 통신하기 위해서 kube-proxy 를 통해서 통신하게 됩니다.

이러한 네트워크는 쿠버네티스에서 기본 제공을 하지 않고  
사용자에서 직접 선택하게끔 하고있습니다. 이를 컨테이너 네트워크 인터페이스라고 합니다.  
예를들어 calico 를 사용했습니다.

# 쿠버네티스 파드에 문제가 생겼다면

- 파드를 실수로 지웠다면?
  - 파드만 배포된 경우
    - 난감 복구할 수 없음
  - 디플로이먼트 형태로 배포된 파드
    - 디플로이먼트가 파드를 유지하기 때문에 문제가 생기지 않음

쿠버네티스의 파드는 언제든지 삭제되고 다시 생성될 수 있다.

> kubectl delete pod [deployment pod name]

deployment pod 를 삭제해도 바로 추가 생성됩니다.  
이는 deployment 가 특정 갯수를 유지해야 한다는 설정이 있기 때문에 상태가 보존되는것

## 쿠버네티스 워커 노드의 구성 요소에 문제가 생겼다면

특정 노드에서 kubelet 종료 후 마스터 노드에서 pods 생성 후 상태확인을 하면  
예를들어 3개중 2개는 정상적으로 동장했지만 종료된 노드에서만 pending 상태로 나옵니다.  
다시 종료된 kubelet 을 실행하고 마스터 노드에서 상태 확인을 하면 정상적으로 3개가 실행되는 것을 확인할 수 있습니다. 

이번엔 컨테이너 런타임을 중지 시키면 어떻게 될까?  
서브 노드(w1-k8s)에서 systemctl stop docker 명령어를 실행  
다음 마스터 노드에서 scale 명령어로 6개로 배포해본다  
다음 pods 상태를 확인해보면 [NODE] 항목에 보면 w1-k8s 에만 배포하지 않는 것을 확인할 수 있습니다.

w1-k8s 1개  
w2-k8s 2개  
w3-k8s 3개  

이미 균형이 깨진 배포를 scale 값을 9개로 늘려 다시 배포하면 `스케줄러`의 역할로  
균등하게 배포되게 끔해줄 것입니다.

w1-k8s 3개    
w2-k8s 3개  
w3-k8s 3개  

## 쿠버네티스 마스터 노드에 문제가 생긴다면?

만약 마스터 노드의 스케줄러가 삭제가 된다면?  

마스터 노드에서 kube-system 을 확인해보자  
> kubectl get pods -n kube-system -o wide

kube-scheduler-m-k8s 를 삭제하면 어떻게 될까?   
pods 이기때문에 보호해줄 장치가 없는데 어떻게 될까  

> kubectl delete pod kube-scheduler-m-k8s -n kube-system 

네임 스페이스를 주어 삭제합니다.  
그리고 다시 조회하면 새로운 pods 로 생성된것을 확인할 수 있습니다.

쿠버네티스의 마스터 노드에 존재하는 중요한 것들은 특별하게 관리됩니다.  
pods 라 할지라도 자동으로 다시 만들어 집니다.

### 만약 마스터 노드에있는 kubelet 이 중단된다면?

마스터 노드의 systemctl stop kubelet  
그 후 삭제 명령어

> kubectl delete pod kube-scheduler-m-k8s -n kube-system 

를 실행하면 실행되지 않습니다. 이 상태를 확인하면  

> kubectl get pods -n kube-system -o wide

kube-scheduler-m-k8s 의 상태는 `Terminationg` 삭제 진행중으로 계속 출력되게 됩니다.  
이는 kubelet 이 중단되어 배포가 되고 있지 않아서 그렇습니다.  

그러면 이런 상태에서 워크노드의 어플리케이션은 제대로 배포할 수 있을까?  

> kubectl create deployment nginx --image=nginx
> kubectl get pod

결과를 보면 생성되는 것을 확인할 수 있습니다.  
그렇다면 scheduler 가 문제가 있으니 스케일링은 안되지 않을까?  

> kubectl scale deployment nginx --replicas=3
> kubectl get pod

정상적으로 스케일링이 됩니다. 이는 정상적으로 스케줄링이 된다는 뜻입니다.

이는 kube-scheduler-m-k8s 의 상태 `Terminationg` 삭제 진행중 이지만 사실 kubelet 을 통해서  
전달되지 않기 때문에 문제가 생기지 않는것입니다.

### 만약 마스터 노드의 컨테이너가 중단된다면?

마스터 노드  
> systemctl stop docker
> kubectl get pods

API 서버에 연결이 되지않는다는 오류가 출력됩니다.  
컨테이너 런타임이 없으니 통신을 못하기 때문에 그렇습니다.  

이러면 배포한 어플리케이션 들은 어떻게 될까?  
이렇게 되면 통신을 하고있던 것들까지 모두 문제가 생기게 됩니다.  
`따라서 마스터 노드에 있는 컨테이너 런타임은 가장 중요한 요소입니다.`  
API 서버를 핸들링 하는 서버이기 때문입니다.

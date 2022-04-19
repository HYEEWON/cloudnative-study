# K8S in GKE (Google Kubernetes Engine)

<br>

## 목차
* [Kubernetes 클러스터 생성](#kubernetes-클러스터-생성)
* [CLOUD SHELL에서 Kubenetes 사용하기](#cloud-shell에서-kubenetes-사용하기)
  * [Replicaset](#replicaset)
  * [Deployment](#deployment)

<br>

## Kubernetes 클러스터 생성

<img src="https://user-images.githubusercontent.com/38900338/163932468-e3c942bb-0532-43ec-bfc6-9c558af49924.png" width="450px">

### GKE Standard
* Master에 Worker Node 배치
* 노드를 직접 관리하고 구성, 확장을 직접 구성
* 노드 단위로 결제: 사용하지 않아도(내부에 컨테이너 없음) 결제

### GKE Autopilot
* `Serverless`
* Google에서 노드를 관리하고 구성, 워크로드를 기준으로 자동 확장
* Pod 단위로 결제: 사용해야 결제

<img src="https://user-images.githubusercontent.com/38900338/163909298-d97ece27-2820-4e2d-bb45-826a36d6f4f2.png" width="600px">

<br>

## CLOUD SHELL에서 Kubenetes 사용하기

### ✅ 클러스터 연결

### ✅ 사용 중인 노드 목록 확인

```bash
kubectl get nodes
```

<img src="https://user-images.githubusercontent.com/38900338/163996902-e677ff84-ccb4-4f06-a6ec-1c543d0e1482.png" width="600px">

### ✅ Deployment 생성
* 톰켓 이미지를 사용해 Pod의 수가 5개인 Deployment 생성

```bash
kubectl create deploy tc --image=consol/tomcat-7.0 --replicas=5
```

#### Replicaset
* 실행되는 파드 개수에 대한 가용성 보장, 항상 지정한 파드 수만큼 실행되도로 관리
* 예) replicas=5 : 1개가 삭제될 경우 다시 파드 1개를 실행해 5개를 유지

#### Deployment
* Replicaset의 상위 개념
* 배포할 때 사용하는 컨트롤러로 배포에 대한 기능이 세분화되어 있음 (Pod 수 유지, 일시 정지 등)

### ✅ Deployment를 외부로 노출시키는 서비스 오브젝트 생성

```bash
kubectl expose deploy tc --type=LoadBalancer --port=80 --target-port=8080
```

### ✅ Pod와 Service 조회

```bash
kubectl get pod,svc
```

<img src="https://user-images.githubusercontent.com/38900338/164000242-2ef0f0ec-86e5-4dd8-b436-dea161fa3d34.png" width="600px">

* Pod는 Replicaset의 개수인 5개가 조회됨 (아래부터 5개)
* NodePort 서비스: 30000-32767 포트 사용

### ✅ 모든 Deployment, Service 삭제

```bash
kubectl delete all --all
```

### ✅ 기타 명령어 참고
* [링크](https://judo0179.tistory.com/66)


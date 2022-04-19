# K8S in GCE

<br>

## 목차
* [Compute Engine에서 환경 구성](#compute-engine에서-환경-구성)
* [사전 준비 사항](#사전-준비-사항)
* [Kubernetes 런타임 관련 정보](#kubernetes-런타임-관련-정보)
* [containerd 런타임 설치](#containerd-런타임-설치)
* [kubeadm, kubelet 및 kubectl 설치](#kubeadm-kubelet-및-kubectl-설치)
* [Cgroup 드라이버 구성](#cgroup-드라이버-구성)
* [노드 설정](#노드-설정)
* [Deployment 생성 및 배포](#deployment-생성-및-배포)
* [결과 확인](#결과-확인)
* [기타 명령어](#기타-명령어)

<br>

## Compute Engine에서 환경 구성
* GCP의 `Compute Engine`에서 인스턴스 3개 생성

```
    마스터: master-1
    노드: worker-1, worker-2
```

* OS를 Ubuntu로 설정
* 방화벽 해제는 하지 않아도 됨
  * 서비스는 3XXXX 포트를 사용하기 때문에 불필요

<img src="https://user-images.githubusercontent.com/38900338/164015411-36a989aa-b8d1-4c83-8ecb-b8c517af9960.png" width="450px">

<br>

## 사전 준비 사항
* 컴퓨터 당 2GB 이상의 RAM, CPU 2개 이상
* 모든 클러스터의 공용 또는 사설 네트워크 연결
* 노드가 특정 포트를 사용할 수 있어야 함 ([참고](https://kubernetes.io/ko/docs/reference/ports-and-protocols))
  * 참고 사이트에 있는 포트를 사용할 수 있어야 함
* 모든 노드에 대한 고유한 호스트 이름, 고유한 MAC 주소, 고유한 product_uuid
  * MAC 주소 확인 방법 : `ifconfig -a` 또는 `ip link`
  * product_uuid 확인 방법 : `sudo cat /sys/class/dmi/id/product_uuid
* Swap을 사용하지 않아야 함 ([참고](https://askubuntu.com/questions/21`4805/how-do-i-disable-swap))
  * Swap: 메모리가 부족할 경우 하드 디스크 공간을 활용해 작업을 하도록 도와주는 영역
  * kubernetes에서 가장 기본이 되는 Pod의 컨셉은 필요한 리소스 만큼 호스트 자원에서 할당 받아 사용하는 것
  * 때문에, Pod를 할당하고 제어하는 kubelet은 스왑 상황을 처리하도록 설계되지 않음

```bash
sudo swapoff -a # 현재 시스템의 스왑을 비활성화 
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab # 리부팅 후부터 스왑을 비활성화
```

* [참고](https://medium.com/finda-tech/overview-8d169b2a54ff)

<br>

## Kubernetes 런타임 관련 정보
* Kubernetes 1.24에서는 `dockershim`이 삭제됨
* 도커는 OCI에서 정한 표준이 아니기 때문에 문제가 많아 삭제
* 아래 표에서 도커를 제외한 `containerd`, `CRI-O`가 남게 됨

|런타임|유닉스 도메인 소켓 경로|
|:--:|:---|
|도커|/var/run/dockershim.sock|
|containerd|/run/containerd/containerd.sock|
|CRI-O|/var/run/crio/crio.sock|

* 도커를 설치하면 containerd도 포함되어 설치됨
* 도커와 containerd 중에 도커가 우선적으로 선택되며, 도커를 설치하면 containerd 런타임을 사용할 수 있음

<br>

## containerd 런타임 설치

```bash
sudo -i # 관리자 권한으로 전환
apt update && apt install docker.io -y # 도커 설치
```

<br>

## kubeadm, kubelet 및 kubectl 설치
* kubeadm: 클러스터를 부트스트랩(설치)하는 명령
* kubelet: 클러스터의 모든 머신에서 실행되는 Pod와 컨테이너 시작과 같은 작업을 수행하는 컴포넌트, 쿠버네티스 런타임을 조정
* kubectl: 클러스터와 통신하기 위한 커맨드 라인 유틸리티, 쿠버네티스를 제어함

```yaml
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# gpg key 다운로드
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

# 패키지 관리 도구에 도커 다운로드 링크 추가
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
```

<br>

## Cgroup 드라이버 구성
* `컨테이너 런타임`과 `kubelet`의 `cgroup 드라이버`가 일치해야 쿠버네티스 클러스터가 정상 동작 가능
* 아래와 같은 상황에서 `systemd`가 권장됨 -> 컨테이너 런타임을 `systemd`로 변경

```
    - 컨테이너 런타임: cgroupfs
    - kubelet: systemd
```

### ✅ 현재 사용 중인 Cgroup 이름 확인

```bash
docker info | grep -i cgroup # 대소문자 무시
```

### ✅ Cgroup 이름 변경
* [참고](https://stackoverflow.com/questions/43794169/docker-change-cgroup-driver-to-systemd)

```bash
cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
```

```bash
sudo systemctl restart docker
```

<br>

## 노드 설정

### 🔵 마스터 노드: Init

```bash
kubeadm init
```

### 🔵 마스터 노드: 클러스터 유저 설정
* 마스터 노드 Init 결과 메시지에서 확인 가능

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 🟡 워커노드: Join
* 마스터 노드 Init 결과 메시지에서 확인 가능
* 연결하고자 하는 모든 워커노드에서 실행

```bash
kubeadm join ip:6443 --token text \
        --discovery-token-ca-cert-hash sha256:token

# 6443: 쿠버네티스 API 서버 포트
```


### 🔵 마스터 노드: Pod 네트워크 설정
* 마스터 노드 Init 결과 메시지에서 확인 가능
* 노드 상태 변화: NotReady -> Ready

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

<br>

## Deployment 생성 및 배포
* 마스터에서 실행

```bash
kubectl create deploy tc --image=consol/tomcat-7.0 --replicas=5
kubectl expose deploy tc --type=NodePort --port=80 --target-port=8080
```

<br>

## 결과 확인
* 마스터에서 실행

### ✅ 노드 확인

```bash
kubectl get nodes
```

<img src="https://user-images.githubusercontent.com/38900338/164012472-7b0d7cb3-08ef-4074-9341-8a4a9904bdf4.png" width="700px">

* NodePort 서비스: 30000-32767 포트 사용

### ✅ 컨테이너 배포 위치 확인

```bash
kubectl get pod -o wide
```

* 어떤 워커노드에서 실행되는지 알 수 있음

<img src="https://user-images.githubusercontent.com/38900338/164012483-221831aa-fae6-4f11-aa3f-67afd7712e11.png" widh="700px">

<br>

## 기타 명령어
### ✅ kubeadm 버전 확인

```bash
kubeadm version
```

### ✅ kubeadm 설정 리셋

```bash
kubeadm reset
```

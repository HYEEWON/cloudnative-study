# Kubernates

## 역사
* 구글이 사용하던 내부 시스템을 2014년 오픈소스로 공개

## 정의
*  도커를 통해 구동되는 `컨테이너의 배포/확장/관리`를 자동화해주는 오픈소스 플랫폼
* MSA 형태로 설계되어 있음
* GO 언어로 구현되어 있음, 퍼블릭 클라우드에서 사용 가능, 프라이빗 클라우드나 베어메탈(가상화X)에서도 배포 가능

## 쿠버네티스 아키텍처

#### 클러스터
* 효율적인 리소스 공유와 균형 배분을 위해 `여러 노드를 묶은 그룹`

#### 클러스터링
* 여러 서버를 묶어 하나의 시스템처럼 동작하게 하며, 클라이언트에게 고가용성 서비스를 제공
* Active-Active 구조: 서버들이 항상 Active 상태
* Active-Standby 구조: Standby 서버는 Active 서버에 장애가 발생하면 바로 Active 상태로 바뀔 수 있음
  * Active -> Standby: Fail Over / Standby -> Active: Fail Back

#### 마스터 - 노드 구조
* 클러스터를 관리하는 마스터와 컨테이너가 배포되는 노드로 구성
* `Master`에 `API 서버`와 `상태 저장소(etcd)`를 두고 각 서버(`Node`)의 에이전트(`kubelet`)와 통신하는 구조
* 모든 명령은 마스터의 `API 서버`를 호출하고 노드는 마스터와 통신하면서 필요한 작업을 수행

##### Master
* 클러스터의 설정 환경 저장 및 클러스터 관리
* Api Server, Controller, Scheduler, etcd로 구성
* Api Server: 관리자의 요청이 kubectl을 통해 클러스터에 요청됨 -> Api Server가 받아 etcd에 상태 저장
* Controller: `Desired State`를 유지하기 위해 변경 사항이 있는지 지속적으로 확인
* Scheduler: 할당되지 않은 Pod을 적절한 노드에 할당해주는 모듈
* etcd: 클러스터의 모슨 상태 정보 저장, Key-Value 저장소

##### Node
* 쿠버네티스에서 최소 단위의 컴퓨팅 하드웨어
* 실제 컨테이너가 생성되는 장소
* 워크로드(Pod) 실행
* Kubelet(큐블릿): Pod의 생명주기 관리, Pod 생성, Pod의 상태를 주기적으로 API Server에 전달
* Kube-proxy: Pod로 연결되는 네트워크 관리

#### Pod
* 배포 가능한 가장 작은 컴퓨팅 단위
* 파드 사이에는 NAT Gateway가 없음, SNAT를 통해 주소 변환 발생
* 하나의 파드에 프로세스가 2개인 경우(중간 그림)
  * 로그 수집: file system을 공유하는 보조 컨테이너를 두어 log 수집 담당
  * Proxy: 프로세스와 Proxy는 1개 Pod에 넣음
  * 보통 3~5개가 들어가는 경우가 많음


## Liveness, Readiness, Startup
* 3개를 다 설정했다면 Startup이 가장 먼저 시작

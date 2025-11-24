---
title: Core Kubernetes
date: 2023-11-28
pin: false
tags:
- DevOps
- Kubernetes
---

## 쿠버네티스 감 잡기

### The Way

- DevOps를 전체 Application Life Cycle에 점차적으로 통합되도록 나아감
  - 핵심 사항
    Infra Reproducibility; 인프라의 재현성
  - 장애 수정을 위한 변경 사항이 모든 컴포넌트에 완벽하게 복제되어야 함

### Infra Drift Problem

- 패치가 누락되거나 부적절하게 적용되어 일부 시스템이 다른 것과 다르게 동작하는 문제
- 예시
  - 여러 서버에서 자바 버전 업데이트
  - 특정 애플리케이션이 특정 위치에서 실행되지 않도록 설정
  - 로드 밸런싱 경로의 수동 관리
- 쿠버네티스는 모든 애플리케이션의 상태를 중앙에서 관리하도록 도와줌
  - `kubectl`

### Containerization

- 애플리케이션은 실행되는 호스트와의 높은 종속성을 지님
  - 자바 애플리케이션을 위해 방화벽 규칙이 설정된 JVM을 필요로 하는 등
- 컨테이너는 실행 중인 OCI[^1] 이미지
  - 서로 다른 버전의 라이브러리를 필요로 하는 두 애플리케이션이 있는 경우 두 개의 컨테이너를 사용하는 등

- 컨테이너에 대한 자동화가 필요하며, 쿠버네티스가 자동화에 적합

[^1]: OCI; Open Container Initiative, 실행 가능한 독립 애플리케이션을 위한 이미지 형식, 도커 이미지라고도 불림

### 쿠버네티스 개요

![](images/kuberenetes-cluster-example.png "쿠버네티스 클러스터의 예")

- 쿠버네티스는 같은 클러스터 내에서 실행되는 애플리케이션의 수명주기 관리를 표준화
  - Container Orchestration

![](images/kubernetes-control-plane-and-node.png "쿠버네티스의 구성")

- HW Infrastructure
- Kubernetes Worker Node
- Kubernetes Control Plane

## 파드 감 잡기

![](images/kubernetes-brief-architecture.png "쿠버네티스 간략 구조")

### About Kubernetes Objects

- Pod; 파드
  - 쿠버네티스 클러스터에 배치할 수 있는 가장 작은 원자적 단위
  - 쿠버네티스 클러스터 노드에서 컨테이너로 실행되는 하나 이상의 OCI 이미지
  - 대부분 파드는 직접 배포되지 않고 상위 수준의 추상화가 사용됨
    1. Deployment
    2. Job
    3. StatefulSet
    4. DaemonSet
- Node; 노드
  - `kubelet`을 실행하는 단일 컴퓨팅 파워
  - Control Loop를 통해 쿠버네티스 API 서버와 통신

### Control Plane

1. 쿠버네티스 API 서버 `kube-apiserver`
   - `kubectl apply` 실행 시 통신하는 첫번째 쿠버네티스 컴포넌트
   - 노드가 시작되면, API 서버와 통신을 통해 클러스터에 잘 등록됐는지 확인
   - 쿠버네티스 데이터베이스인 etcd와 통신
2. 쿠버네티스 스케줄러 `kube-scheduler`
   - 노드에 새로운 파드를 할당
   - 파드의 생명주기를 제어
3. 쿠버네티스 컨트롤러 관리자; KCM `kube-controller-manager`
   - PersistentVolume과 PersistentVolumeClaim 객체를 활성화

### Scaling

- High Availability를 위한 기본적인 메커니즘

- 자주 발생하는 실패나 작업

  1. Pod Outage Problem
  2. Node Outage Problem
  3. Software Update Outage Problem

- Noisy Neighbor 현상 고려

  > Noisy Neighbor Problem
  >
  > : 한 서버에서 혼잡할 정도로 많은 프로세스를 실행하면 부족한 리소스에 대한 과도한 경쟁을 이어지는 문제



## 용어

- CNI

  : Container Networking Interface

- CSI

  : Container Storage Interface
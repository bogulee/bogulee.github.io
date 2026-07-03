+++
title = 'KubeReleaseLab : 02. Vagrant와 kubeadm으로 3-Node 클러스터 구성'
date = 2026-07-03T09:00:00+09:00
draft = true
description = 'Vagrant와 kubeadm으로 Control Plane 1대와 Worker 2대로 구성된 Kubernetes 클러스터를 만들고, 네트워크와 DNS 동작을 검증한 과정을 정리합니다.'
categories = ['KubeReleaseLab']
tags = ['Project', 'Kubernetes', 'Vagrant', 'VirtualBox', 'kubeadm', 'Calico']
+++

## 이번 단계의 목표

앞선 글에서는 Kubernetes의 기본 구조와 클러스터 구축 과정에 등장하는 도구의 역할을 정리했다. 이번에는 그 개념을 실제 환경으로 연결한다.

이번 단계의 목표는 다음과 같다.

- Vagrant로 Rocky Linux VM 3대 생성
- 모든 Node에 containerd와 Kubernetes 패키지 설치
- kubeadm으로 Control Plane 초기화
- Calico CNI 설치
- Worker 2대를 클러스터에 가입
- Node, 시스템 Pod와 클러스터 DNS 상태 검증

최종 구성은 다음과 같다.

| 역할 | Hostname | IP | CPU | Memory |
| --- | --- | --- | --- | --- |
| Control Plane | `k8s-control-plane` | `192.168.56.30` | 2 | 4 GB |
| Worker | `k8s-worker-1` | `192.168.56.31` | 2 | 3 GB |
| Worker | `k8s-worker-2` | `192.168.56.32` | 2 | 3 GB |

## Vagrantfile의 역할

이번 구성에서 Vagrant는 Kubernetes 자체를 설치하는 도구가 아니다. VirtualBox VM을 만들고 각 VM에서 필요한 Shell 명령을 실행하는 역할을 담당한다.

Vagrantfile은 크게 세 부분으로 나눴다.

```text
NETWORK_SETUP
→ hostname과 고정 IP 설정

COMMON_SETUP
→ 모든 Node에 공통 환경과 Kubernetes 패키지 설치

CONTROL_PLANE_SETUP
→ kubeadm init과 Calico 설치
```

Worker의 `kubeadm join`은 자동화하지 않았다. Control Plane에서 생성한 명령을 직접 확인하고 각 Worker에서 실행하는 과정을 학습하기 위해서다.

전체 Vagrantfile은 [KubeReleaseLab 저장소](https://github.com/bogulee/KubeReleaseLab/blob/main/vagrant/Vagrantfile)에서 확인할 수 있다.

## VM 네트워크 구성

VirtualBox는 각 VM에 두 개의 네트워크 인터페이스를 제공한다.

```text
eth0: NAT 네트워크
→ 패키지와 컨테이너 이미지 다운로드

eth1: Host-only 네트워크
→ Control Plane과 Worker 간 통신
```

Control Plane과 Worker가 같은 주소로 계속 통신할 수 있도록 `eth1`에는 VM별 고정 IP를 사용했다.

```ruby
control_plane.vm.network "private_network",
  ip: "192.168.56.30",
  auto_config: false
```

Vagrantfile의 네트워크 설정과 VM 내부의 실제 IP가 일치하는지는 `ip address`로 확인했다.

## kubeadm 사전 준비

모든 Node에서 다음 공통 설정을 수행했다.

### Swap 비활성화

```bash
swapoff -a
```

재부팅 후에도 Swap이 활성화되지 않도록 `/etc/fstab`도 함께 수정했다.

### 커널 모듈과 네트워크 설정

```text
overlay
br_netfilter
```

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

`br_netfilter`는 Linux Bridge를 통과하는 패킷이 iptables 규칙의 영향을 받을 수 있도록 하고, IP Forwarding은 Node가 패킷을 다른 인터페이스로 전달할 수 있게 한다.

### containerd 설치

containerd를 설치한 뒤 Kubernetes와 cgroup 관리 방식을 맞추기 위해 systemd cgroup을 사용하도록 설정했다.

```toml
SystemdCgroup = true
```

### Kubernetes 패키지 설치

각 Node에 다음 패키지를 설치했다.

- kubelet
- kubeadm
- kubectl

kubeadm은 클러스터 초기화와 Node 가입에 사용하고, kubelet은 각 Node에서 Pod 상태를 관리한다. kubectl은 API Server에 요청을 보내는 명령줄 클라이언트다.

## Control Plane 초기화

Control Plane에서는 다음 설정으로 `kubeadm init`을 실행했다.

```bash
kubeadm init \
  --apiserver-advertise-address=192.168.56.30 \
  --pod-network-cidr=10.244.0.0/16
```

`--apiserver-advertise-address`는 다른 Node가 API Server에 접근할 주소다. `--pod-network-cidr`는 클러스터의 Pod에 할당할 네트워크 대역이다.

초기화 후 `vagrant` 사용자가 kubectl을 사용할 수 있도록 관리자 kubeconfig를 복사했다.

```bash
mkdir -p /home/vagrant/.kube
cp /etc/kubernetes/admin.conf /home/vagrant/.kube/config
chown -R vagrant:vagrant /home/vagrant/.kube
```

## Calico CNI 설치

kubeadm은 Pod 네트워크 구현을 자동으로 선택하지 않는다. 이번 환경에서는 Calico를 설치했다.

Calico의 IP Pool은 `kubeadm init`에 지정한 Pod CIDR과 동일한 `10.244.0.0/16`으로 맞췄다.

```text
kubeadm Pod CIDR: 10.244.0.0/16
Calico IP Pool:   10.244.0.0/16
```

## Worker Node 가입

Control Plane에서 Worker 가입 명령을 생성했다.

```bash
sudo kubeadm token create --print-join-command
```

출력된 명령을 각 Worker에서 실행했다.

```bash
sudo kubeadm join 192.168.56.30:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

토큰과 인증서 해시는 일시적인 인증 정보이므로 글이나 저장소에 실제 값을 남기지 않는다.

## 구성 중 발생한 문제

### Guest Additions 자동 업데이트 실패

호스트 VirtualBox와 VM의 Guest Additions 버전이 달라 `vagrant-vbguest`가 자동 업데이트를 시도했다. 그러나 실행 중인 Rocky Linux 커널과 일치하는 `kernel-devel` 패키지를 찾지 못해 VM 구성이 중단됐다.

이 프로젝트에서는 VirtualBox 공유 폴더가 필수가 아니므로 Guest Additions 자동 업데이트를 비활성화했다.

```ruby
if Vagrant.has_plugin?("vagrant-vbguest")
  config.vbguest.auto_update = false
end
```

### VirtualBox 공유 폴더 마운트 실패

Worker 가입 명령을 전달하기 위해 `/vagrant-join` 공유 폴더를 사용하려 했지만, Guest Additions 문제로 `vboxsf` 파일시스템을 사용할 수 없었다.

```text
mounting failed with the error: No such device
```

공유 폴더를 복구하기 위해 구성 요소를 더 추가하기보다 의존성을 제거했다. Worker 가입 명령은 각 VM에서 직접 실행하도록 단순화했다.

## 클러스터 상태 검증

### Node 상태

```bash
kubectl get nodes -o wide
```

검증 결과 Control Plane 1대와 Worker 2대가 모두 `Ready` 상태가 됐다.

### 시스템 Pod 상태

```bash
kubectl get pods -A -o wide
```

다음 구성 요소가 모두 `Running`이고 재시작 횟수가 0인 것을 확인했다.

- etcd
- kube-apiserver
- kube-controller-manager
- kube-scheduler
- kube-proxy
- CoreDNS
- calico-node
- calico-kube-controllers

### Pod 네트워크와 DNS 검증

Worker Node에 테스트 Pod를 실행했다.

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600
```

Pod는 `k8s-worker-1`에 배치됐고 `10.244.230.2`를 할당받았다.

```bash
kubectl exec dns-test -- \
  nslookup kubernetes.default.svc.cluster.local
```

```text
Server:  10.96.0.10
Name:    kubernetes.default.svc.cluster.local
Address: 10.96.0.1
```

Pod의 `/etc/resolv.conf`도 확인했다.

```text
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

이를 통해 다음 흐름이 정상적으로 동작하는 것을 확인했다.

```text
Worker에서 Pod 실행
→ Calico가 Pod IP 할당
→ kube-dns Service에 요청
→ CoreDNS가 Kubernetes Service 이름 해석
```

검증 후 테스트 Pod는 삭제했다.

```bash
kubectl delete pod dns-test
```

## 정리

이번 단계에서는 Kubernetes 패키지를 설치하는 것뿐 아니라 Control Plane과 Worker의 연결, CNI와 DNS까지 검증했다.

또한 실습에 꼭 필요하지 않은 공유 폴더 자동화를 제거하고 Worker 가입 과정을 직접 수행하면서 `kubeadm init`과 `kubeadm join`의 역할을 구분할 수 있었다.

다음 글에서는 별도 Namespace를 만들고 Pod, Deployment와 Service를 순서대로 구성하면서 Kubernetes 기본 리소스의 연결 관계를 확인한다.

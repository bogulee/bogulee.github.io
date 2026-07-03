+++
title = 'KubeReleaseLab : 02. Vagrant와 kubeadm으로 3-Node 클러스터 구성'
date = 2026-07-03T09:00:00+09:00
draft = false
description = 'Vagrant와 kubeadm으로 Control Plane 1대와 Worker 2대로 구성된 Kubernetes 클러스터를 만들고, 네트워크와 DNS 동작을 검증한 과정을 정리합니다.'
categories = ['KubeReleaseLab']
tags = ['Project', 'Kubernetes', 'Vagrant', 'VirtualBox', 'kubeadm', 'Calico']
+++

## 이번 단계의 목표

앞선 글에서는 Kubernetes의 기본 구조와 클러스터 구축 과정에 등장하는 도구의 역할을 정리했고 이번에는 그 개념을 실제 환경으로 연결합니다.

이번 단계의 목표는 다음과 같습니다.

- Vagrant로 Rocky Linux VM 3대 생성
- 모든 Node에 containerd와 Kubernetes 패키지 설치
- kubeadm으로 Control Plane 초기화
- Calico CNI 설치
- Worker 2대를 클러스터에 가입
- Node, 시스템 Pod와 클러스터 DNS 상태 검증

최종 구성은 다음과 같습니다.

| 역할 | Hostname | IP | CPU | Memory |
| --- | --- | --- | --- | --- |
| Control Plane | `k8s-control-plane` | `192.168.56.30` | 2 | 4 GB |
| Worker | `k8s-worker-1` | `192.168.56.31` | 2 | 3 GB |
| Worker | `k8s-worker-2` | `192.168.56.32` | 2 | 3 GB |

## Vagrantfile의 역할

이번 구성에서 Vagrant는 Kubernetes 자체를 설치하는 도구가 아닌 VirtualBox VM을 만들고 각 VM에서 필요한 Shell 명령을 실행하는 역할을 담당합니다.

재구성한 Vagrantfile은 크게 세 부분으로 나뉩니다.

```text
NETWORK_SETUP
→ hostname과 고정 IP 설정

COMMON_SETUP
→ 모든 Node에 공통 환경과 Kubernetes 패키지 설치

CONTROL_PLANE_SETUP
→ kubeadm init과 Calico 설치
```

전체 Vagrantfile은 [KubeReleaseLab 저장소](https://github.com/bogulee/KubeReleaseLab/blob/main/vagrant/Vagrantfile)에서 확인할 수 있습니다.

## VM 네트워크 구성

VirtualBox는 각 VM에 두 개의 네트워크 인터페이스를 제공합니다.

```text
eth0: NAT 네트워크
→ 패키지와 컨테이너 이미지 다운로드

eth1: Host-only 네트워크
→ Control Plane과 Worker 간 통신
```

Control Plane과 Worker가 같은 주소로 계속 통신할 수 있도록 `eth1`에는 VM별 고정 IP를 사용했습니다.

```ruby
control_plane.vm.network "private_network",
  ip: "192.168.56.30",
  auto_config: false
```

Vagrantfile의 네트워크 설정과 VM 내부의 실제 IP가 일치하는지는 `ip address`로 확인했습니다.

## kubeadm 사전 준비

모든 Node에서 다음 공통 설정을 수행합니다.

### Swap 비활성화

```bash
swapoff -a
```

재부팅 후에도 Swap이 활성화되지 않도록 `/etc/fstab`도 함께 수정합니다.

<details>
<summary><strong>참고: Kubernetes에서 Swap을 비활성화하는 이유</strong></summary>

Swap은 물리 메모리(RAM)가 부족할 때 디스크의 일부를 임시 메모리처럼 사용하는 기능입니다.

Kubernetes는 Node의 실제 메모리와 Pod의 `requests`, `limits`를 기준으로 스케줄링과 메모리 부족 상태를 판단합니다. 
이때 Swap이 자동으로 개입하면 메모리가 부족한 프로세스가 종료되지 않고 디스크로 이동할 수 있습니다.

이 경우 다음과 같은 문제가 발생할 수 있습니다.

- 메모리 부족 상태를 늦게 감지할 수 있다.
- Pod가 종료되지는 않지만 응답 속도가 크게 느려질 수 있다.
- kubelet의 Eviction과 메모리 사용량 판단이 복잡해진다.
- 설정한 memory limit과 실제 자원 상태를 예측하기 어려워질 수 있다.

이 때문에 kubelet은 기본적으로 Swap이 활성화된 Linux Node에서 시작하지 않습니다. 이번 실습에서는 별도의 Swap 정책을 구성하지 않고 kubelet의 기본 동작을 사용하기 위해 Swap을 비활성화했습니다.

재부팅 후에도 Swap이 다시 활성화되지 않도록 `/etc/fstab`의 Swap 설정도 비활성화합니다.

최신 Kubernetes에서는 Swap을 반드시 꺼야 하는 것은 아니지만 다음과 같이 kubelet이 Swap이 있는 Node에서 실행되도록 설정할 수 있습니다.

```yaml
failSwapOn: false
```

다만 이 설정만으로 Pod가 Swap을 사용하는 것은 아닙니다. 기본 정책은 `NoSwap`입니다.

```yaml
memorySwap:
  swapBehavior: NoSwap
```

Pod의 제한적인 Swap 사용까지 허용하려면 다음 정책을 명시해야 합니다.

```yaml
memorySwap:
  swapBehavior: LimitedSwap
```

이번 프로젝트에서는 Swap 메모리 관리가 핵심 학습 범위가 아니므로 Swap을 비활성화하고 Pod의 `requests`, `limits`, OOMKilled와 Eviction 동작에 집중합니다.

</details>

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

`br_netfilter`는 Linux Bridge를 통과하는 패킷이 iptables 규칙의 영향을 받을 수 있도록 하고 IP Forwarding은 Node가 패킷을 다른 인터페이스로 전달할 수 있게 합니다.

### containerd 설치

containerd를 설치한 뒤 Kubernetes와 cgroup 관리 방식을 맞추기 위해 systemd cgroup을 사용하도록 설정합니다.

```toml
SystemdCgroup = true
```

<details>
<summary><strong>참고 : containerd의 cgroup driver 설정</summary></strong>

cgroup은 Linux에서 프로세스가 사용할 CPU와 메모리를 제한하고 측정하는 기능입니다.
Kubernetes의 `requests`와 `limits`도 최종적으로는 Linux cgroup을 통해 적용됩니다.

Node에서는 kubelet과 containerd가 모두 cgroup을 다룹니다.

```text
kubelet
→ Pod의 자원 관리 구조 구성

containerd
→ 컨테이너 프로세스를 해당 cgroup에서 실행
```

두 구성 요소가 서로 다른 cgroup 관리 방식을 사용하면 Pod의 자원 상태를 일관되게 관리하기 어렵습니다.

kubeadm으로 구성한 kubelet은 기본적으로 systemd cgroup driver를 사용합니다. 따라서 containerd도 같은 방식을 사용하도록 설정했습니다.

SystemdCgroup = true

최종적으로 kubelet과 containerd가 모두 systemd를 통해 cgroup을 관리합니다.

kubelet:    systemd
containerd: systemd
</details>

### Kubernetes 패키지 설치

각 Node에 다음 패키지를 설치합니다.

- kubelet
- kubeadm
- kubectl

kubeadm은 클러스터 초기화와 Node 가입에 사용하고 kubelet은 각 Node에서 Pod 상태를 관리합니다. kubectl은 API Server에 요청을 보내는 명령줄 클라이언트입니다.

## Control Plane 초기화

Control Plane에서는 다음 설정으로 `kubeadm init`을 실행합니다.

```bash
kubeadm init \
  --apiserver-advertise-address=192.168.56.30 \
  --pod-network-cidr=10.244.0.0/16
```

`--apiserver-advertise-address`는 다른 Node가 API Server에 접근할 주소, `--pod-network-cidr`는 클러스터의 Pod에 할당할 네트워크 대역입니다.

초기화 후 `vagrant` 사용자가 kubectl을 사용할 수 있도록 관리자 kubeconfig를 복사합니다.

```bash
mkdir -p /home/vagrant/.kube
cp /etc/kubernetes/admin.conf /home/vagrant/.kube/config
chown -R vagrant:vagrant /home/vagrant/.kube
```

## Calico CNI 설치

kubeadm은 Pod 네트워크 구현을 자동으로 선택하지 않습니다. 이번 환경에서는 Calico를 설치합니다.

Calico의 IP Pool은 `kubeadm init`에 지정한 Pod CIDR과 동일한 `10.244.0.0/16`으로 맞춥니다.

```text
kubeadm Pod CIDR: 10.244.0.0/16
Calico IP Pool:   10.244.0.0/16
```

## Worker Node 가입

Control Plane에서 Worker 가입 명령을 생성합니다.

```bash
sudo kubeadm token create --print-join-command
```

출력된 명령을 각 Worker에서 실행합니다.

```bash
sudo kubeadm join 192.168.56.30:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

토큰과 인증서 해시는 일시적인 인증 정보이므로 글이나 저장소에 실제 값을 남기지 않습니다.

## 클러스터 상태 검증

### Node 상태

```bash
kubectl get nodes -o wide
```

검증 결과 Control Plane 1대와 Worker 2대가 모두 `Ready` 상태가 됐습니다.

```bash
NAME                STATUS   ROLES           AGE    VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                      KERNEL-VERSION                 CONTAINER-RUNTIME
k8s-control-plane   Ready    control-plane   12m    v1.34.9   10.0.2.15       <none>        Rocky Linux 9.6 (Blue Onyx)   5.14.0-570.17.1.el9_6.x86_64   containerd://2.2.5
k8s-worker-1        Ready    <none>          2m5s   v1.34.9   192.168.56.31   <none>        Rocky Linux 9.6 (Blue Onyx)   5.14.0-570.17.1.el9_6.x86_64   containerd://2.2.5
k8s-worker-2        Ready    <none>          105s   v1.34.9   192.168.56.32   <none>        Rocky Linux 9.6 (Blue Onyx)   5.14.0-570.17.1.el9_6.x86_64   containerd://2.2.5
```

### 시스템 Pod 상태

```bash
kubectl get pods -A -o wide
```

다음 구성 요소가 모두 `Running`이고 재시작 횟수가 0인 것을 확인했습니다.

```bash
NAMESPACE     NAME                                        READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-6469cdbdc4-lm7hw    1/1     Running   0          13m
kube-system   calico-node-7q5g4                           1/1     Running   0          13m
kube-system   calico-node-f6fxs                           1/1     Running   0          3m40s
kube-system   calico-node-xwztq                           1/1     Running   0          3m59s
kube-system   coredns-66bc5c9577-kcpgn                    1/1     Running   0          13m
kube-system   coredns-66bc5c9577-nsgq4                    1/1     Running   0          13m
kube-system   etcd-k8s-control-plane                      1/1     Running   0          13m
kube-system   kube-apiserver-k8s-control-plane            1/1     Running   0          13m
kube-system   kube-controller-manager-k8s-control-plane   1/1     Running   0          13m
kube-system   kube-proxy-hd62g                            1/1     Running   0          3m59s
kube-system   kube-proxy-hglpm                            1/1     Running   0          3m40s
kube-system   kube-proxy-p6rnc                            1/1     Running   0          13m
kube-system   kube-scheduler-k8s-control-plane            1/1     Running   0          13m
```

### Pod 네트워크와 DNS 검증

Worker Node에 테스트 Pod를 실행했습니다.

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600
```

Pod는 `k8s-worker-1`에 배치됐고 `10.244.230.2`를 할당받았다.

```bash
NAME       READY   STATUS    RESTARTS   AGE   IP             NODE           NOMINATED NODE   READINESS GATES
dns-test   1/1     Running   0          14s   10.244.230.2   k8s-worker-1   <none>           <none>
```

```bash
kubectl exec dns-test -- \
  nslookup kubernetes.default.svc.cluster.local
```

```text
Server:  10.96.0.10
Name:    kubernetes.default.svc.cluster.local
Address: 10.96.0.1
```

Pod의 `/etc/resolv.conf`도 확인했습니다.

```text
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

이를 통해 다음 흐름이 정상적으로 동작하는 것을 확인했습니다.

```text
Worker에서 Pod 실행
→ Calico가 Pod IP 할당
→ kube-dns Service에 요청
→ CoreDNS가 Kubernetes Service 이름 해석
```

검증 후 테스트 Pod는 삭제합니다.

```bash
kubectl delete pod dns-test
```

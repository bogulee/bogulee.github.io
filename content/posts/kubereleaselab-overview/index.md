+++
title = 'KubeReleaseLab : 00. 프로젝트 개요'
date = 2026-07-03T06:00:00+09:00
draft = false
description = 'Kubernetes와 DevOps의 동작 원리를 직접 구성하고 검증하기 위한 KubeReleaseLab 프로젝트의 배경과 계획을 정리합니다.'
categories = ['KubeReleaseLab']
tags = ['Project', 'Kubernetes', 'DevOps']
+++

안녕하세요!

드디어 미루고 미루던 첫 글을 작성하게 되었습니다!
제가 블로그를 시작하게 된 가장 큰 이유는 추가 프로젝트를 진행하기 위함이었는데요,
과연 이것들이 완벽한 제 것이 될 수 있을지는 잘 모르겠지만 열심히 해보려고 합니다.. ㅠㅠ

## 프로젝트를 시작한 이유

이전에 Vagrantfile을 이용해 Kubernetes 클러스터를 구성해 본 적은 있습니다. 
명령을 실행하면 가상 머신과 클러스터가 만들어져서 굉장히 신기했던 기억이 나는군요.. 
하지만.. 당시에는 각 도구가 무슨 역할을 담당하고 어떤 순서로 연결되는지 제대로 이해하지 못했더랬죠.. ㅠㅠ

그치만 모든 프로젝트들을 잘 마무리(?) 지어서 그런가 딱히 엄청난 불편함이나 어려움은 느끼지 못했는데요..
이제 시간이 지나 취업을 위해 면접 준비를 하면서 여러 질문들에 답하기 어려워지자 아... x됐구나를 느끼게 되었답니다..

- Vagrant와 kubeadm은 각각 무엇을 구성하는가?
- Control Plane과 Worker Node는 어떤 과정을 통해 연결되는가?
- containerd와 kubelet은 어떤 관계인가?
- CNI를 설치하지 않으면 왜 Pod 간 통신이 되지 않는가?
- 배포 자동화와 모니터링은 Kubernetes 위에서 어떻게 연결되는가?

이번 프로젝트를 통해 완성된 설정을 그대로 실행하는 데 그치지 않고 작은 단계로 직접 구성하고 결과를 확인하면서 이 질문에 답해보려고 합니다!

## KubeReleaseLab이란

KubeReleaseLab은 Kubernetes 클러스터 구성부터 애플리케이션 배포, 장애 분석, GitOps와 모니터링 기반 배포 자동화까지 단계적으로 학습하는 미니 프로젝트입니다.

프로젝트의 최종 흐름은 다음과 같습니다.

```text
코드 Push
→ 테스트
→ 컨테이너 이미지 빌드
→ 이미지 보안 검사
→ 컨테이너 레지스트리에 이미지 저장
→ GitOps 매니페스트 변경
→ Kubernetes 배포
→ 카나리 버전의 메트릭 분석
→ 정상 버전 승격 또는 장애 버전 롤백
```

처음부터 이 모든 기능을 한꺼번에 구현하지는 않습니다. 각 단계의 구성 요소를 설명하고 검증할 수 있을 때 다음 단계로 넘어갈 계획입니다.

## 프로젝트 목표

### 1. Kubernetes 기본 구조 이해

Pod, Deployment, Service, ConfigMap, Secret과 같은 기본 리소스를 직접 구성합니다. 
단순히 YAML을 적용하는 것이 아니라 각 리소스가 어떤 기준으로 연결되는지 확인합니다.

### 2. 장애 재현과 원인 분석

정상 배포만 확인하지 않고 다음과 같은 장애를 의도적으로 만들어 볼 예정입니다.

- Service Selector 불일치
- ImagePullBackOff
- CrashLoopBackOff
- OOMKilled
- Readiness Probe 실패
- 리소스 부족으로 인한 Pending

장애가 발생하면 `kubectl get`, `describe`, `logs`, `events` 순서로 상태를 확인하고 원인과 해결 과정을 기록합니다.

### 3. CI와 GitOps 기반 배포 구성

GitHub Actions로 테스트, 이미지 빌드와 보안 검사를 수행합니다. 이후 Argo CD가 Git 저장소에 정의된 상태를 기준으로 Kubernetes 배포를 관리하도록 구성할 계획입니다.

### 4. 메트릭 기반의 안전한 배포

Prometheus에서 수집한 오류율과 응답 시간 지표를 Argo Rollouts의 배포 판단에 연결합니다. 정상 버전은 점진적으로 확대하고 문제가 있는 버전은 자동으로 중단하거나 롤백하는 것이 최종 목표입니다.

## 단계별 진행 계획

### 1단계: 클러스터와 기본 리소스

- 기존 Vagrantfile 분석
- kubeadm 기반 클러스터 구성 과정 이해
- FastAPI 애플리케이션 컨테이너화
- Deployment와 Service 구성
- Probe와 리소스 제한 적용

### 2단계: 네트워크와 트러블슈팅

- Service Discovery와 CNI 확인
- Kubernetes Event와 로그 확인
- 대표 장애 재현 및 해결
- 트러블슈팅 과정 문서화

### 3단계: CI와 GitOps

- GitHub Actions 구성
- 이미지 빌드와 Trivy 검사
- GitHub Container Registry 사용
- Argo CD 기반 배포와 Self-Heal 확인

### 4단계: 모니터링과 카나리 배포

- Prometheus와 Grafana 구성
- 애플리케이션 메트릭 노출
- Argo Rollouts 카나리 전략 구성
- 오류율 또는 P95 기준 자동 중단과 롤백

### 5단계: 운영 관점의 정리

- Non-root 컨테이너와 Secret 관리
- 배포 및 장애 대응 Runbook 작성
- 아키텍처와 배포 흐름 문서화
- 프로젝트의 한계와 개선 방향 정리

## 기록 방법

GitHub 저장소에는 다른 사람이 실행하고 검증할 수 있는 결과물을 남깁니다.

- 애플리케이션 코드
- Dockerfile과 Kubernetes 매니페스트
- CI/CD Workflow
- 실행 및 검증 방법
- 트러블슈팅 문서

블로그에는 구현 결과만 나열하기보다 학습 과정과 판단 근거를 기록합니다.

- 개념을 이해한 과정
- 도구를 선택한 이유
- 실패한 접근과 원인
- 실제로 확인한 결과
- 아직 해결하지 못한 부분

## 완료 기준

이 프로젝트는 여러 도구를 설치하는 것만으로 완료되지 않습니다. 다음 내용을 직접 설명하고 재현할 수 있는 상태를 목표로 합니다.

- Deployment와 Service가 Pod에 연결되는 과정
- Event와 로그를 이용한 Kubernetes 장애 분석
- CI와 GitOps CD의 역할 차이
- Prometheus 메트릭을 이용한 배포 판단
- 장애 버전의 카나리 배포 중단과 롤백
- README와 문서를 이용한 환경 재현

이 모든 것들을 다 마무리 할 수 있을지 모르겠지만 잘 정리해보고 싶네요!
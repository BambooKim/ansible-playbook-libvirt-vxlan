# ansible-playbook-libvirt-vxlan

3개의 Ubuntu 머신에 libvirt KVM 가상화 환경을 구성하고, VXLAN으로 오버레이 네트워크를 연결한 후 Kubernetes 클러스터를 배포하는 Ansible 플레이북

## 기능

- **인프라 구성**: libvirt + VXLAN 오버레이 네트워크
- **VM 프로비저닝**: cloud-init 기반 자동화
- **Kubernetes 배포**: 1 마스터 + 3 워커 클러스터 (v1.34.3)

## 네트워크 토폴로지

```
┌─────────────────────────────────────────────────────────────┐
│              VXLAN Overlay (VNI 100, 10.200.1.0/24)         │
└─────────────────────────────────────────────────────────────┘
        │                    │                    │
   ┌────┴────┐          ┌────┴────┐          ┌────┴────┐
   │br-vxlan │          │br-vxlan │          │br-vxlan │
   │10.200.1.1│         │10.200.1.2│         │10.200.1.3│
   ├─────────┤          ├─────────┤          ├─────────┤
   │vxlan100 │          │vxlan100 │          │vxlan100 │
   ├─────────┤          ├─────────┤          ├─────────┤
   │ eth0    │          │ eth0    │          │ eth0    │
   │.0.50    │          │.0.46    │          │.0.49    │
   └────┬────┘          └────┬────┘          └────┬────┘
        │                    │                    │
        └────────────────────┴────────────────────┘
                   Underlay (192.168.0.0/24)
```

## 호스트 정보

### 물리 호스트

| 호스트 | 물리 IP | 오버레이 IP | 역할 |
|--------|---------|-------------|------|
| kvm-host01 | 192.168.0.50 | 10.200.1.1 | KVM Host + NAT Gateway |
| kvm-host02 | 192.168.0.46 | 10.200.1.2 | KVM Host |
| kvm-host03 | 192.168.0.49 | 10.200.1.3 | KVM Host |

### Kubernetes VM

| VM | 호스트 | IP | vCPU | RAM | 디스크 | 역할 |
|----|--------|-----|------|-----|--------|------|
| k8s-master-01 | kvm-host01 | 10.200.1.10 | 4 | 4GB | 40GB | Master |
| k8s-worker-01 | kvm-host02 | 10.200.1.11 | 4 | 4GB | 50GB | Worker |
| k8s-worker-02 | kvm-host03 | 10.200.1.12 | 4 | 4GB | 50GB | Worker |
| k8s-worker-03 | kvm-host03 | 10.200.1.13 | 4 | 4GB | 50GB | Worker |

## 요구사항

- Ubuntu 22.04 LTS
- CPU 가상화 지원 (VT-x/AMD-V)
- SSH 접속 가능 (ubuntu 사용자)
- Ansible 2.15+

## 사전 준비

### 1. Ansible Galaxy 컬렉션 설치

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. 인벤토리 수정

`inventory/hosts.yml` 파일에서 각 호스트의 `underlay_interface`를 실제 인터페이스 이름으로 수정:

```yaml
kvm-host01:
  underlay_interface: eth0    # ip link 명령으로 확인
```

## 실행 방법

### 1단계: 인프라 구성 (VXLAN + libvirt)

```bash
ansible-playbook -i inventory/all_hosts.yml playbooks/site.yml
```

### 2단계: Kubernetes 클러스터 배포

```bash
# 전체 배포 (VM 생성 + K8s 설치)
ansible-playbook -i inventory/all_hosts.yml playbooks/k8s_deploy.yml

# 단계별 실행
ansible-playbook -i inventory/all_hosts.yml playbooks/k8s_deploy.yml --tags vm      # VM만 생성
ansible-playbook -i inventory/all_hosts.yml playbooks/k8s_deploy.yml --tags k8s     # K8s만 설치
ansible-playbook -i inventory/all_hosts.yml playbooks/k8s_deploy.yml --tags workers # 워커 join만
```

### 전체 배포 (레거시)

```bash
ansible-playbook playbooks/site.yml
```

### 태그별 실행

```bash
# 공통 설정만
ansible-playbook playbooks/site.yml --tags common

# VXLAN 네트워크만
ansible-playbook playbooks/site.yml --tags vxlan

# libvirt만
ansible-playbook playbooks/site.yml --tags libvirt
```

### 검증

```bash
ansible-playbook playbooks/verify.yml
```

### Dry-run

```bash
ansible-playbook playbooks/site.yml --check --diff
```

## 프로젝트 구조

```
.
├── ansible.cfg
├── requirements.yml
├── inventory/
│   ├── all_hosts.yml         # 통합 인벤토리 (물리 호스트 + VM)
│   ├── hosts.yml             # 물리 호스트 (레거시)
│   └── group_vars/
│       ├── all.yml
│       ├── kvm_hosts.yml     # VM 정의 및 cloud-init 설정
│       └── k8s_cluster.yml   # Kubernetes 설정
├── roles/
│   ├── common/               # 공통 패키지, 시스템 설정
│   ├── vxlan/                # VXLAN 오버레이 네트워크 + NAT
│   ├── libvirt/              # KVM/libvirt 설치 및 구성
│   ├── vm_provision/         # VM 생성 및 프로비저닝 (cloud-init)
│   └── k8s_cluster/          # Kubernetes 클러스터 설치
│       ├── tasks/
│       │   ├── prerequisites.yml    # 커널 모듈, sysctl 설정
│       │   ├── install_runtime.yml  # containerd 설치
│       │   ├── install_k8s.yml      # kubeadm, kubelet, kubectl
│       │   ├── init_master.yml      # 마스터 초기화
│       │   ├── setup_cni.yml        # Flannel CNI 설치
│       │   └── join_workers.yml     # 워커 join
│       └── templates/
│           └── containerd-config.toml.j2
├── playbooks/
│   ├── site.yml              # 인프라 배포
│   ├── k8s_deploy.yml        # Kubernetes 배포
│   ├── cleanup_vms.yml       # VM 정리
│   └── verify.yml            # 검증
└── README.md
```

## 검증 명령어

### 인프라 검증

배포 후 각 물리 호스트에서 확인:

```bash
# VXLAN 인터페이스 확인
ip -d link show vxlan100

# 브리지 확인
ip -d link show br-vxlan

# FDB 엔트리 확인
bridge fdb show dev vxlan100

# 오버레이 네트워크 연결 테스트
ping -c 3 10.200.1.1
ping -c 3 10.200.1.2
ping -c 3 10.200.1.3

# libvirt 네트워크 확인
virsh net-list --all

# VM 목록 확인
sudo virsh list --all
```

### Kubernetes 클러스터 검증

마스터 노드에 접속하여 확인:

```bash
# SSH 접속 (ProxyJump 설정 필요)
ssh k8s-master-01

# 또는
ssh -o ProxyCommand="ssh -i ~/.ssh/kvm-host02.pem -W %h:%p -q ubuntu@192.168.0.50" ubuntu@10.200.1.10

# 노드 상태 확인
kubectl get nodes -o wide

# 예상 출력:
# NAME            STATUS   ROLES           AGE   VERSION
# k8s-master-01   Ready    control-plane   1h    v1.34.3
# k8s-worker-01   Ready    <none>          1h    v1.34.3
# k8s-worker-02   Ready    <none>          1h    v1.34.3
# k8s-worker-03   Ready    <none>          1h    v1.34.3

# 시스템 Pod 확인
kubectl get pods -A

# CNI 상태 확인
kubectl get pods -n kube-flannel
kubectl get pods -n kube-system

# 테스트 Pod 생성
kubectl run nginx-test --image=nginx --restart=Never
kubectl wait --for=condition=ready pod/nginx-test --timeout=120s
kubectl get pod nginx-test -o wide
kubectl delete pod nginx-test
```

## VM 생성 예시

```bash
# Ubuntu Cloud 이미지 다운로드
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# VM 생성
virt-install \
  --name test-vm \
  --memory 1024 \
  --vcpus 1 \
  --disk path=/var/lib/libvirt/images/test-vm.qcow2,size=10 \
  --network network=vxlan-network \
  --os-variant ubuntu22.04 \
  --import \
  --graphics none \
  --noautoconsole
```

## 주요 설정값

### 네트워크

| 설정 | 값 |
|------|-----|
| VNI | 100 |
| VXLAN 포트 | 4789 |
| MTU | 1450 |
| 언더레이 네트워크 | 192.168.0.0/24 |
| 오버레이 네트워크 | 10.200.1.0/24 |
| libvirt 네트워크 | vxlan-network |
| 브리지 | br-vxlan |

### Kubernetes

| 설정 | 값 |
|------|-----|
| 버전 | 1.34.3 |
| Container Runtime | containerd 1.7 |
| CNI Plugin | Flannel (VXLAN 모드) |
| Pod CIDR | 10.244.0.0/16 |
| Service CIDR | 10.96.0.0/12 |
| API Server | 10.200.1.10:6443 |

## 문제 해결

### MTU 관련 이슈

**증상**: VM에서 GitHub/ghcr.io 연결 시 TLS handshake timeout

**원인**: VXLAN 오버헤드로 인한 MTU 불일치 (VM: 1500, VXLAN: 1450)

**해결**:
```bash
# VM의 MTU를 1450으로 설정 (cloud-init 템플릿에 이미 반영됨)
sudo ip link set enp1s0 mtu 1450

# 영구 적용 (netplan)
sudo vim /etc/netplan/50-cloud-init.yaml
# enp1s0 섹션에 "mtu: 1450" 추가

# containerd 재시작
sudo systemctl restart containerd
```

### GitHub 연결 문제

**증상**: Flannel manifest 다운로드 실패

**해결**: raw.githubusercontent.com 사용 (이미 setup_cni.yml에 반영됨)

```yaml
url: https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### VM 정리

```bash
# 모든 VM 삭제
ansible-playbook -i inventory/all_hosts.yml playbooks/cleanup_vms.yml

# 특정 호스트의 VM만 삭제
ansible-playbook -i inventory/all_hosts.yml playbooks/cleanup_vms.yml --limit kvm-host01
```

## SSH 접속 설정

`~/.ssh/config`에 다음 설정 추가:

```
Host kvm-host01
    HostName 192.168.0.50
    User ubuntu
    IdentityFile ~/.ssh/kvm-host02.pem

Host k8s-master-01
    HostName 10.200.1.10
    User ubuntu
    IdentityFile ~/.ssh/id_rsa
    ProxyJump kvm-host01

Host k8s-worker-*
    HostName 10.200.1.%h
    User ubuntu
    IdentityFile ~/.ssh/id_rsa
    ProxyJump kvm-host01
```

이후 간단히 접속:
```bash
ssh k8s-master-01
ssh k8s-worker-01
```

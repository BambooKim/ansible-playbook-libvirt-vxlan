# ansible-playbook-libvirt-vxlan

3개의 Ubuntu 머신에 libvirt KVM 가상화 환경을 구성하고, VXLAN으로 오버레이 네트워크를 연결하는 Ansible 플레이북

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

| 호스트 | 물리 IP | 오버레이 IP |
|--------|---------|-------------|
| kvm-host01 | 192.168.0.50 | 10.200.1.1 |
| kvm-host02 | 192.168.0.46 | 10.200.1.2 |
| kvm-host03 | 192.168.0.49 | 10.200.1.3 |

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

### 전체 배포

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
│   ├── hosts.yml
│   └── group_vars/
│       ├── all.yml
│       └── kvm_hosts.yml
├── roles/
│   ├── common/          # 공통 패키지, 시스템 설정
│   ├── vxlan/           # VXLAN 오버레이 네트워크
│   └── libvirt/         # KVM/libvirt 설치 및 구성
├── playbooks/
│   ├── site.yml         # 메인 플레이북
│   └── verify.yml       # 검증 플레이북
└── README.md
```

## 검증 명령어

배포 후 각 호스트에서 확인:

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

| 설정 | 값 |
|------|-----|
| VNI | 100 |
| VXLAN 포트 | 4789 |
| MTU | 1450 |
| 오버레이 네트워크 | 10.200.1.0/24 |
| libvirt 네트워크 | vxlan-network |
| 브리지 | br-vxlan |

RHEL 8.10에서 containerd가 이미 설치된 상태로 air-gapped(폐쇄망) 환경에서 Kubernetes를 kubeadm으로 설치하고 초기화(kubeadm init)하려면, 인터넷 연결 없이 미리 다운로드한 이미지를 활용해야 합니다. 성공을 위한 필수 요소는 다음과 같습니다. 이는 시스템 prerequisites, containerd 구성, 이미지 처리, 그리고 kubeadm 실행 측면으로 나눠 설명하겠습니다. (이 과정은 Kubernetes 공식 문서와 air-gapped 설치 사례를 기반으로 하며, RHEL 특성을 고려합니다.)

### 1. 시스템 Prerequisites 설정 (기본 요구사항 무시 시 실패 원인)
   - **Swap 비활성화**: Kubernetes는 swap을 사용하지 않도록 요구합니다. `swapoff -a` 명령 실행 후, `/etc/fstab` 파일에서 swap 관련 라인을 주석 처리하거나 제거하세요. 부팅 시 자동 적용되도록 하세요.
   - **커널 모듈 로드**: OverlayFS와 네트워킹을 위해 `modprobe overlay`와 `modprobe br_netfilter`를 실행하세요. `/etc/modules-load.d/containerd.conf` 파일에 추가해 부팅 시 로드되도록 하세요.
   - **sysctl 설정**: `/etc/sysctl.d/99-kubernetes-cri.conf` 파일에 다음을 추가하고 `sysctl --system`으로 적용:
     ```
     net.bridge.bridge-nf-call-iptables = 1
     net.ipv4.ip_forward = 1
     net.bridge.bridge-nf-call-ip6tables = 1
     ```
     이는 브릿지 트래픽과 IP forwarding을 허용합니다.
   - **SELinux 관리**: RHEL에서 SELinux가 enforcing 모드라면 `setenforce 0`으로 permissive 모드로 변경하세요. `/etc/selinux/config` 파일에서 `SELINUX=permissive`로 설정. (enforcing 모드에서 정책 오류 발생 가능)
   - **Firewall 관리**: firewalld가 활성화되어 있다면 `systemctl stop firewalld`와 `systemctl disable firewalld`로 중지하세요. 또는 Kubernetes 포트(예: 6443/TCP for API server, 10250/TCP for kubelet)를 열어주세요. air-gapped 환경에서 불필요한 포트 노출을 피하세요.

### 2. Containerd 구성 (CRI로서 제대로 동작해야 함)
   - **config.toml 설정**: `/etc/containerd/config.toml` 파일을 확인/편집하세요. 기본적으로 containerd 설치 후 `containerd config default > /etc/containerd/config.toml`로 생성. 필수 항목:
     - `[plugins."io.containerd.grpc.v1.cri"]` 섹션에서 `sandbox_image = "registry.k8s.io/pause:3.9"` (Kubernetes 버전에 맞는 pause 이미지 버전으로 설정, 예: v1.30+ 기준 3.9 이상).
     - Systemd cgroup 사용: `SystemdCgroup = true` (RHEL에서 cgroup v2 지원 시).
   - **서비스 재시작**: `systemctl restart containerd`로 적용. `systemctl status containerd`로 정상 작동 확인.
   - **CRI 소켓 확인**: `/run/containerd/containerd.sock`이 존재하고 listening 상태여야 합니다. kubeadm init 시 `--cri-socket unix:///run/containerd/containerd.sock` 옵션을 명시적으로 사용하세요.

### 3. Kubernetes 패키지 설치 (air-gapped 고려)
   - kubeadm, kubelet, kubectl 패키지를 미리 다운로드하세요. RHEL 8에서 Kubernetes 리포지토리(`/etc/yum.repos.d/kubernetes.repo`)를 설정하지만, air-gapped이므로 인터넷 머신에서 `yumdownloader --resolve kubeadm kubelet kubectl`로 RPM 파일을 다운로드해 옮기세요.
   - 설치: `dnf install -y <rpm-files>`로 설치 후, `systemctl enable --now kubelet`.
   - 버전 일치: containerd와 Kubernetes 버전 호환 확인 (예: Kubernetes v1.30+ with containerd 1.6+).

### 4. 이미지 처리 (air-gapped 핵심: 미리 다운로드한 이미지 import)
   - **필요 이미지 목록 확인**: 인터넷 머신에서 `kubeadm config images list --kubernetes-version v1.xx.xx` (원하는 Kubernetes 버전으로)로 목록 추출. 일반적으로 포함: kube-apiserver, kube-controller-manager, kube-scheduler, kube-proxy, etcd, coredns, pause.
   - **다운로드 및 export (인터넷 머신에서)**: containerd 사용 시 `ctr -n k8s.io images pull <image>`로 pull 후, `ctr -n k8s.io images export <image.tar> <image>`로 tar 파일 생성. (k8s.io 네임스페이스 사용 필수)
   - **import (air-gapped 머신으로)**: tar 파일을 옮긴 후 `ctr -n k8s.io images import <image.tar>`로 로드. 모든 이미지(포함 pause)를 import하세요. `ctr -n k8s.io images ls`로 확인.
   - **추가 이미지**: CNI(예: Flannel 또는 Calico) 관련 이미지와 workload 이미지도 미리 import. pause 이미지가 없으면 pod 생성 실패.
   - **주의**: 이미지가 registry.k8s.io 태그로 import되어야 kubeadm이 로컬에서 인식. pull 실패 무시 옵션: kubeadm init 시 `--ignore-preflight-errors=ImagePull` 추가 가능하지만, 이미지가 없으면 실패.

### 5. kubeadm 초기화 실행
   - 명령 예시: `kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket unix:///run/containerd/containerd.sock --kubernetes-version v1.xx.xx --ignore-preflight-errors=all` (CIDR은 CNI에 맞게 조정, 예: Flannel용 10.244.0.0/16).
   - 성공 후: 생성된 kubeconfig를 `mkdir -p $HOME/.kube; cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`로 설정.
   - 워커 노드 조인: 마스터에서 생성된 `kubeadm join` 토큰 사용 (air-gapped 워커에도 동일 prerequisites 적용).
   - CNI 적용: air-gapped이므로 CNI manifest YAML 파일을 미리 다운로드해 `kubectl apply -f <cni.yaml>`로 설치. CNI 이미지들도 import 필수.

이 요소들을 모두 충족하면 kubeadm init이 성공할 가능성이 높습니다. 실패 시 로그(`/var/log/kubelet.log`, `journalctl -u kubelet`) 확인하세요. RHEL 8.10은 cgroup v2를 지원하니 containerd와 호환 좋습니다. 추가로, etcd나 CoreDNS 이미지 누락이 흔한 오류입니다.
RHEL 8.10에서 containerd가 설치된 상태에서 폐쇄망(오프라인) 환경에서 kubeadm으로 Kubernetes 클러스터 초기화를 성공하려면 다음 필수 조건들이 있습니다.

## 이미지 사전 준비 및 로드
- 인터넷이 닿는 환경에서 Kubernetes 핵심 컴포넌트 이미지(kube-apiserver, kube-controller-manager, kube-scheduler, etcd, pause, coredns, kube-proxy 등)를 미리 다운로드하여 tar 파일로 저장하고 폐쇄망 서버에 전달 후 containerd에 Import하여 로드해야 합니다.
- 예) `sudo ctr -n k8s.io images import <image>.tar` 명령어 사용.[1][2]

## containerd 및 CRI 설정 확인
- kubeadm 초기화시 containerd 소켓 경로(`criSocket`)가 정확히 지정되어야 하며 보통 `/var/run/containerd/containerd.sock` 또는 `/run/containerd/containerd.sock` 이어야 합니다.
- containerd가 CRI 기능을 제대로 지원하고 활성화 되어 있는지 확인. config.toml에서 `disabled_plugins=["cri"]`와 같은 설정은 제거하거나 주석 처리하고 재시작 필요.[3]

## 시스템 설정 
- br_netfilter 커널 모듈이 로드되어 있어야 하며, 네트워크 브리징 시 필터링을 위해 활성화 필요 (`lsmod | grep br_netfilter` 확인, 없으면 `modprobe br_netfilter`)[4].
- swap 비활성화(`swapoff -a`) 및 다음 sysctl 설정 적용 필요:
  - `net.bridge.bridge-nf-call-ip6tables=1`
  - `net.bridge.bridge-nf-call-iptables=1`
- 방화벽과 SELinux 설정을 Kubernetes 클러스터에 맞게 조정하거나 비활성화 검토.[5]

## kubeadm 초기화 설정
- 적절한 kubeadm 초기화 설정 파일을 사용하여 `kubeadm init --config=kubeadm-config.yaml` 형태로 실행.
- 해당 YAML 파일에는 advertiseAddress, podNetworkCIDR, criSocket 등 환경과 일치하는 정보가 꼭 들어가야 함.
- podNetworkCIDR은 이후 CNI 플러그인과 일치해야 함(예: flannel은 10.244.0.0/16).[3]

## 네트워크 및 CNI 플러그인
- 패킷 필터링과 네트워크 설정이 초기화시 반드시 고려되어야 하며, flannel, calico 같은 CNI 플러그인을 폐쇄망에 맞게 설치 및 설정해야 함.
- podNetworkCIDR 값과 CNI 플러그인 설정이 맞지 않으면 클러스터가 제대로 동작하지 않음.[6][3]

## 요약
- 폐쇄망용 Kubernetes 이미지 사전 다운로드 및 containerd에 이미지 Import 필수.
- containerd CRI 설정 활성화 및 kubeadm 초기화 구성파일 내 criSocket 설정 정확.
- 네트워크 커널 모듈 및 시스템 설정(swap off, br_netfilter 등) 필수.
- kubeadm 초기화 시 pod 네트워크 CIDR과 CNI 플러그인 일치 및 네트워크 설정 완비.
- 방화벽, SELinux 등 보안 정책 조정 필요.

위 사항을 모두 준비하고 맞춰야 RHEL 8.10 환경에서 containerd 기반 Kubernetes를 폐쇄망에서 kubeadm으로 성공적으로 초기화할 수 있습니다.[2][4][1][5][3]

[1](https://lumenvox.capacity.com/article/153332/kubeadm-setup-on-offline-server)
[2](https://kubernetes.io/blog/2023/10/12/bootstrap-an-air-gapped-cluster-with-kubeadm/)
[3](https://velog.io/@johnsuhr4542/Kubeadm%EC%9D%84-%EC%82%AC%EC%9A%A9%ED%95%98%EC%97%AC-%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EA%B5%AC%EC%B6%95%ED%95%B4%EB%B3%B4%EA%B8%B0)
[4](https://majjangjjang.tistory.com/215)
[5](https://kmaster.tistory.com/71)
[6](https://terianp.tistory.com/177)
[7](https://hxgo.tistory.com/7)
[8](https://linuxias.github.io/cloud/kubernetes_install/)
[9](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
[10](https://www.armosec.io/glossary/air-gapped-kubernetes/)
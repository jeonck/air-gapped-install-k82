폐쇄망 환경에서 containerd 및 소켓 경로 사전 조건과 이미지 로드가 제대로 되어 있어도 kubeadm 초기화 실패 가능성이 있는 추가 점검 사항은 다음과 같습니다.

## 1. 이미지 불일치 및 태그 문제
- kubeadm이 요구하는 이미지 버전과 정확히 일치하는 이미지가 containerd에 로드되어 있어야 합니다.
- 특히 pause 이미지(pause:3.9 권장)와 kube-apiserver, etcd, coredns 등 핵심 컴포넌트 이미지 버전이 완전히 일치하는지 확인해야 합니다.
- 로컬 레지스트리를 만들어 이미지를 태그 및 밀어넣기(push) 한 경우에도 태그오류가 없도록 주의.[1][2]

## 2. 이미지 풀링 정책(ImagePullPolicy)
- 모든 관련 Pod들의 ImagePullPolicy가 `IfNotPresent` 또는 `Never`로 설정되어 오프라인에서 이미 로드된 이미지를 재사용하는지 확인해야 합니다.
- 기본이 Always로 되어 있으면 인터넷 접속 시도 중 실패로 초기화 실패 가능.

## 3. kubelet 및 API 서버 통신 문제
- kubelet이 정적 Pod(manifest)들을 제대로 가져와 실행하는지 확인.
- 초기화 로그에서 `Waiting for the API server to be healthy` 구간에서 멈출 경우, API 서버 Pod가 제대로 올라오지 않은 것임.
- kubelet 로그(`journalctl -xeu kubelet`)를 확인해 인증서 오류, 네트워크 문제, 권한 문제 등이 없는지 점검.

## 4. 네트워크 관련 점검
- br_netfilter 커널 모듈 및 sysctl 네트워크 설정이 올바른지 재확인.
- 방화벽과 SELinux 설정에 의해 필요한 포트(6443 등)가 차단되지 않았는지 확인.[1]

## 5. DNS 설정 확인
- kubeadm 초기화 중 클러스터 내부 DNS가 제대로 동작하지 않으면 여러 에러 발생 가능.
- 특히 폐쇄망 환경이라 DNS 관련 설정을 미리 잘 맞춰야 함.

## 6. 필요한 시스템 패키지 누락 여부
- iptables, socat, ebtables 등 Kubernetes 운영에 필요한 도구들이 모두 설치되어 있어야 함.

종합적으로 이미지 태그 및 로딩 문제, 네트워크 및 DNS 설정, kubelet 상태 및 로그 확인, 이미지 풀 정책 점검이 폐쇄망 kubeadm 초기화 실패 주요 원인입니다. 이들 모두를 꼼꼼히 체크해야 성공 확률을 높일 수 있습니다.[3][2][1]

[1](https://kubernetes.io/blog/2023/10/12/bootstrap-an-air-gapped-cluster-with-kubeadm/)
[2](https://stackoverflow.com/questions/76801956/kubernetes-1-27-4-init-fails-for-offline-environment)
[3](https://github.com/kubernetes/kubeadm/issues/742)
[4](https://github.com/kubernetes/kubeadm/issues/1041)
[5](https://github.com/kubernetes/kubeadm/issues/2370)
[6](https://discuss.kubernetes.io/t/help-with-setting-air-gapped-kubernetes-cluster/25812)
[7](https://www.linkedin.com/pulse/install-docker-setup-kubernetes-cluster-offline-using-ayaz-khan)
[8](https://github.com/kubernetes-sigs/kubespray/issues/11004)
[9](https://www.reddit.com/r/kubernetes/comments/1buz5fh/getting_the_status_error_errimageneverpull_but/)
[10](https://lumenvox.capacity.com/article/153332/kubeadm-setup-on-offline-server)
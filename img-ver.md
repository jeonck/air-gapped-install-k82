Kubernetes 1.30 버전 kubeadm 초기화에 필요한 이미지 버전 정보는 다음과 같습니다.

- kube-apiserver: registry.k8s.io/kube-apiserver:v1.30.3
- kube-controller-manager: registry.k8s.io/kube-controller-manager:v1.30.3
- kube-scheduler: registry.k8s.io/kube-scheduler:v1.30.3
- kube-proxy: registry.k8s.io/kube-proxy:v1.30.3
- coredns: registry.k8s.io/coredns/coredns:v1.11.1
- pause: registry.k8s.io/pause:3.9
- etcd: registry.k8s.io/etcd:3.5.12-0

이 이미지를 사전에 폐쇄망 환경에 맞게 다운로드 및 containerd에 import 하여야 kubeadm init 과정이 원활합니다.[1]

[1](https://learn-k8s-from-scratch.readthedocs.io/en/latest/k8s-install/kubeadm.html)
[2](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
[3](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
[4](https://github.com/kubernetes/kubernetes/issues/127350)
[5](https://github.com/kubernetes/kubernetes/issues/132618)
[6](https://inganyoyo.tistory.com/53)
[7](https://kubernetes.io/docs/tasks/access-application-cluster/list-all-running-container-images/)
[8](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)
[9](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
[10](https://kubernetes.io/blog/2024/04/17/kubernetes-v1-30-release/)
[11](https://github.com/kubernetes/kubernetes/issues/126170)
[12](https://devocean.sk.com/blog/techBoardDetail.do?ID=166956&boardType=techBlog)
[13](https://dev-yubin.tistory.com/187)
[14](https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
[15](https://velog.io/@beomjin/kubernetes-upgrade)
[16](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-config/)
[17](https://league-cat.tistory.com/405)
[18](https://every-up.tistory.com/88)
권한 문제 체크 방법
Kubernetes 및 containerd 설치 후 kubeadm 초기화 시 발생할 수 있는 권한 문제를 체크하는 방법은 다음과 같습니다.

## 1. kubelet 로그에서 권한 오류 확인
```
journalctl -xeu kubelet
```
- kubelet 서비스의 최근 로그를 보고 Permission denied, 인증서 불일치, 권한 부족 메시지를 찾습니다.

## 2. containerd 프로세스 권한 확인
```
ps aux | grep containerd
```
- containerd가 root 권한으로 실행 중인지 확인합니다. 일반적으로는 root 권한이어야 정상 작동합니다.

## 3. 컨테이너 런타임 소켓 파일 권한 확인
```
ls -l /var/run/containerd/containerd.sock
```
- 소켓 파일이 kubeadm 및 kubelet에서 접근 가능한 권한(주로 소유자가 root이고, 그룹과 사용자가 액세스 가능)인지 봅니다.
- 권한 문제 시 아래 명령어로 권한 조정 가능:
  ```
  sudo chown root:root /var/run/containerd/containerd.sock
  sudo chmod 660 /var/run/containerd/containerd.sock
  ```

## 4. kubeadm 실행 권한 확인
- kubeadm 명령은 root 또는 sudo 권한으로 실행되어야 합니다.

## 5. SELinux 상태 확인 및 설정
```
getenforce
```
- SELinux가 Enforcing 상태일 경우 권한 문제 발생 가능.
- 테스트 용으로는 Permissive 또는 Disabled로 설정 후 재시도:
  ```
  sudo setenforce 0
  ```
- 영구 변경은 `/etc/selinux/config` 파일 수정.

## 6. 디렉터리 및 파일 권한 점검
- `/etc/kubernetes/`, `/var/lib/kubelet/`, `/var/lib/etcd/` 등 Kubernetes 관련 디렉터리 권한이 kubeadm 및 kubelet 프로세스가 접근 가능해야 함.

위 점검 항목들을 순서대로 확인하면 권한 문제 여부와 원인을 찾을 수 있습니다. 특히 로그에서 권한 관련 메시지가 매우 중요한 단서가 됩니다.Kubernetes 및 containerd 설치 후 kubeadm 초기화 시 발생할 수 있는 권한 문제를 점검하기 위한 주요 방법은 다음과 같습니다.

1. kubelet 로그 확인  
```
journalctl -xeu kubelet
```
- kubelet 실행 중 권한 부족, Permission denied 관련 로그 메시지 여부 확인.

2. containerd 프로세스 권한 확인  
```
ps aux | grep containerd
```
- containerd가 root 권한으로 실행 중인지 점검.

3. containerd 소켓 파일 권한 확인  
```
ls -l /var/run/containerd/containerd.sock
```
- 소켓 파일의 소유자 및 권한 확인. 일반적으로 root:root 소유에 660 권한이 적절함.  
- 필요 시 권한 수정:  
```
sudo chown root:root /var/run/containerd/containerd.sock  
sudo chmod 660 /var/run/containerd/containerd.sock
```

4. kubeadm 실행 권한  
- kubeadm 명령은 반드시 root 또는 sudo 권한으로 실행해야 함.

5. SELinux 상태 점검 및 일시적 비활성화  
```
getenforce
sudo setenforce 0
```
- SELinux가 Enforcing 상태라면 권한 문제 유발 가능. Permissive 상태로 변경 후 테스트 권장.

6. Kubernetes 관련 디렉터리 권한 점검  
- `/etc/kubernetes/`, `/var/lib/kubelet/`, `/var/lib/etcd/` 등 디렉터리에 kubeadm/kubelet 접근 가능 여부 확인.

이 점검들을 통해 권한 문제를 진단하고 해결할 수 있습니다. 특히 로그에서 권한 관련 에러 메시지를 꼼꼼히 살펴보는 것이 중요합니다.
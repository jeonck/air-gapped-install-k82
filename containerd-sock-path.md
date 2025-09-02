containerd 및 CRI 설정과 kubeadm 초기화 시 사용할 containerd 소켓 경로(criSocket)를 정확히 확인하기 위한 명령어는 다음과 같습니다.

## 1. containerd 서비스 상태 확인
```
systemctl status containerd
```
- containerd가 제대로 실행 중인지 확인합니다.

## 2. containerd 소켓 파일 존재 여부 확인
```
ls -l /var/run/containerd/containerd.sock
ls -l /run/containerd/containerd.sock
```
- containerd socket 파일이 어느 경로에 있는지 확인합니다.

## 3. kubeadm 초기화 시 사용하는 criSocket 설정 확인
- kubeadm 명령어에 명시적으로 지정한 소켓경로를 확인합니다.
  보통 `kubeadm init --cri-socket /var/run/containerd/containerd.sock` 형태로 실행하므로 초기화 커맨드에서 criSocket 경로를 확인하거나 아래와 같이 설정파일을 확인합니다.

```
cat <kubeadm-config.yaml 파일 경로> | grep criSocket
```

- 혹은 기본값으로 `unix:///var/run/containerd/containerd.sock` 가 사용됩니다.

## 4. containerd CRI 상태 확인
containerd가 올바른 CRI를 지원하고 있는지 확인하려면 containerd config 파일 확인:

```
cat /etc/containerd/config.toml | grep disabled_plugins -A 3
```
- disabled_plugins에 "cri"가 포함되어 있으면 안 됩니다.

## 5. containerd 현재 사용 중인 소켓 경로 조회 (추가 확인용)
```
sudo crictl config view
```
- 이 명령으로 crictl이 사용하는 CRI socket 경로를 확인할 수 있습니다.

위 명령어를 통해 containerd의 실행 상태, 소켓 파일 위치, kubeadm 초기화에서 사용할 소켓 경로를 정확히 확인할 수 있습니다.
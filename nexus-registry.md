# RHEL 8.10 에어갭 환경에서 Nexus 설치 및 kubeadm 연계 절차서

이 문서는 RHEL 8.10 에어갭 환경에서 물리 서버인 마스터 노드에 Nexus를 설치하고, kubeadm init에서 사용할 수 있도록 연계하는 절차를 단계별로 설명합니다.

## 1단계. 설치 준비

- **자바 설치 확인 및 설치**  
  RHEL 8.10 기본적으로 OpenJDK 11 이상 설치를 권장합니다.  
  ```bash
  java -version
  ```
  자바가 없거나 버전이 낮으면 설치합니다:
  ```bash
  sudo dnf install java-11-openjdk-devel
  ```

- **Nexus 설치 파일 확보**  
  인터넷이 차단된 에어갭 환경이므로 Nexus 설치 파일(`nexus-3.x.y-xx-unix.tar.gz`)을 인터넷 연결 가능한 다른 환경에서 미리 다운로드하여 마스터 노드로 전송합니다.

***

## 2단계. Nexus 설치 및 서비스 구성

- **압축 해제 및 사용자 생성**  
  ```bash
  sudo tar -zxvf nexus-3.x.y-xx-unix.tar.gz -C /opt/
  sudo useradd nexus
  sudo chown -R nexus:nexus /opt/nexus-3.x.y-xx /opt/sonatype-work
  ```

- **systemd 서비스 파일 작성**  
  `/etc/systemd/system/nexus.service` 파일을 생성합니다:
  ```
  [Unit]
  Description=Nexus Repository Manager
  After=network.target

  [Service]
  Type=forking
  User=nexus
  ExecStart=/opt/nexus-3.x.y-xx/bin/nexus start
  ExecStop=/opt/nexus-3.x.y-xx/bin/nexus stop
  Restart=on-failure

  [Install]
  WantedBy=multi-user.target
  ```

- **서비스 등록 및 실행**  
  ```bash
  sudo systemctl daemon-reload
  sudo systemctl start nexus
  sudo systemctl enable nexus
  ```

- **포트 및 방화벽 설정**  
  기본 넥서스 웹 UI 8081 포트를 개방합니다:
  ```bash
  sudo firewall-cmd --add-port=8081/tcp --permanent
  sudo firewall-cmd --reload
  ```

***

## 3단계. 넥서스에서 Docker Registry 저장소 생성

- 넥서스 웹 UI에 접속합니다: `http://<마스터노드 IP>:8081`
- 로그인 후 "Repositories" 메뉴에서 `Create repository` 클릭합니다.
- `docker (hosted)` 레포지토리 유형을 선택합니다.
- HTTP 또는 HTTPS 설정, 포트 예: `5000`을 지정합니다 (에어갭 환경에서는 HTTPS 설정을 권장합니다).
- 저장소 생성이 완료되면 Docker 레지스트리 역할을 수행할 수 있습니다.

***

## 4단계. 필요한 Kubernetes 이미지 준비 및 업로드

- 인터넷 환경에서 아래 명령어로 필요한 컨테이너 이미지 리스트를 확보합니다:
  ```bash
  kubeadm config images list
  ```

- 각 이미지를 인터넷이 되는 환경에서 Docker pull 후 save합니다:
  ```bash
  docker pull registry.k8s.io/kube-apiserver:vX.Y.Z
  docker save registry.k8s.io/kube-apiserver:vX.Y.Z -o kube-apiserver_vX.Y.Z.tar
  ```

- 넥서스 마스터 노드에 이미지 tar 파일을 전송한 후 load합니다:
  ```bash
  docker load -i kube-apiserver_vX.Y.Z.tar
  ```

- 이미지 태그를 넥서스 레지스트리 주소로 변경하여 push합니다:
  ```bash
  docker tag registry.k8s.io/kube-apiserver:vX.Y.Z <nexus 도메인>:5000/kube-apiserver:vX.Y.Z
  docker push <nexus 도메인>:5000/kube-apiserver:vX.Y.Z
  ```

***

## 5단계. 마스터 및 워커 노드 Container Runtime 설정

- 컨테이너 런타임(containerd 또는 Docker) 설정에서 넥서스 레지스트리를 신뢰하는 미러로 등록합니다.  
  예) containerd `/etc/containerd/config.toml`
  ```toml
  [plugins."io.containerd.grpc.v1.cri".registry]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."<nexus 도메인>:5000"]
      endpoint = ["https://<nexus 도메인>:5000"]
  ```

- 필요한 CA 인증서가 있다면 각 노드에 신뢰하도록 설치합니다.

***

## 6단계. kubeadm init 실행 시 넥서스 레지스트리 경로 지정

- kubeadm init 명령어에 이미지 레포지토리 경로 옵션을 추가합니다:
  ```bash
  kubeadm init --image-repository <nexus 도메인>:5000 ...
  ```

- 설치가 넥서스에서 이미지를 직접 pull 하여 진행됩니다.

***

이 절차를 따르면 RHEL 8.10 폐쇄망 마스터 서버에 Nexus를 설치하고, 에어갭 환경에서 kubeadm을 통한 쿠버네티스 초기화를 넥서스를 사설 레지스트리로 연동해 실행할 수 있습니다.

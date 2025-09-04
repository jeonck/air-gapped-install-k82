# 에어갭 환경에서 사설 레지스트리 구축 및 kubeadm init 절차서

## 목적
에어갭(인터넷 완전 차단) 환경에서 Kubernetes 클러스터를 구축하기 위해 도메인명 기반 사설 레지스트리를 설정하고, kubeadm init을 수행하는 절차를 설명한다.

## 범위
- 사설 레지스트리 구축 및 설정
- 워커노드에서 마스터노드의 사설 레지스트리 접근
- kubeadm init 실행을 위한 필수 구성

## 절차

### 1. 레지스트리 컨테이너 준비 및 인증서 생성

#### 1.1 DNS 또는 hosts 파일 설정
레지스트리 서버의 IP를 도메인명(FQDN, 예: registry.my-master.domain)으로 접근하도록 설정한다. 모든 노드의 `/etc/hosts` 파일에 아래와 같이 추가:
```
<Master Node IP> registry.my-master.domain
```
내부 DNS를 사용할 경우, 동일한 매핑을 DNS 서버에 등록한다.

#### 1.2 TLS 인증서 생성
`openssl`을 사용하여 사설 레지스트리용 TLS 인증서(`domain.crt`, `domain.key`)를 생성한다. 인증서의 Common Name(CN)을 `registry.my-master.domain`으로 지정한다.
```bash
openssl req -newkey rsa:2048 -nodes -keyout domain.key -x509 -days 365 -out domain.crt -subj "/CN=registry.my-master.domain"
```
생성된 인증서를 `/opt/registry/certs` 디렉터리에 저장한다.

### 2. Docker Private Registry 컨테이너 실행

#### 2.1 필요 디렉터리 및 파일 준비
인증서(`/opt/registry/certs`), 이미지 저장소(`/opt/registry/store`), 인증 파일(`/opt/registry/auth`) 디렉터리를 생성한다.
(선택) 기본 인증 설정을 위해 htpasswd 파일을 생성:
```bash
htpasswd -Bc /opt/registry/auth/htpasswd <username> <password>
```

#### 2.2 레지스트리 컨테이너 실행
아래 명령으로 Docker 레지스트리 컨테이너를 실행한다:
```bash
docker run -d -p 5000:5000 --restart=always --name registry \
  -v /opt/registry/certs:/certs \
  -v /opt/registry/store:/var/lib/registry \
  -v /opt/registry/auth:/auth \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM=Registry-Realm \
  registry:2
```

### 3. 이미지 준비 및 업로드

#### 3.1 인터넷 연결 환경에서 이미지 수집
인터넷 연결이 가능한 환경에서 kubeadm에 필요한 이미지를 확인:
```bash
kubeadm config images list
```
필요한 이미지를 Docker로 다운로드:
```bash
docker pull registry.k8s.io/kube-apiserver:vX.Y.Z
docker save registry.k8s.io/kube-apiserver:vX.Y.Z -o kube-apiserver.tar
```

#### 3.2 이미지 업로드
`.tar` 파일을 에어갭 환경의 레지스트리 서버로 복사. 파일을 로드하고 사설 레지스트리로 태그 변경 후 푸시:
```bash
docker load -i kube-apiserver.tar
docker tag registry.k8s.io/kube-apiserver:vX.Y.Z registry.my-master.domain:5000/kube-apiserver:vX.Y.Z
docker push registry.my-master.domain:5000/kube-apiserver:vX.Y.Z
```
모든 필수 이미지에 대해 위 과정을 반복한다.

### 4. kubeadm 및 컨테이너 런타임 설정

#### 4.1 컨테이너 런타임 설정

**containerd 설정**
`/etc/containerd/config.toml` 파일에 사설 레지스트리 설정 추가:
```toml
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.my-master.domain:5000"]
      endpoint = ["https://registry.my-master.domain:5000"]
```
containerd 재시작:
```bash
systemctl restart containerd
```

**Docker 설정**
`/etc/docker/daemon.json`에 사설 레지스트리 추가:
```json
{
  "insecure-registries": ["registry.my-master.domain:5000"]
}
```
Docker 재시작:
```bash
systemctl restart docker
```

#### 4.2 TLS 인증서 신뢰 설정
모든 노드에서 사설 레지스트리 인증서(`domain.crt`)를 신뢰하도록 시스템 CA에 추가:
```bash
cp domain.crt /usr/local/share/ca-certificates/
update-ca-certificates
```

#### 4.3 kubeadm 설정
kubeadm 설정 파일(`init.yaml`)에 사설 레지스트리 경로 지정:
```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "<Master Node IP>"
  bindPort: 6443
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
imageRepository: registry.my-master.domain:5000
```

### 5. kubeadm init 수행

#### 5.1 클러스터 초기화
사설 레지스트리의 모든 이미지가 준비된 후, 아래 명령으로 클러스터 초기화:
```bash
kubeadm init --config init.yaml
```

#### 5.2 워커노드 조인
`kubeadm init` 완료 후 생성된 조인 명령어를 워커노드에서 실행:
```bash
kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### 6. 워커노드 설정

#### 6.1 DNS 또는 hosts 파일 설정
워커노드의 `/etc/hosts`에 마스터노드의 사설 레지스트리 도메인 추가:
```
<Master Node IP> registry.my-master.domain
```

#### 6.2 인증서 신뢰 설정
워커노드에서도 마스터노드의 `domain.crt`를 시스템 CA에 추가:
```bash
cp domain.crt /usr/local/share/ca-certificates/
update-ca-certificates
```

#### 6.3 컨테이너 런타임 설정
워커노드에서도 containerd 또는 Docker에 사설 레지스트리 설정을 적용(4.1 참조).

## 주요 참고 사항
- **도메인명 일관성**: 모든 노드에서 사설 레지스트리 도메인명(`registry.my-master.domain`)이 동일한 IP로 매핑되어야 한다.
- **인증서 신뢰**: 모든 노드에서 사설 레지스트리의 TLS 인증서를 신뢰하도록 설정해야 이미지 pull이 가능하다.
- **에러 방지**: kubeadm, 컨테이너 런타임, 인증서 설정 누락 시 이미지 pull 실패로 클러스터 초기화가 중단될 수 있다.
- **도메인명 사용 이점**: 도메인명을 사용하면 IP 변경 시에도 설정 수정이 간편하다.

## 검증
- 사설 레지스트리에서 이미지 pull 테스트:
```bash
docker pull registry.my-master.domain:5000/kube-apiserver:vX.Y.Z
```
- 클러스터 상태 확인:
```bash
kubectl get nodes
```

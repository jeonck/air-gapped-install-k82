Kubernetes kubeadm 초기화 시 자주 발생하는 인증서 오류에 대한 대책은 다음과 같습니다.

## 1. 인증서 및 키 파일 삭제 후 재생성
- 기존 인증서 파일이 손상되었거나 만료되었을 수 있으므로 초기화 전에 기존 인증서 삭제 후 재생성 권장.
- 관련 파일 경로: `/etc/kubernetes/pki/`
- 삭제 명령 예:
  ```
  sudo rm -rf /etc/kubernetes/pki
  sudo kubeadm reset
  ```
- 이후 다시 `kubeadm init` 실행 시 자동으로 새 인증서 생성.

## 2. 클럭 동기화 확인
- 인증서 오류는 마스터 노드 및 모든 워커 노드의 시스템 시간이 다르면 발생 가능.
- NTP 서비스 또는 chrony 등을 이용해 시간 동기화 필수:
  ```
  timedatectl set-ntp true
  chronyc sources
  ```
- 시간이 크게 벗어나면 인증서 검증 실패 발생.

## 3. 인증서 만료 여부 확인
- 인증서 유효기간 확인:
  ```
  openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
  ```
- 만료 시 재생성이 필요.

## 4. 인증서 사용자 및 권한 확인
- `/etc/kubernetes/pki/` 내 파일 권한이 kubeadm과 kubelet이 접근 가능해야 함.
- 권한 예:
  ```
  sudo chown -R root:root /etc/kubernetes/pki
  sudo chmod -R 600 /etc/kubernetes/pki
  ```

## 5. kubeadm 초기화 시 올바른 API 서버 광고 주소 설정
- `--apiserver-advertise-address` 또는 config 파일 내 advertiseAddress를 정확한 IP로 지정해야 인증서 SAN에 포함되어 인증서 오류 방지 가능.

## 6. 인증서 관련 로그 확인
- kube-apiserver, kubelet 로그에서 인증서 관련 오류 메시지를 확인하여 원인 분석:
  ```
  journalctl -u kubelet
  docker logs kube-apiserver
  ```

위 대응책들을 점검하면 kubeadm 초기화 또는 클러스터 운영 중 발생하는 인증서 오류 문제를 효과적으로 해결할 수 있습니다. 특히 시간 동기화와 인증서 재생성이 가장 빈번한 대책입니다.
# 다중 VM 네트워크 및 자동 배포

### vmWare 기반의 다중 VM 환경 구성, SSH 보안 접속, 파일 변경 감지를 통한 자동 재실행 프로세스를 정리한 학습 기록 입니다.
---

## 다중 VM 네트워크 설정

여러 대의 서버가 서로 통신하고 외부 인터넷을 사용하기 위한 네트워크 아키텍처를 구성

- **NAT(Network Address Translation)** : VM이 호스트의 IP를 빌려 외부 인터넷에 접속하는 방식

- Static IP 설정: 서버 간 통신 시 IP가 변하지 않도록 netplan을 통한 수동 IP 할당이 필요함

- Hostname & Hosts: IP 대신 server01, server02와 같은 이름으로 식별 가능

## VM 네트워크 구성 정보

| 서버명   | 역할            | IP (내부)   | 포트 포워딩 (SSH)        |
|----------|-----------------|-------------|--------------------------|
| server01 | Master Node     | 10.0.2.15   | Host 2015 → Guest 22     |
| server02 | Worker Node 1   | 10.0.2.20   | Host 2020 → Guest 22     |

## 고정 IP 설정 (Ubuntu Netplan)

**/etc/netplan/01-netcfg.yaml** 변경을 통해 고정 IP 적용 가능

```bash
sudo nano /etc/netplan/01-netcfg.yaml

network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3: # 네트워크 인터페이스명이 다를 수 있으므로 확인 필요 (ip addr, ip a..)
      dhcp4: no
      addresses:
        - 10.0.2.20/24
      routes:
        - to: default
          via: 10.0.2.2
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]

sudo netplan apply
```
---

## Host 이름 변경 및 접속 설정

### Host 이름 변경

```bash
# host이름 확인
cat /etc/hostname

server01 
sudo hostnamectl set-hostname server01

server02
sudo hostnamectl set-hostname server02

# 변경된 host이름 확인
cat /etc/hostname

# 실제 적용을 위한 재부팅
sudo init 6
```

### Host name 접속 설정

```bash
sudo nano /etc/hosts

127.0.0.1 localhost
10.0.2.15 server01

10.0.2.20 server02
```
---

## SSH 키 설정 및 다른 사용자와 네트워크 공유

### SSH 키 설정

- 비밀번호 없이 파일을 전송할 경우 로컬 서버에서 SSH 키를 생성 후 공유하여 설정 가능함

```bash
cd .ssh
ls
cat authorized_keys

ssh-keygen -t rsa -b 4096 # RSA 방식의 SSH 키를 4096비트로 생성

생성 후 확인
 ls -al .ssh
```

- 개인키 파일명: ~/.ssh/id_rsa
- 공개키 파일명: ~/.ssh/id_rsa.pub

### 생성된 SSH 키 공유

```
server01에서 key 생성 후 server02 key 파일에 추가
server01의 공개키를 server02에 저장 
server01에서 server02로 로그인시에는 별도의 pw입력없이 접속 가능

ssh-copy-id ubuntu@server02
cat .ssh/authorized_keys
ssh ubuntu@server02  


예시 : 10.0.2.20 or server02 host 명으로 접속 해 보기 
ssh ubuntu@10.0.2.20
ssh ubuntu@server02
```

### 파일 공유

```bash
scp -P 포트명 /공유할/파일/경로 ubuntu@10.0.2.20:~저장할/파일/경로
```
---

## File 변경 감지

리눅스의 파일 시스템 이벤트 감지 기능인 inotify를 활용하여 파일 및 디렉터리의 변경 사항을 실시간으로 모니터링

- 주요 도구
  - **ignotifywait**: 이벤트 발생 시까지 대기 후 출력 및 스트림 방식으로 지속 감지
  - **ignotifywatch**: 일정 시간 동안 이벤트 수집 후 통계 출력 및 지정된 디렉토리에서 발생한 이벤트를 모니터링하고 통계 정보를 출력

### ignotify-tools 설치 및 사용 예시

```bash
sudo apt-get update
sudo apt-get install inotify-tools


파일 수정 감지
-m : 지속 감시(monitor mode)
     무한 모드로 이벤트 발생할 떄까지 대기
close_write : 파일 쓰기 완료 시점 감지

inotifywait -m -e close_write /path/to/your/file


디렉토리 내 모든 파일에서 이벤트 감지
inotifywait -m -e modify,create,delete /path/to/your/directory


지정된 디렉토리나 파일에서 발생한 이벤트를 모니터링하고, 그 결과를 출력
이벤트가 발생한 후 통계 출력
inotifywatch -v -r /path/to/your/directory
```

### 사용 예시(자동화 스크립트 예시)

```sh
bash
#!/usr/bin/env bash

FISA_FILE="./fisa.txt"
COOLDOWN=10   # 10초 쿨다운
LAST_RUN=0

inotifywait -m -e close_write "$FISA_FILE" | while read path action file; do
  NOW=$(date +%s)
  if (( NOW - LAST_RUN < COOLDOWN )); then
    echo "쿨다운 중, 작업 스킵"
    continue
  fi

  echo "파일 변경 감지: $file (action: $action)"

  # TODO: 여기에서 실제 실행할 작업을 정의 (예: jar 재실행)
  # 예:
  # pkill -f myapp.jar || true
  # java -jar /path/to/myapp.jar &

  LAST_RUN=$NOW
done
```
---

### 트러블 슈팅

#### vmWare 설정 문제: vmWare에서 네트워크 설정을 할 때, gateway 설정 시, D 클래스를 1로 하였으나 vmWare 워크스테이션에서는 D 클래스를 2로 설정해야 했던 오류가 있었음

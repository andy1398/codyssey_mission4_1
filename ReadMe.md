# 미션 개요
기본 보안 강화: SSH 접속 포트 변경(20022) 및 Root 원격 로그인 차단[cite: 2, 3, 6, 7]
네트워크 보안: UFW 방화벽 활성화를 통한 필요 최소 인바운드 포트만 허용[cite: 4]
최소 권한 원칙 (RBAC): 역할 기반 그룹/계정 분리 및 보안 디렉터리 접근 제어[cite: 8, 9, 10, 12]
애플리케이션 구동: 일반 계정 기반 5단계 Boot Sequence 통과 (Agent READY)[cite: 16, 17]
관제 자동화: 프로세스/포트 Health Check, 자원 수집, 임계값 경고 및 로그 롤링 구현
주기적 실행: crontab 매분 실행을 통한 관제 데이터 누적 및 자동화 검증[cite: 4]

# 개발 및 실습 환경
OS : Ubuntu 22.04 LTS (Multipass Virtual Machine)
Resources : 2 Cores / 2 GB RAM / 10 GB Disk
Shell : Bash (Bourne-Again SHell)
Daemon/Tool : OpenSSH Server, UFW, Cron, top, free, df

# 디렉토리 구조
```
/home/agent-admin/agent-app/
├── api_keys/               [770, agent-admin:agent-core]   # 보안 디렉터리 (Secret Keys)
│   ├── secret.key          [660, agent-admin:agent-core]   # API 비밀키 파일
│   └── t_secret.key        [660, agent-admin:agent-core]   # 테스트 비밀키 파일
├── upload_files/           [775, agent-admin:agent-common] # 일반 공유 디렉터리
├── bin/                    [750, agent-admin:agent-core]   # 관제 스크립트 저장소
│   └── monitor.sh          [750, agent-dev:agent-core]     # 시스템 관제 자동화 스크립트
└── agent-app-linux-arm64   [755, agent-admin:agent-common] # 애플리케이션 실행 파일

/var/log/agent-app/         [770, agent-admin:agent-core]   # 시스템 관제 로그 저장소
└── monitor.log             [660, agent-admin:agent-core]   # 관제 데이터 누적 로그 파일
```
# 실행과정
가상머신을 설치하여 진행하였다. 
```
brew install --cask multipass
```
가상머신 버젼은 아래와 같다.
```
multipass   1.16.4+mac
multipassd  1.16.4+mac
```

우분투 22.04 LTS 최신 이미지로 가상머신을 생성 및 실행+생성할 가상머신의 이름을 agent-server+CPU 2코어, RAM 2GB, 용량 10GB의 독립된 리눅스 자원을 할당
```
multipass launch 22.04 --name agent-server --cpus 2 --memory 2G --disk 10G
```

현재 동작 중인 가상머신 목록과 IP, 상태(Running)를 확인
```
 ~ % multipass list
Name                    State             IPv4             Image
agent-server            Running           192.168.252.2    Ubuntu 22.04 LTS
```

가상머신 내부 접속
```
multipass shell agent-server
```

* 성공시 ubuntu@agent-server:~$  이렇게 뜬다.

SSH 설정 파일 편집
sudo nano /etc/ssh/sshd_config
sudo: SuperUser DO의 약자로, 관리자(root) 권한으로 명령을 실행합니다. (보안 설정 수정 시 필수)
nano: 터미널 기반 텍스트 편집기입니다.
/etc/ssh/sshd_config: SSH 서버 데몬의 보안 및 접속 규칙이 담긴 핵심 설정 파일 경로입니다.

#Port 22 를 Port 20022 로 수정
#PermitRootLogin prohibit-password 를 PermitRootLogin no 수정 후 SSH 설정 파일을 저장

방금 변경한 포트(20022)와 Root 차단 설정을 SSH 데몬에 실제로 적용하기 위해 서비스를 재시작
sudo systemctl restart ssh

SSH 서비스가 변경된 포트(20022)에서 정상적으로 요청을 대기(LISTEN)하고 있는지 네트워크 상태를 검증
ss -tulnp | grep sshd

* 위의 결과가 출력이 안되는 문제가 발생
Ubuntu 22.04 환경에서는 SSH 서비스 이름이 ssh가 아닌 sshd로 등록되어 있거나, 포트 변경 후 서비스가 제대로 시작되지 않았을 수 있다고 함
sudo systemctl status ssh 를 사용하여, SSH 서비스의 현재 동작 상태(Active: active (running) 인지 확인)와 에러 로그를 점검
Active: active (running) 를 통해 SSH 서비스가 에러 없이 정상적으로 구동 중임을 알수 있다. 
sshd[1944]: Server listening on 0.0.0.0 port 20022. 를 통해 포트 변경이 20022가 성공했읍을 알수 있다. 
프로세스 이름인 ssh 대신 변경된 포트 번호 20022가 정상동작이 확인 되었기 때문에 ss -tulnp | grep 20022 로 하면 정상 출력된 결과를 확인 할수 있음
결과는 아래와 같음
tcp LISTEN 0 128 0.0.0.0:20022 0.0.0.0:* 이것은 IPv4 기반이다. 
tcp LISTEN 0 128 [::]:20022 [::]:* 이것은 IPv6 기반이다. 
모두 20022 포트가 (LISTEN 이 적혀있으므로) 접속 대기중임을 나타낸다. 

1. 1. Root 원격 접속 차단 설정 확인
grep -i "PermitRootLogin" /etc/ssh/sshd_config

__________________________________________________________

SSH 통신용 TCP 20022번 포트 진입을 허용
sudo ufw allow 20022/tcp

애플리케이션 서비스용 TCP 15034번 포트 진입을 허용
sudo ufw allow 15034/tcp

UFW 방화벽을 활성화
sudo ufw enable

방화벽 동작 상태(Active)와 개방된 포트 목록의 세부 내역을 출력
sudo ufw status verbose
위를 치면 아래와 같은 출력이 나온다. 
Status: active =>UFW 방화벽이 정상적으로 켜져 작동 중
Default: deny (incoming), allow (outgoing), disabled (routed) => 허용 규칙에 명시되지 않은 모든 외부 들어오는 접속(Incoming)은 기본적으로 차단
20022/tcp      ALLOW IN    Anywhere =>SSH개방                 
15034/tcp      ALLOW IN    Anywhere =>APP개방

* 기본 보안 및 네트워크 설정 완료

관리자(sudo) 권한으로 agent-common이라는 새로운 사용자 그룹을 시스템에 생성, 성공시에, 관리자(sudo) 권한으로 agent-core라는 두 번째 그룹을 시스템에 생성
sudo groupadd agent-common && sudo groupadd agent-core

관리자 계정으로 agent-admin 을 생성(기본 그룹: agent-common, 보조 그룹: agent-core, 모든 공용 디렉토리와 보안 디렉토리에 접근 가능하다. 예를 들어서 pi_keys, /var/log/agent-app)
개발자 계정으로 agent-dev 을 생성(기본 그룹: agent-common, 보조 그룹: agent-core, 관제 스크립트(monitor.sh) 작성자로서 보안 파일 및 시스템 로그에 접근 가능)
QA/테스트 계정으로 agent-test 생성 (기본 그룹: agent-common, 보조그룹 없음, 일반 공유 디렉토리(upload_files)에는 접근할 수 있지만, 보안 그룹인 agent-core에는 속하지 않아 Secret Key나 시스템 로그(api_keys, /var/log/agent-app)에는 접근이 차단 == 최소 권한 원칙 적용)
sudo useradd -m -g agent-common -G agent-core agent-admin && sudo useradd -m -g agent-common -G agent-core agent-dev && sudo useradd -m -g agent-common agent-test

* 최소 권한 원칙 : 사용자, 프로그램, 또는 시스템 프로세스에게 업무를 수행하는 데 필요한 '최소한의 권한'만 부여해야 한다는 보안 기본 원칙

agent-admin와 agent-dev와 agent-test를 별 번호(UID), 기본 그룹(GID), 그리고 소속된 모든 보조 그룹 목록을 화면에 출력
id agent-admin && id agent-dev && id agent-test 
결과는 아래와 같다. 
uid=1001(agent-admin) gid=1001(agent-common) groups=1001(agent-common),1002(agent-core) =>agent-common의 정보는 다음과 같다. 기본그룹은 agent-common이다. 보조그룹은 agent-common과 agent-core 이 있다. 
uid=1002(agent-dev) gid=1001(agent-common) groups=1001(agent-common),1002(agent-core)
uid=1003(agent-test) gid=1001(agent-common) groups=1001(agent-common)

* 마지막에서 agent-core이 보조그룹에서 제외된것은 최소 권한 원칙 때문임

디렉토리 생성 : 앱이 사용할 업로드 폴더, 보안 키 폴더, 스크립트 실행 폴더, 그리고 시스템 로그 저장용 /var/log/agent-app 폴더를 한 번에 생성 (-p 는 부모 디렉토리가 없으면 자동으로 함께 생성, {}를 이용하여 폴더 3개를 한번에 생성)
sudo mkdir -p /home/agent-admin/agent-app/{upload_files,api_keys,bin} && sudo mkdir -p /var/log/agent-app

디렉토리 소유권 설정 명령어 : 공용 작업 폴더는 일반 그룹(agent-common)이 다룰 수 있게 하고, 로그 폴더는 보안 그룹(agent-core)만 다룰 수 있도록 주인과 그룹을 구분 (-R 는 해당 디렉토리 내부의 모든 하위 파일/폴더에도 소유권을 동일하게 적용, agent-admin:agent-common: 소유자는 agent-admin, 소유 그룹은 agent-common으로 지정)
sudo chown -R agent-admin:agent-common /home/agent-admin/agent-app && sudo chown -R agent-admin:agent-core /var/log/agent-app

* chown 은 소유자와 소유그룹을 변경하는 명령어이다. 현재 sudo mkdir로 디렉토리를 만들었기때문에 소유자와 소유그룹은 모두 root 이다. 이를 소유자는 agent-admin로, 소유 그룹은 agent-common 로 설정한것이. 앱 운영 관리자인 agent-admin 계정과 기본 그룹인 agent-common 사용자들이 해당 애플리케이션 디렉토리에 접근하여 작업하게끔 하기위해서 이다.
sudo chown [소유자]:[그룹] [대상경로] 

각 그룹에 접근 권한을 설정하였다. (소유자,그룹, 기타 // 읽기,쓰기,실행)
sudo chmod -R 775 /home/agent-admin/agent-app/upload_files && sudo chmod -R 770 /home/agent-admin/agent-app/api_keys && sudo chmod -R 770 /var/log/agent-app

디렉터리 3곳의 소유권과 접근 권한(읽기/쓰기/실행) 상태를 확인
sudo ls -ld /home/agent-admin/agent-app/upload_files /home/agent-admin/agent-app/api_keys /var/log/agent-app 를 이용한 결과

* 
drwxrwx--- 2 agent-admin agent-common 4096 Sep 18 18:30 /home/agent-admin/agent-app/api_keys=> 소유 그룹이 보안 전용 그룹인 agent-core가 아닌 agent-common으로 잘못 지정되어 있음. 이렇게 되면 보안 그룹이 아닌 agent-test 계정도 키 파일에 접근할 수 있게 됨. 따라서 소유그룹을 agent-core로 변경해야 함 
drwxrwxr-x 2 agent-admin agent-common 4096 Sep 18 18:30 /home/agent-admin/agent-app/upload_files =>소유 그룹 agent-common 및 권한(775 = rwxrwxr-x)이 요구사항대로 완벽하게 설정됨
drwxrwx--- 2 agent-admin agent-core   4096 Sep 18 18:30 /var/log/agent-app =>소유 그룹 agent-core 및 보안 권한(770 = rwxrwx---)이 요구사항대로 완벽하게 적용됨

* 해당 문제를 고치기 위해 
sudo chown -R agent-admin:agent-core /home/agent-admin/agent-app/api_keys 를 사용한 결과 agent-admin agent-core 로 해당 부분이 수정됨을 확인하였음

________________________________________________________

키파일 생성
echo "agent_api_key_test" | sudo tee /home/agent-admin/agent-app/api_keys/t_secret.key > /dev/null =>sudo tee [파일명]: 전달받은 텍스트를 관리자(root) 권한으로 파일에 작성, > /dev/null은 "파일 저장은 하되, 터미널 화면에는 아무것도 출력하지 마라"라는 의미

키파일의 소유자/그룹 정의 및 권한 정의
sudo chown agent-admin:agent-core /home/agent-admin/agent-app/api_keys/t_secret.key && sudo chmod 660 /home/agent-admin/agent-app/api_keys/t_secret.key => 660 인 이유는 외부자가 사용 못하게 하려고 한 것

새 터미널을 열어서, 파이썬 서버를 열음
cd /Users/byeong/Downloads && python3 -m http.server 8000

가상머신에서 wget으로 다운
sudo wget http://192.168.252.1:8000/agent-app.zip -O /home/agent-admin/agent-app/agent-app.zip

agent-app.zip 파일의 압축을 풀어서 /home/agent-admin/agent-app/ 경로에 덮어씌워 추출 && 압축 해제되어 생성된 agent-app-linux-arm64 파일에 실행 권한(+x)을 부여하여 프로그램으로 작동할 수 있게 만듬
sudo unzip -o /home/agent-admin/agent-app/agent-app.zip -d /home/agent-admin/agent-app/ && sudo chmod +x /home/agent-admin/agent-app/agent-app-linux-arm64 =>+x: 해당 파일에 '실행 권한'을 추가, -o: 압축을 풀 때 기존에 같은 이름의 파일이 존재하더라도 묻지 않고 덮어씌움, 

agent-app 디렉터리와 그 하위에 있는 모든 파일/폴더의 소유자를 agent-admin, 소유 그룹을 agent-common으로 일괄 변경 && api_keys 디렉터리와 그 내부 파일들만 소유 그룹을 agent-core로 다시 변경
sudo chown -R agent-admin:agent-common /home/agent-admin/agent-app && sudo chown -R agent-admin:agent-core /home/agent-admin/agent-app/api_keys

압축해제 검증
sudo ls -la /home/agent-admin/agent-app/

* 지금까지 일반 계정(agent-admin)으로 실행(루트 실행 금지)"과 "보안 정책 준수"를 만족하기 위해서 파일 압축 해제후 권한을 설정했음

키파일 생성 & 660 권한 설정
echo "agent_api_key_test" | sudo tee /home/agent-admin/agent-app/api_keys/t_secret.key > /dev/null && sudo chown agent-admin:agent-core /home/agent-admin/agent-app/api_keys/t_secret.key && sudo chmod 660 /home/agent-admin/agent-app/api_keys/t_secret.key => echo "" > 로 하면 일반 사용자 권한으로 실행됨. 앞서 디렉토리 권한을 770 으로 설정해서 sudo 권한을 쓰는 tee 를 사용함. 

agent-admin 일반 계정으로 전환
sudo -u agent-admin -i

환경변수 적용
export AGENT_HOME=/home/agent-admin/agent-app
export AGENT_PORT=15034
export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
export AGENT_KEY_PATH=$AGENT_HOME/api_keys/t_secret.key
export AGENT_LOG_DIR=/var/log/agent-app =>export를 쓴 이유는 다음과 같다. 파일 수정 없이 바로 적용되며, 이번 미션처럼 "앱을 한 번 구동해서 Boot Sequence [OK]를 검증"하는 일회성/테스트 목적에 가장 빠르고 안전함. 현재 열려있는 터미널 세션의 메모리에만 일시적으로 변수를 올리기 때문에 종료후 설정값이 사라짐. 반면 파일로 하면 계속 유지됨.(.bashrc 또는 .env 등 nano 편집기로 작성)

앱 실핼
AGENT_HOME/agent-app-linux-arm64

* 에러발생 : Key Path Mismatch. Expected: /home/agent-admin/agent-app/api_keys => AGENT_KEY_PATH 환경 변수에 키 파일의 전체 경로(파일명 포함)가 지정되어 있기 때문임. AGENT_KEY_PATH 환경 변수 재설정하여 해결. export AGENT_KEY_PATH=/home/agent-admin/agent-app/api_keys

* 에러발생 : [3/5] Checking Required Files [FAIL]
>>> Missing File: secret.key
>>> (Expected location: /home/agent-admin/agent-app/api_keys/secret.key)
애플리케이션이 요구하는 비밀키 파일 이름이 t_secret.key가 아니라 secret.key이기 때문임. 기존에 만들어둔 t_secret.key 파일을 앱이 찾는 secret.key 이름으로 복사하거나 새로 생성해 주면 됨. cp /home/agent-admin/agent-app/api_keys/t_secret.key /home/agent-admin/agent-app/api_keys/secret.key 

해결 완료
All Boot Checks Passed!
Agent READY 
가 출려됨.

_____________________________________

monitor.sh 스크립트 작성
sudo nano /home/agent-admin/agent-app/bin/monitor.sh

스크립트 내용
#!/bin/bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
LOG_DIR="/var/log/agent-app"
LOG_FILE="$LOG_DIR/monitor.log"
mkdir -p "$LOG_DIR"

if ! pgrep -f "agent-app-linux-arm64" > /dev/null && ! pgrep -f "agent_app.py" > /dev/null; then
    echo "[CRITICAL] Process agent_app is not running." >> "$LOG_FILE"
    exit 1
fi

if ! ss -tuln | grep -q ":15034 "; then
    echo "[CRITICAL] Port 15034 is not listening." >> "$LOG_FILE"
    exit 1
fi

if command -v ufw > /dev/null 2>&1; then
    if ! ufw status 2>/dev/null | grep -q "Status: active"; then
        echo "[WARNING] UFW Firewall is inactive." >> "$LOG_FILE"
    fi
fi

PID=$(pgrep -f "agent-app-linux-arm64" | head -n 1)
[ -z "$PID" ] && PID=$(pgrep -f "agent_app.py" | head -n 1)

CPU=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')
CPU_INT=$(printf "%.0f" "$CPU")

MEM=$(free | awk '/Mem:/ {print $3/$2 * 100.0}')
MEM_INT=$(printf "%.0f" "$MEM")

DISK_USED=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')

[ "$CPU_INT" -gt 20 ] && echo "[WARNING] CPU usage is over 20%: ${CPU_INT}%" >> "$LOG_FILE"
[ "$MEM_INT" -gt 10 ] && echo "[WARNING] MEM usage is over 10%: ${MEM_INT}%" >> "$LOG_FILE"
[ "$DISK_USED" -gt 80 ] && echo "[WARNING] DISK usage is over 80%: ${DISK_USED}%" >> "$LOG_FILE"

TIMESTAMP=$(date "+%Y-%m-%d HH:%M:%S")
echo "[$TIMESTAMP] PID:$PID CPU:${CPU_INT}% MEM:${MEM_INT}% DISK_USED:${DISK_USED}%" >> "$LOG_FILE"

MAX_SIZE=$((10 * 1024 * 1024))
if [ -f "$LOG_FILE" ]; then
    FILE_SIZE=$(stat -c%s "$LOG_FILE" 2>/dev/null || echo 0)
    if [ "$FILE_SIZE" -ge "$MAX_SIZE" ]; then
        mv "$LOG_FILE" "$LOG_FILE.$(date +%Y%m%d%H%M%S)"
        touch "$LOG_FILE"
        chmod 660 "$LOG_FILE"
        ls -t $LOG_DIR/monitor.log.* 2>/dev/null | tail -n +10 | xargs -r rm -f
    fi
fi

권한설정 및 crontab 등록
sudo chown agent-dev:agent-core /home/agent-admin/agent-app/bin/monitor.sh
sudo chmod 750 /home/agent-admin/agent-app/bin/monitor.sh
echo "* * * * * /home/agent-admin/agent-app/bin/monitor.sh >/dev/null 2>&1" | sudo crontab -u agent-admin -l

자동 누적 확인
sudo tail -n 5 /var/log/agent-app/monitor.log

결과
[WARNING] MEM usage is over 10%: 16%
[2026-09-19 HH:14:01] PID:2854 CPU:2% MEM:16% DISK_USED:20%
[WARNING] UFW Firewall is inactive.
[WARNING] MEM usage is over 10%: 17%
[2026-09-19 HH:15:02] PID:2854 CPU:4% MEM:17% DISK_USED:20% 


# 터미널 입력만 모음
# SSH 포트 변경(20022) 및 Root 로그인 차단
sudo sed -i 's/^#*Port 22/Port 20022/' /etc/ssh/sshd_config
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart ssh

# UFW 방화벽 포트 개방 및 활성화
sudo ufw allow 20022/tcp
sudo ufw allow 15034/tcp
sudo ufw --force enable

# 그룹 및 계정 생성
sudo groupadd agent-common
sudo groupadd agent-core
sudo useradd -m -g agent-common -G agent-core agent-admin
sudo useradd -m -g agent-common -G agent-core agent-dev
sudo useradd -m -g agent-common agent-test

# 디렉터리 생성
sudo mkdir -p /home/agent-admin/agent-app/{upload_files,api_keys,bin}
sudo mkdir -p /var/log/agent-app

# 소유권 지정
sudo chown -R agent-admin:agent-common /home/agent-admin/agent-app
sudo chown -R agent-admin:agent-core /home/agent-admin/agent-app/api_keys
sudo chown -R agent-admin:agent-core /var/log/agent-app

# 권한 지정 (775 / 770)
sudo chmod -R 775 /home/agent-admin/agent-app/upload_files
sudo chmod -R 770 /home/agent-admin/agent-app/api_keys
sudo chmod -R 770 /var/log/agent-app

# 압축 해제 유틸리티 설치
sudo apt update && sudo apt install -y unzip

# 비밀키 파일 생성 및 660 권한 부여
echo "agent_api_key_test" | sudo tee /home/agent-admin/agent-app/api_keys/secret.key > /dev/null
echo "agent_api_key_test" | sudo tee /home/agent-admin/agent-app/api_keys/t_secret.key > /dev/null
sudo chown -R agent-admin:agent-core /home/agent-admin/agent-app/api_keys
sudo chmod 660 /home/agent-admin/agent-app/api_keys/*.key

# 앱 압축 해제 및 실행 권한 부여
sudo unzip -o /home/agent-admin/agent-app/agent-app.zip -d /home/agent-admin/agent-app/
sudo chmod +x /home/agent-admin/agent-app/agent-app-linux-arm64
sudo chown -R agent-admin:agent-common /home/agent-admin/agent-app

# 백그라운드로 애플리케이션 구동
sudo -u agent-admin bash -c '
export AGENT_HOME=/home/agent-admin/agent-app
export AGENT_PORT=15034
export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
export AGENT_KEY_PATH=$AGENT_HOME/api_keys
export AGENT_LOG_DIR=/var/log/agent-app

nohup $AGENT_HOME/agent-app-linux-arm64 > /dev/null 2>&1 &
'

# 스크립트 작성(생략)

# agent-admin crontab 매분 등록
echo "* * * * * /home/agent-admin/agent-app/bin/monitor.sh >/dev/null 2>&1" | sudo crontab -u agent-admin -

# 등록 결과 확인
sudo crontab -u agent-admin -l

# 1분 뒤 누적 로그 확인
sudo tail -n 5 /var/log/agent-app/monitor.log
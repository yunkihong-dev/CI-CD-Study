# 단일 서버에서 하이브리드 클라우드까지: CI/CD 파이프라인 구축

단일 서버(All-in-One) 환경에서 시작해, 서버 분리 → Kubernetes → Cloud로 점진적으로 고도화해나가는 CI/CD 구축 과정을 다루었습니다.

---

## 👥팀원
<table>
  <tr>
    <td align="center">
      <a href="https://github.com/Jsumin07">
        <img src="https://github.com/Jsumin07.png" width="100px;" alt="전수민"/><br />
        <sub><b>전수민</b></sub>
      </a>
    </td>
    <td align="center">
      <a href="https://github.com/0-zoo">
        <img src="https://github.com/0-zoo.png" width="100px;" alt="이영주"/><br />
        <sub><b>이영주</b></sub>
      </a>
    </td>
    <td align="center">
      <a href="https://github.com/yunkihong-dev">
        <img src="https://github.com/yunkihong-dev.png" width="100px;" alt="홍윤기"/><br />
        <sub><b>홍윤기</b></sub>
      </a>
    </td>
  </tr>
</table>


---

## 1. 개요

### 목적

- CI/CD 파이프라인을 직접 구축하고, **단계별로 확장**
- 코드 푸시부터 배포까지 전 과정 자동화 → 개발자는 코드 작성에만 집중 가능

### 범위

- **1단계**: 단일 서버 (Jenkins + WAS 통합)
- **2단계**: 서버 분리 (CI 서버 ↔ WAS 서버)
- **차후**: Kubernetes / Cloud 확장

---

## 2. 사용 환경 및 기술 스택

| 구분 | 기술 | 설명 |
| --- | --- | --- |
| **CI/CD** | Jenkins (Docker) | 업계 표준 CI 서버, GitHub Webhook 연동 |
| **SCM** | GitHub + ngrok | ngrok으로 로컬 Jenkins에 GitHub Webhook 전달 |
| **자동화** | Shell Script | JAR 변경 감지 및 자동 실행, 원격 배포 자동화 |
| **빌드 도구** | Gradle / Maven | Spring Boot 프로젝트 빌드 및 JAR 산출 |
| **원격 배포** | scp / ssh | 빌드 서버 → WAS 서버 JAR 파일 복사 및 실행 제어 |
| **런타임** | OpenJDK 17 | Spring Boot 3.x 호환 안정 버전 |

---

## 3. 1단계: 단일 서버 (All-in-One)

### 3.1 단계 배경과 목표

- **Jenkins와 WAS(Spring Boot 앱)를 하나의 서버에서 운영**
- Jenkins가 **빌드와 실행을 모두 담당**
- 빠르게 CI/CD 전체 사이클을 실습할 수 있도록 구성
- 단, 빌드와 실행이 한 서버에서 동시에 일어나므로 빌드 오류 시 서비스 다운 가능성이 있음

---

### 3.2 아키텍처 구조

<img width="1261" height="743" alt="아키텍처" src="https://github.com/user-attachments/assets/bf8b3b40-0a29-4353-b827-b83eb801fd14" />

- Jenkins: 빌드 + 실행
- appjardir: 빌드된 JAR 저장 경로
- auto_deploy.sh: 변경 감지 후 자동 실행

---

### 3.3 동작 흐름

1. 개발자가 GitHub에 push
2. GitHub Webhook → ngrok → Jenkins로 전달
3. Jenkins가 Gradle/Maven 빌드 실행 → JAR 산출
4. `auto_deploy.sh`가 `app_latest.jar` 변경을 감지
5. 기존 앱 종료 → 새 JAR 실행

---

### 3.4 주요 스크립트 (`auto_deploy.sh`)

```bash
#!/bin/bash
cd /home/user/appjardir
echo "자동 배포 서비스 시작..."
LAST_CHECK=0
CURRENT_JAR=""

kill_app() {
    echo "기존 앱 완전 종료 시도..."
    # 포트 기반 종료
    PID_PORT=$(lsof -t -i:8080 2>/dev/null)
    if [ ! -z "$PID_PORT" ]; then
        echo "포트 8080 사용 중인 PID: $PID_PORT 종료"
        kill $PID_PORT
        sleep 2
        kill -9 $PID_PORT 2>/dev/null
    fi
    # JAR 이름으로 종료
    if [ ! -z "$CURRENT_JAR" ]; then
        pkill -f "java.*$CURRENT_JAR"
    fi
    echo "✅ 종료 완료"
}

start_app() {
    echo "새 앱 시작 중..."
    if [ -L app_latest.jar ]; then
        CURRENT_JAR=$(readlink app_latest.jar)
    else
        CURRENT_JAR="app_latest.jar"
    fi
    java -jar app_latest.jar > app.log 2>&1 &
    echo "✅ 앱 시작됨 (JAR: $CURRENT_JAR)"
}

while true; do
    if [ -f app_latest.jar ]; then
        CURRENT_TIME=$(stat -c %Y app_latest.jar)
        if [ $CURRENT_TIME -gt $LAST_CHECK ]; then
            echo "새로운 JAR 감지됨!"
            kill_app
            sleep 3
            start_app
            LAST_CHECK=$CURRENT_TIME
        fi
    fi
    sleep 5
done
```

---

### 3.5 실행 방법

![ezgif-8f959a8261ea3f](https://github.com/user-attachments/assets/0a319205-7838-4f0c-9651-8f43f7c2f665)

### (1) Jenkins 컨테이너 실행

```bash
# Jenkins 컨테이너 시작 (JDK 17 포함)
docker run --name myjenkins2 -p 8888:8080 \
  -v $(pwd)/appjardir:/var/jenkins_home/appjar \
  jenkins/jenkins:lts-jdk17
```

- `p 8888:8080`: 호스트 8888 포트를 Jenkins 웹 UI와 매핑
- `v $(pwd)/appjardir:/var/jenkins_home/appjar`: 빌드 결과물을 호스트에서도 바로 확인 가능

---

### (2) 초기 설정

```bash
# Jenkins 초기 비밀번호 확인
docker logs myjenkins2

# 브라우저에서 접속
http://localhost:8888
```

### Webhook + ngrok 연결

```bash
ngrok http 8888
```

- GitHub → ngrok 주소로 Webhook 설정 → Jenkins 자동 빌드 가능

### 배포 확인

```bash
cd appjardir
java -jar app_latest.jar
```

---

### (3) 파이프라인 생성

1. Jenkins 대시보드 → **New Item → Pipeline**
2. 아래 Jenkinsfile 입력
3. **Save → Build Now**

👉 Jenkinsfile 관리 → 버전 추적 & 이식성 확보

---

### (4) Jenkins Pipeline 설정

```groovy
pipeline {
  agent any
  options {
    timestamps()
  }
  environment {
    DEST_DIR = '/var/jenkins_home/appjar'
    TZ       = 'Asia/Seoul'
    JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'
    PATH = "${JAVA_HOME}/bin:${PATH}"
  }
  stages {
    stage('Checkout') {
      steps {
        echo 'Git 저장소에서 코드 가져오기'
        git branch: 'main', url: '<https://github.com/yunkihong-dev/CI-CD-Study.git>'
      }
    }
    stage('Build with Gradle') {
      steps {
        echo 'Gradle 빌드 시작'
        dir('step04_gradleBuild') {
          sh '''#!/bin/bash
            set -euo pipefail
            chmod +x ./gradlew
            ./gradlew clean build -x test
          '''
        }
      }
    }
    stage('Copy Jar to Build Directory') {
      steps {
        echo '빌드된 JAR을 날짜별 파일명으로 DEST_DIR에 저장'
        sh '''#!/bin/bash
          set -euo pipefail

          ts="$(date +%Y%m%d_%H%M%S)"
          mkdir -p "${DEST_DIR}"

          jar="$(ls -1 step04_gradleBuild/build/libs/*.jar \
                 | grep -vE '(-plain|sources|javadoc)\\.jar$' \
                 | head -n 1 || true)"

          if [ -z "${jar}" ]; then
            echo "❌ JAR 파일을 찾을 수 없습니다."
            exit 1
          fi

          cp "${jar}" "${DEST_DIR}/app_${ts}.jar"
          chmod 644 "${DEST_DIR}/app_${ts}.jar"

          ln -sfn "app_${ts}.jar" "${DEST_DIR}/app_latest.jar"

          ls -la "${DEST_DIR}/"
        '''
      }
    }
    stage('Archive Artifacts') {
      steps {
        archiveArtifacts artifacts: 'step04_gradleBuild/build/libs/*.jar',
                         fingerprint: true,
                         onlyIfSuccessful: true
      }
    }
  }
  post {
    success {
      echo '✅ Build & copy complete.'
    }
    failure { echo '❌ Build failed — 로그 확인 필요.' }
  }
}

```

---

### (5) 프로젝트 구조

```
appjardir
├── app_20250916_170006.jar
├── app_latest.jar -> app_20250916_170006.jar
├── app.log
├── auto_deploy.sh
└── deploy.log

```

- `app_latest.jar`: 최신 빌드
- `app_YYYYMMDD_HHMMSS.jar`: 버전 관리 가능
- `auto_deploy.sh`: 자동 배포 스크립트
- `deploy.log`: 배포 이력 로그

---

✅ 여기까지가 **1단계 단일 서버 (All-in-One)** 전체 내용입니다.

2단계부터는 Jenkins와 WAS 서버를 분리해 더 안전하고 현실적인 구조로 발전시킵니다.

---

## 4. 2단계: 서버 분리 (CI 서버 + WAS 서버)

### 4.1 단계 배경과 목표

- **빌드 서버(Jenkins)** 와 **운영 서버(WAS)** 분리
- CI/CD 안정성과 보안성 확보
- Jenkins는 빌드만, WAS는 실행만 담당

---

### 4.2 아키텍처 구조

<img width="1773" height="786" alt="제목 없는 다이어그램 drawio" src="https://github.com/user-attachments/assets/8685a715-d7fb-4a5f-905a-2749440b6cea" />


- Jenkins 서버: 빌드 + JAR 전송
- WAS 서버: JAR 실행 + 서비스 운영


---

### 4.3 동작 흐름

1. 개발자가 GitHub에 push
2. Webhook → Jenkins 서버 트리거
3. Jenkins → 빌드 → JAR 생성
4. `scp` → JAR을 WAS 서버로 전송
5. Jenkins → `ssh` → WAS 서버의 `deploy.sh` 실행
6. WAS 서버: 기존 앱 종료 → 새 JAR 실행

---

### 4.4 자동화 스크립트

### Jenkins 서버 (`myserver01 /appjardir/auto_deploy.sh`)

```bash
# myserver01 /appjardir/auto_deploy.sh
#!/bin/bash
WATCH_DIR="/home/ubuntu/appjardir"
REMOTE_USER="ubuntu"
REMOTE_HOST="myserver02"
REMOTE_DIR="/home/ubuntu/appjardir"
echo ":로켓: 서버01: 최신 JAR 전송 시작..."
LAST_MOD=0
while true; do
    # 최신 JAR 찾기 (실제 파일)
    LATEST_JAR=$(find "$WATCH_DIR" -maxdepth 1 -type f -name "*.jar" -printf "%T@ %p\n" | sort -n | tail -1 | awk '{print $2}')
    if [ -n "$LATEST_JAR" ]; then
        MOD_TIME=$(stat -c %Y "$LATEST_JAR")
        if [ $MOD_TIME -gt $LAST_MOD ]; then
            echo ":로켓: 새로운 JAR 감지: $LATEST_JAR"
            # 실제 파일 전송
            scp "$LATEST_JAR" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/"
            echo ":흰색_확인_표시: 원격 서버로 전송 완료"
            LAST_MOD=$MOD_TIME
        fi
    fi
    sleep 5
done

```

### WAS 서버 (`myserver02 /appjardir/auto_deploy.sh`)

```bash
# myserver02 /appjardir/auto_deploy.sh
#!/bin/bash
APP_DIR="/home/ubuntu/appjardir"
echo ":로켓: 서버02: JAR 실행 감시 시작..."
LAST_MOD=0
while true; do
    # 가장 최근에 수정된 JAR 찾기 (실제 파일)
    NEW_JAR=$(find "$APP_DIR" -maxdepth 1 -type f -name "*.jar" -printf "%T@ %p\n" | sort -n | tail -1 | awk '{print $2}')
    if [ -n "$NEW_JAR" ]; then
        MOD_TIME=$(stat -c %Y "$NEW_JAR")
        if [ $MOD_TIME -gt $LAST_MOD ]; then
            BASENAME=$(basename "$NEW_JAR")
            echo ":로켓: 새로운 JAR 감지: $BASENAME"
            # 기존 앱 종료
            pkill -f java 2>/dev/null
            # 새 앱 실행
            nohup /usr/bin/java -jar "$NEW_JAR" > "$APP_DIR/app.log" 2>&1 </dev/null &
            echo ":흰색_확인_표시: 앱 실행 완료: $BASENAME"
            LAST_MOD=$MOD_TIME
        fi
    fi
    sleep 5
done
```

---

### 4.5 실행 방법

### Jenkins 서버 환경 구성

- Docker 기반 Jenkins 설치
- `scp`, `ssh` 명령어 사용 가능하도록 세팅

### WAS 서버 환경 구성

- OpenJDK 17 설치
- `/home/user/appjardir` 디렉토리 생성

### SSH 키 교환

```bash
ssh-keygen -t rsa
ssh-copy-id user@myserver02
```

### 배포 테스트

```bash
./jenkins_copy.sh step10-CICD/build/libs/app.jar
```

---

## 5. 1단계 vs 2단계 비교

| 구분 | 1단계 (단일 서버) | 2단계 (서버 분리) |
| --- | --- | --- |
| **구조** | Jenkins + WAS 통합 | Jenkins(CI) + WAS(CD) 분리 |
| **장점** | 단순, 빠른 학습 | 안정성, 보안성, 운영 유사 |
| **단점** | 빌드 오류 시 서비스 영향 | 설정 복잡도 증가 |

---

## 6. 핵심 포인트

- **Webhook + ngrok**: 로컬 환경에서도 CI 자동화 테스트 가능
- **Shell Script 제어**: `auto_deploy.sh`, `deploy.sh`로 서비스 재시작 자동화
- **scp + ssh**: 원격 서버 배포 자동화의 핵심 도구
- **CI vs CD 분리**: Jenkins는 빌드, WAS는 실행 → DevOps 구조

---

## 🛠️ 트러블슈팅

| 문제 상황 | 원인 | 해결 방법 |
| --- | --- | --- |
| **권한 오류 (`/opt/builds` 접근 불가)** | Jenkins 컨테이너 내부의 기본 사용자(UID 1000)가 호스트 디렉토리에 접근 권한이 없음 | Jenkins 홈 디렉토리(`/var/jenkins_home/appjar`)를 볼륨으로 마운트하거나, 호스트 디렉토리 소유자를 UID 1000으로 맞춰줌 |
| **Java 버전 문제** | Jenkins 컨테이너에 설치된 JDK 버전과 프로젝트의 요구 버전(Spring Boot 3.x → JDK 17 이상)이 불일치 | 환경 변수 `JAVA_HOME`을 `/usr/lib/jvm/java-17-openjdk-amd64`로 지정하고, PATH에도 반영해 빌드 시 올바른 JDK를 사용하도록 설정 |
| **Gradle 실행 권한 오류 (`gradlew: Permission denied`)** | 프로젝트에 포함된 `gradlew` 파일이 실행 권한을 갖고 있지 않음 | Jenkins 빌드 스텝에서 `chmod +x ./gradlew` 명령어를 추가하여 실행 권한을 부여 |
| **Bind Mount 경로 접근 실패** | 컨테이너와 호스트 간 UID/GID 불일치 또는 호스트 디렉토리 권한 제한 | 호스트 디렉토리 권한을 UID 1000 (Jenkins 기본 계정)으로 변경하거나 `chmod 755` 이상으로 퍼미션을 수정 |
| **잘못된 배포 확장 시도 (Docker Run 직접 실행)** | Jenkins 파이프라인에 `docker run` 스테이지를 넣어 컨테이너로 바로 실행하는 방법을 테스트했으나, 운영 환경에서는 JAR 교체·롤백 관리가 어렵고 컨테이너 충돌 문제가 발생 | 단일 서버 환경에서는 `auto_deploy.sh` 스크립트를 통한 JAR 교체 방식이 더 안정적이고 단순 |

---

## 7. 확장 계획(다음 단계)

- **3단계**: Minikube 기반 Kubernetes 클러스터 구성
    - Jenkins → Kubernetes Deployment 자동화
    - Pod Rolling Update, Service LoadBalancer 적용
- **4단계**: Cloud & Hybrid Cloud 배포
    - AWS/GCP 환경에 CI/CD 적용
    - 멀티 환경(dev/stage/prod) 관리 및 모니터링

---



# 🚀 AI 모의 면접관 프로젝트: AWS & Jenkins CI/CD 구축 학습 노트

## 1. 프로젝트 개요
* **프로젝트명**: AI Interviewer (가칭: PreView)
* **목표**: 프론트엔드(React)와 백엔드(Spring Boot)를 AWS에 배포하고, Jenkins를 통해 자동 배포(CI/CD) 파이프라인을 구축한다.
* **최종 아키텍처**:
    * **Frontend**: AWS S3 (정적 호스팅) + CloudFront (HTTPS/CDN)
    * **Backend**: AWS EC2 (Docker Container)
    * **Deployment**: Jenkins (Docker Container)
    * **Domain**: `previewai.store` (Route 53)

---

## 2. 서버 환경 구축 (AWS EC2)

### 2-1. 인스턴스 생성
* **OS**: Amazon Linux 2023 AMI
* **Type**: `t3.small` (초기 `t2.micro`에서 메모리 부족으로 업그레이드)
* **Storage**: 30GB (gp3)
* **Security Group (포트 오픈)**:
    * `22` (SSH), `80/443` (HTTP/HTTPS), `8080` (Backend), `9000` (Portainer), `9090` (Jenkins)

### 2-2. 필수 소프트웨어 설치 (Magic Script)
도커 설치, 권한 설정, 스왑 메모리(Swap) 설정을 한 번에 처리하는 스크립트.

```bash
# Amazon Linux용 설정 스크립트
sudo yum update -y && \
sudo yum install docker -y && \
sudo service docker start && \
sudo usermod -aG docker ec2-user && \
sudo chmod 666 /var/run/docker.sock && \

# Swap 메모리 2GB 설정 (서버 다운 방지)
sudo dd if=/dev/zero of=/swapfile bs=128M count=16 && \
sudo chmod 600 /swapfile && \
sudo mkswap /swapfile && \
sudo swapon /swapfile && \
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab && \

# Portainer(GUI 관리툴) 실행
sudo docker run -d -p 9000:9000 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest


-----

## 3\. Jenkins(젠킨스) 설치 및 설정

### 3-1. Jenkins 컨테이너 실행

Docker in Docker(DinD) 방식을 위해 소켓을 공유하여 실행.

```bash
sudo docker run -d \
  --name jenkins \
  -p 9090:8080 -p 50000:50000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /home/ec2-user/jenkins_home:/var/jenkins_home \
  -u root \
  jenkins/jenkins:jdk17
```

### 3-2. 내부 빌드 도구 설치

젠킨스 컨테이너 내부로 진입하여 `Node.js`와 `AWS CLI`를 설치해야 함.

```bash
# 1. 젠킨스 내부 접속
sudo docker exec -u root -it jenkins bash

# 2. Node.js 20 (LTS) & AWS CLI 설치
curl -fsSL [https://deb.nodesource.com/setup_20.x](https://deb.nodesource.com/setup_20.x) | bash - && \
apt-get install -y nodejs awscli

# 3. 설치 확인
node -v && aws --version
```

-----

## 4\. 프론트엔드 배포 (S3 + CloudFront)

### 4-1. S3 버킷 설정

1.  **버킷 생성**: `previewai-frontend` (퍼블릭 액세스 허용)
2.  **속성**: 정적 웹 사이트 호스팅 **활성화** (`index.html`)
3.  **권한 (버킷 정책)**:
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "PublicReadGetObject",
                "Effect": "Allow",
                "Principal": "*",
                "Action": "s3:GetObject",
                "Resource": "arn:aws:s3:::버킷이름/*"
            }
        ]
    }
    ```

### 4-2. CloudFront (CDN) & HTTPS 설정

  * **Origin (S3)**: S3 웹사이트 엔드포인트 주소 직접 입력 (`http://...`). **Protocol: HTTP Only**.
  * **Origin (Backend)**: EC2의 Public DNS 주소 입력. **Protocol: HTTP Only**, Port: `8080`.
  * **Behavior (동작)**:
      * `/*` → S3 (프론트엔드)
      * `/api/*` → EC2 (백엔드) (Allowed Methods: All, Cache Policy: Disabled)
  * **Viewer Protocol**: `Redirect HTTP to HTTPS`
  * **Custom Domain**: `previewai.store` (Route 53 연결, ACM 인증서 적용)

-----

## 5\. 자동 배포 파이프라인 (Jenkinsfile)

GitHub에서 코드를 가져와 백엔드는 Docker로, 프론트는 S3로 배포하는 전체 스크립트.

```groovy
pipeline {
    agent any
    
    environment {
        // AWS 설정 (Jenkins Credentials에 'aws-iam-key' 등록 필요)
        S3_BUCKET_NAME = "previewai-frontend" 
        
        // 백엔드 설정
        BACKEND_IMAGE = "preview-backend"
        BACKEND_CONTAINER = "preview-api-server"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: '[https://github.com/PreView-Labs/PreView-app.git](https://github.com/PreView-Labs/PreView-app.git)'
            }
        }

        // 1. Backend: Build -> Dockerize -> Run
        stage('Backend Build & Deploy') {
            steps {
                dir('backend') {
                    sh 'chmod +x gradlew'
                    sh './gradlew clean build -x test'
                    sh "docker build -t ${BACKEND_IMAGE} ."
                    script {
                        try {
                            sh "docker stop ${BACKEND_CONTAINER}"
                            sh "docker rm ${BACKEND_CONTAINER}"
                        } catch (e) { echo "No existing container" }
                    }
                    // TZ=Asia/Seoul: 서버 시간 한국으로 설정
                    sh "docker run -d --name ${BACKEND_CONTAINER} -p 8080:9117 -e TZ=Asia/Seoul ${BACKEND_IMAGE}"
                }
            }
        }

        // 2. Frontend: Install -> Build (Vite)
        stage('Frontend Build') {
            steps {
                dir('frontend') {
                    sh 'npm install'
                    sh 'npm run build'
                }
            }
        }

        // 3. Frontend: Upload to S3
        stage('Upload to S3') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-iam-key', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    dir('frontend') {
                        // dist 폴더 내용 동기화 (삭제 옵션 포함)
                        sh "aws s3 sync dist/ s3://${S3_BUCKET_NAME} --delete --region ap-northeast-2"
                    }
                }
            }
        }
    }
}
```

-----

## 6\. 트러블슈팅 로그 (Troubleshooting)

### 🚨 Issue 1: Jenkins "Offline" 및 빌드 멈춤

  * **원인**: `t2.micro` (RAM 1GB) 메모리 부족.
  * **해결**:
    1.  EC2 인스턴스 유형을 `t3.small` (RAM 2GB)로 업그레이드.
    2.  `sudo docker restart jenkins` 로 컨테이너 재시작.

### 🚨 Issue 2: Mixed Content Error (프론트/백엔드 통신 불가)

  * **원인**: 프론트는 HTTPS(보안)인데 백엔드는 HTTP(비보안) IP로 요청함.
  * **해결**: **CloudFront**가 중간에서 중계(Proxy)하도록 설정.
      * 프론트 코드: `fetch('http://IP:8080/api/...')` ❌ → `fetch('/api/...')` ✅
      * CloudFront 동작 설정: `/api/*` 경로를 EC2(`8080`)로 전달하도록 추가.

### 🚨 Issue 3: 배포 후에도 옛날 화면이 뜸

  * **원인**: CloudFront가 캐시(Cache)된 이전 파일을 계속 보여줌.
  * **해결**: **Invalidation (무효화)** 실행.
      * 경로: `/*` (전체 파일 캐시 삭제)

### 🚨 Issue 4: Docker Build 에러 (openjdk not found)

  * **원인**: `openjdk:17-jdk-slim` 이미지가 Docker Hub에서 만료됨.
  * **해결**: AWS 환경에 최적화된 **`amazoncorretto:17`** 로 Dockerfile 수정.

-----

## 7\. 주요 명령어 요약

| 동작 | 명령어 |
| :--- | :--- |
| **서버 접속 (SSH)** | `ssh -i "key.pem" ec2-user@{IP}` |
| **젠킨스 재시작** | `sudo docker restart jenkins` |
| **도커 청소 (용량확보)** | `sudo docker system prune -a -f` |
| **젠킨스 내부 접속** | `sudo docker exec -it -u root jenkins bash` |

```
```

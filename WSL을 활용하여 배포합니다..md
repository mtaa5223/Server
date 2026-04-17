# CI/CD 자동 배포 환경 구축 (WSL + GitHub Actions + Cloud Run)

## 1. 개요

본 문서는 WSL2 기반 Ubuntu 환경에서 GitHub Actions self-hosted runner를 구축하고,  
Docker를 이용하여 Google Cloud Run으로 자동 배포하는 전체 과정을 정리한다.

구성 목표는 다음과 같다:

- 로컬 개발 환경과 서버 환경의 일치
    
- GitHub push 기반 자동 배포
    
- Docker 기반 컨테이너화
    
- Cloud Run을 통한 서버리스 배포
    

---

## 2. 전체 아키텍처

```
[개발 PC]
    ↓ (git push)
[GitHub Repository]
    ↓ (GitHub Actions 실행)
[Self-hosted Runner (WSL Ubuntu)]
    ↓ (Docker build)
[Artifact Registry]
    ↓
[Cloud Run (실제 서비스)]
```

---

## 3. 기술 스택

- WSL2 (Ubuntu)
    
- GitHub Actions (self-hosted runner)
    
- Docker
    
- Google Cloud Run
    
- Kotlin (Ktor 서버)
    

---

## 4. WSL 환경 구성

### 설치

```bash
wsl --install
```

또는

```bash
wsl --install -d Ubuntu
```

### 확인

```bash
uname -a
```
우분투 깔렸는지 확인.

출력 예시:

```
Linux localhost ... microsoft-standard-WSL2
```

→ WSL2 환경 정상 동작 확인

---

## 5. GitHub Actions Runner 설정

### Runner 다운로드 및 설치

```bash
mkdir actions-runner && cd actions-runner

curl -o actions-runner.tar.gz -L https://github.com/actions/runner/releases/download/v2.333.1/actions-runner-linux-x64-2.333.1.tar.gz

tar xzf actions-runner.tar.gz
```

### GitHub 연결

```bash
./config.sh
```

입력 항목:

- Repository URL
    
- Token
    
- Runner name
    
- Labels (선택)
    

### 실행

```bash
./run.sh
```

또는 서비스 등록:

```bash
sudo ./svc.sh install
sudo ./svc.sh start
```

---

## 6. Docker 설정

```bash
sudo apt update
sudo apt install -y docker.io
sudo usermod -aG docker $USER
```

재 로그인 후 확인:

```bash
docker ps
```

---

## 7. GCP 설정

### 로그인

```bash
gcloud auth login --no-launch-browser
```

### 프로젝트 설정

```bash
gcloud config set project helical-ion-460310-m0
```

### Docker Registry 연결

```bash
gcloud auth configure-docker asia-northeast3-docker.pkg.dev
```

---

## 8. GitHub Actions (deploy.yml)

```yaml
name: Build and Deploy to Cloud Run

on:
  push:
    branches: [main, develop]

jobs:
  build:
    runs-on: self-hosted

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker Image
        run: docker build -t IMAGE_NAME .

      - name: Push Image
        run: docker push IMAGE_NAME

  deploy:
    runs-on: self-hosted
    needs: build

    steps:
      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy trinity-server-prod \
          --image IMAGE_NAME \
          --region asia-northeast3 \
          --platform managed
```

---

## 9. 배포 흐름

1. 개발자가 코드 수정 후 GitHub에 push
    
2. GitHub Actions 실행
    
3. self-hosted runner (WSL)가 job 수행
    
4. Docker 이미지 빌드
    
5. Artifact Registry에 push
    
6. Cloud Run으로 배포
    

---

## 10. 권한 설정 (403 해결)

Cloud Run 기본 설정은 인증 필요 상태이다.

### 해결 방법

```bash
gcloud run services add-iam-policy-binding trinity-server-prod \
  --member="allUsers" \
  --role="roles/run.invoker" \
  --region=asia-northeast3
```

---

## 11. 배포 결과 확인

서비스 URL 접속:

```
https://trinity-server-prod-xxxx.a.run.app
```

정상 응답:

```
Trinity Server Alive
```

```
---

## 12. WSL의 역할

- GitHub Actions 작업 실행
    
- Docker 이미지 빌드
    
- GCP 배포 수행
    

※ 실제 서버는 Cloud Run에서 실행됨

---

## 13. 문제 해결 (Troubleshooting)

### docker: command not found

→ Docker 설치 필요

### permission denied (docker)

→ usermod 설정 후 재로그인 필요

### gcloud not found

→ Google Cloud CLI 설치 필요

### 403 Forbidden

→ Cloud Run IAM 설정 필요

### runner not working

→ ./run.sh 실행 여부 확인

---

## 14. 결론

WSL 기반 self-hosted runner를 통해  
로컬 환경에서 CI/CD 자동 배포 파이프라인을 구축하였다.

해당 구조는:

- 비용 효율적
    
- 빠른 빌드 속도
    
- 환경 일치성 확보
    

라는 장점을 가진다.

---

## 15. 향후 개선 방향

- dev / prod 환경 분리
    
- runner 다중 구성
    
- 자동 롤백 전략
    
- 로그 및 모니터링 시스템 구축
    
- 도메인 연결 및 HTTPS 설정
    

---


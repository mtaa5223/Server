# CI/CD 파이프라인 설정 가이드

## 개요

GitHub Actions(Self-hosted Runner, Windows)를 사용하여 PR 머지 시 Docker 이미지를 자동 빌드하고 GCP Artifact Registry에 push하는 파이프라인.

## 전체 흐름

```
feature 브랜치 → PR → 코드 리뷰 → develop 머지 → 스테이징 빌드/배포
                                  → main 머지    → 프로덕션 빌드/배포
```

## 파이프라인 상세

### 트리거 조건

| 브랜치    | 용도            | 생성되는 태그                       |
| --------- | --------------- | ----------------------------------- |
| `develop` | 스테이징 확인용 | `{sha}`, `dev-{sha}`, `dev-latest`  |
| `main`    | 프로덕션 배포용 | `{sha}`, `latest`, `{git-tag 버전}` |

- `ant/` 하위 소스 파일 변경 시에만 트리거 (불필요한 빌드 방지)

### 이미지 태그 전략

- **commit SHA** — 항상 붙음 (예: `abc1234`), 특정 빌드를 추적할 때 사용
- **dev-latest / latest** — 각 브랜치의 최신 이미지를 가리키는 rolling 태그
- **git tag 버전** — `main`에서만 적용, `git tag v1.0.0` 태그가 있으면 자동으로 붙음

### 이미지 경로

```
asia-northeast3-docker.pkg.dev/helical-ion-460310-m0/test/trinity-server
```

## 셋업 작업

### 1. GCP 서비스 계정 준비

```bash
# 서비스 계정 생성
gcloud iam service-accounts create github-actions-deployer \
  --display-name="GitHub Actions Deployer"

# Artifact Registry Writer 권한 부여
gcloud projects add-iam-policy-binding helical-ion-460310-m0 \
  --member="serviceAccount:github-actions-deployer@helical-ion-460310-m0.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

# JSON 키 생성
gcloud iam service-accounts keys create key.json \
  --iam-account=github-actions-deployer@helical-ion-460310-m0.iam.gserviceaccount.com
```

### 2. GitHub Secrets 등록

팀 repo → **Settings** → **Secrets and variables** → **Actions**에서 등록:

| Secret 이름  | 값                                         |
| ------------ | ------------------------------------------ |
| `GCP_SA_KEY` | GCP 서비스 계정 JSON 키 전체 내용 붙여넣기 |

### 3. Windows 빌드 컴퓨터에 Self-hosted Runner 설치

1. 팀 repo → **Settings** → **Actions** → **Runners** → **New self-hosted runner**
2. OS: **Windows** 선택
3. 안내에 따라 runner 에이전트 설치 및 서비스 등록

#### 빌드 컴퓨터 필수 사항

- [ ] Docker Desktop 설치 및 실행 중
- [ ] `gcloud` CLI 설치
- [ ] Git for Windows 설치 (bash shell 필요)

### 4. 워크플로우 파일

`.github/workflows/deploy.yml`에 생성되어 있음.

## Git에 포함되는 파일 / 제외되는 파일

### 포함 (O)

| 파일/폴더              | 이유                          |
| ---------------------- | ----------------------------- |
| `src/`                 | 소스 코드                     |
| `build.gradle.kts`     | 빌드 설정                     |
| `settings.gradle.kts`  | 프로젝트 설정                 |
| `gradle/`              | Gradle wrapper (빌드 재현성)  |
| `gradlew`, `gradlew.bat` | Gradle wrapper 실행 스크립트 |
| `gradle.properties`    | Gradle 속성                   |
| `.gitignore`           | Git 제외 규칙                 |
| `.env.example`         | 환경변수 템플릿 (실제 값 없음) |

### 제외 (X) — `.gitignore`로 관리

| 파일/폴더          | 이유                                |
| ------------------ | ----------------------------------- |
| `.env`             | DB 비밀번호 등 실제 시크릿 포함     |
| `Dockerfile`       | CI/CD 빌드 컴퓨터에서 별도 관리     |
| `docker-compose.yml` | 로컬 개발 환경용                  |
| `.dockerignore`    | Docker 빌드 전용                    |
| `build/`           | 빌드 산출물                         |
| `.gradle/`         | Gradle 캐시                         |
| `.idea/`           | IDE 설정                            |
| `.kotlin/`         | Kotlin 캐시                         |

## 사용법

### 스테이징 배포 (develop)

1. feature 브랜치에서 작업
2. `develop` 브랜치로 PR 생성
3. 코드 리뷰 후 머지
4. 자동으로 빌드 및 push → `dev-latest` 태그로 스테이징 확인

### 프로덕션 배포 (main)

1. `develop`에서 충분히 검증 완료
2. `main` 브랜치로 PR 생성
3. 코드 리뷰 후 머지
4. 자동으로 빌드 및 push → `latest` 태그로 프로덕션 배포

### 버전 태그 붙이기 (선택)

```bash
git tag v1.0.0
git push origin v1.0.0
```

이후 `main`에 머지하면 `v1.0.0` 태그가 이미지에 자동으로 붙음.

## 트러블슈팅

### Runner가 오프라인으로 표시됨
- 빌드 컴퓨터에서 runner 서비스가 실행 중인지 확인
- `./run.cmd` 또는 Windows 서비스 관리자에서 상태 확인

### Docker 빌드 실패
- Docker Desktop이 실행 중인지 확인
- `docker info` 명령으로 Docker 데몬 상태 확인

### GCP push 실패
- `GCP_SA_KEY` secret이 올바르게 등록되었는지 확인
- 서비스 계정에 `roles/artifactregistry.writer` 권한이 있는지 확인
- `gcloud auth configure-docker asia-northeast3-docker.pkg.dev` 수동 실행하여 테스트

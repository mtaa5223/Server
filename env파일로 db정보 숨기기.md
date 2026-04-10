# application.conf 환경변수화 계획 (옵션 B: .env + dotenv-kotlin)

## 0. 선행 조치 (긴급)

- Neon DB 비밀번호 즉시 로테이션  
    → 기존 비밀번호가 코드(`DataFactory.kt`)에 평문 노출됨
- 새 비밀번호는 절대 코드/설정 파일에 저장하지 않음  
    → `.env` → 환경변수로만 관리

---

## 1. 환경변수 네이밍 규칙

- 형식: `SCREAMING_SNAKE_CASE`
- Prefix 사용:
    - `APP_`
    - `DB_`
    - `UGS_`

정책:

- 시크릿 값 → 기본값 없음 (누락 시 기동 실패)
- 비시크릿 값 → `application.conf`에 기본값 유지

---

## 2. 최종 환경변수 목록

|변수|시크릿|기본값|용도|
|---|---|---|---|
|PORT|❌|8080|Cloud Run 자동 주입|
|APP_ENV|❌|local|환경 구분|
|DB_URL|❌|로컬값|DB 연결|
|DB_USER|❌|postgres|DB 계정|
|DB_PASSWORD|✅|없음|DB 비밀번호|
|DB_POOL_SIZE|❌|10|커넥션 풀|
|UGS_PROJECT_ID|❌|기본값 있음|Unity Project ID|
|UGS_ISSUER|❌|기본값 있음|JWT issuer|
|UGS_JWKS_URL|❌|기본값 있음|JWKS URL|

---

## 3. application.conf 최종안

ktor {  
  deployment {  
    port = 8080  
    port = ${?PORT}  
    host = "0.0.0.0"  
  }  
  application {  
    modules = [ com.example.MainKt.module ]  
  }  
}  
  
app {  
  env = "local"  
  env = ${?APP_ENV}  
}  
  
database {  
  url = "jdbc:postgresql://localhost:5432/newtrinity"  
  url = ${?DB_URL}  
  user = "postgres"  
  user = ${?DB_USER}  
  password = ""  
  password = ${?DB_PASSWORD}  
  poolSize = 10  
  poolSize = ${?DB_POOL_SIZE}  
}  
  
ugs {  
  projectId = "upid:d044100d-5ae9-4b04-aa42-a06e5a47e81d"  
  projectId = ${?UGS_PROJECT_ID}  
  issuer = "https://player-auth.services.api.unity.com"  
  issuer = ${?UGS_ISSUER}  
  jwksUrl = "https://player-auth.services.api.unity.com/.well-known/jwks.json"  
  jwksUrl = ${?UGS_JWKS_URL}  
}

패턴:

key = default  
key = ${?ENV_VAR}

설명:

- ENV 존재 → override
- ENV 없음 → default 사용

---

## 4. dotenv-kotlin 도입

### 4-1. 의존성 추가

dependencies {  
    implementation("io.github.cdimascio:dotenv-kotlin:6.4.1")  
}

---

### 4-2. Main.kt 수정

fun main(args: Array<String>) {  
    io.github.cdimascio.dotenv.dotenv {  
        ignoreIfMissing = true  
        systemProperties = true  
    }  
    io.ktor.server.netty.EngineMain.main(args)  
}

핵심:

- `systemProperties = true`  
    → `.env` 값을 HOCON에서 읽을 수 있도록 함

---

## 5. .env.example (커밋용)

# App  
APP_ENV=local  
  
# Database (local dev)  
DB_URL=jdbc:postgresql://localhost:5432/newtrinity_dev  
DB_USER=postgres  
DB_PASSWORD=changeme  
DB_POOL_SIZE=10  
  
# UGS (override if needed)  
# UGS_PROJECT_ID=  
# UGS_ISSUER=  
# UGS_JWKS_URL=

사용 방법:

cp .env.example .env

---

## 6. .gitignore 설정

.env  
.env.local  
.env.*.local  
*.local.conf

- `.env.example`만 커밋
- `.env`는 절대 커밋 금지

---

## 7. 코드 리팩터 포인트

### 7-1. DataFactory.kt

변경 전:

- DB 정보 하드코딩

변경 후:

fun init(config: ApplicationConfig)

- config 기반으로 DB 초기화
- `by lazy` 제거
- Flyway는 init 내부 유지

---

### 7-2. Main.kt

DataFactory.init(environment.config)

---

### 7-3. UgsJwtConfig.kt

- 하드코딩 제거
- `environment.config.config("ugs")` 사용

---

## 8. 로컬 실행 흐름

1. `.env.example → .env` 복사
2. `.env` 값 입력
3. 앱 실행
4. dotenv → systemProperties 등록
5. application.conf → ENV 치환
6. DB 연결

---

## 9. Cloud Run 실행 흐름

1. `.env 없음`
2. dotenv → 무시
3. Cloud Run이 ENV 주입
4. application.conf 동일하게 동작

---

## 10. Cloud Run 설정 예시

### 비시크릿

--set-env-vars APP_ENV=prod,DB_URL=...,DB_USER=...

### 시크릿

--set-secrets DB_PASSWORD=db-password:latest

---

## 11. 작업 체크리스트

- [ ]  Neon DB 비밀번호 로테이션
- [ ]  dotenv-kotlin 추가
- [ ]  application.conf 적용
- [ ]  Main.kt 수정
- [ ]  DataFactory 리팩터
- [ ]  UgsJwtConfig 수정
- [ ]  .env.example 생성
- [ ]  .gitignore 설정
- [ ]  로컬 실행 테스트
- [ ]  Git 민감정보 확인 및 정리

---

## 핵심 요약

- 설정은 `application.conf`
- 값은 `.env` 또는 Cloud Run ENV
- 코드 수정 없이 환경만 바꿔 배포 가능
# Ktor 서버 구조 및 JWT 인증 정리

## 1. Ktor란 무엇인가

**Ktor**는 Ktor 기반의  
비동기 웹 애플리케이션 프레임워크다.

특징:

- Kotlin 기반
- 비동기 처리 (Coroutine)
- 구조를 강제하지 않음 (직접 설계 필요)

---

## 2. Ktor 전체 구조

Ktor는 다음과 같은 계층으로 구성된다:

- **Application** → 서버 전체
- **Module** → 서버 설정 단위
- **Routing** → API 엔드포인트
- **Service / Domain** → 비즈니스 로직

핵심:

> Ktor는 구조를 강제하지 않기 때문에 아키텍처를 직접 설계해야 한다.

---

## 3. Feature 기반 구조 (권장)

기능 단위로 묶는 구조:

app/  
 ├─ customer/  
 │   ├─ CustomerRoutes.kt  
 │   ├─ CustomerService.kt  
 │   └─ CustomerDto.kt  
 └─ order/  
     ├─ OrderRoutes.kt  
     ├─ OrderService.kt  
     └─ OrderDto.kt

특징:

- 기능 단위로 분리 (feature-based)
- 하나의 feature = 작은 백엔드 단위
- 라우팅 / 서비스 / DTO를 함께 관리

장점:

- 확장성 좋음
- 유지보수 용이
- 마이크로서비스로 분리 가능

---

## 4. JWT 인증 구조 (UGS 기반)

### 4.1 공개키(JWK) 로드 및 캐싱

val jwkProvider: JwkProvider = JwkProviderBuilder(URI(jwksUrl).toURL())  
    .cached(10, 24, TimeUnit.HOURS)  
    .rateLimited(10, 1, TimeUnit.MINUTES)  
    .build()

설명:

- `cached`  
    → 공개키를 메모리에 캐싱 (24시간 유지)
- `rateLimited`  
    → JWKS 요청을 분당 10회로 제한 (과도한 요청 방지)

---

### 4.2 application.conf 설정

ugs {  
  projectId = "upid:xxxx"  
  issuer = "https://player-auth.services.api.unity.com"  
  jwksUrl = "https://player-auth.services.api.unity.com/.well-known/jwks.json"  
}

역할:

- **issuer** → 토큰 발급자 검증
- **projectId (audience)** → 대상 서비스 검증
- **jwksUrl** → 공개키 조회 URL

---

### 4.3 설정 로드 코드

companion object {  
    fun from(application: Application): UgsJwtConfig {  
        val cfg = application.environment.config.config("ugs")  
        return UgsJwtConfig(  
            issuer = cfg.property("issuer").getString(),  
            audience = cfg.property("projectId").getString(),  
            jwksUrl = cfg.property("jwksUrl").getString(),  
        )  
    }  
}

설명:

- application.conf에서 설정을 읽어와 객체로 변환
- 인증 설정에서 사용

---

### 4.4 Ktor 인증 설정

fun Application.configureAuthentication() {  
    val ugs = UgsJwtConfig.from(this)  
  
    install(Authentication) {  
        jwt("ugs") {  
            realm = "trinity"  
  
            verifier(ugs.jwkProvider, ugs.issuer) {  
                withAudience(ugs.audience)  
                acceptLeeway(5)  
            }  
  
            validate { cred ->  
                val sub = cred.payload.subject  
                if (!sub.isNullOrBlank()) JWTPrincipal(cred.payload) else null  
            }  
  
            challenge { _, _ ->  
                call.respond(HttpStatusCode.Unauthorized)  
            }  
        }  
    }  
}

---

## 5. 인증 흐름

전체 흐름:

Client → JWT 포함 요청  
        ↓  
Ktor Authentication  
        ↓  
서명 검증 (JWK)  
        ↓  
issuer 검증 (UGS)  
        ↓  
audience 검증 (projectId)  
        ↓  
payload 검증 (sub 존재 여부)  
        ↓  
JWTPrincipal 생성  
        ↓  
Route에서 사용

---

## 6. 주요 개념 정리

- **install(Authentication)**  
    → 인증 시스템 활성화
- **jwt("ugs")**  
    → "ugs" 이름의 JWT 인증 등록
- **verifier**  
    → JWT 서명 + issuer + audience 검증
- **validate**  
    → 추가 검증 및 Principal 생성
- **JWTPrincipal**  
    → 인증된 사용자 정보 객체 (payload 포함)
- **challenge**  
    → 인증 실패 시 401 반환

---

## 7. 핵심 요약

- Ktor는 구조를 강제하지 않는 프레임워크
- Feature 기반 구조가 유지보수에 유리
- JWT 인증은 **서명 + issuer + audience** 3단계 검증이 핵심
- 공개키(JWKS)는 캐싱과 rate limit 설정이 중요
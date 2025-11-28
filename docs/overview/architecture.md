# 시스템 아키텍처

> MindMate는 **React 기반 SPA**,
> **Spring Boot 기반 백엔드 API**,
> 그리고 **AWS 인프라(EC2 · RDS · Redis · S3 · CloudFront · Route53)** 위에서 동작하는
> 감정 기록 및 AI 분석 중심의 개인 관리 서비스입니다.
>
> 이 문서는 전체 시스템을 **상위 수준 구조 + 핵심 요청 흐름** 중심으로 설명합니다.

---

# 1. 전체 구성 개요

MindMate는 다음 5개 계층으로 구성된다.

1. **Client Browser**
2. **Frontend SPA (React) — S3 + CloudFront**
3. **Backend API (Spring Boot) — Docker + EC2**
4. **Infra Services — RDS, Redis, S3**
5. **Network & Security — Nginx, SSL, Route53, VPC**

---

## 1-1. 구성 요소 상세

### ▶ Frontend (React SPA)

-   React 18 기반 SPA
-   React Router 라우팅
-   Axios 기반 API 통신
-   최종 빌드는 **S3에 업로드**, CloudFront를 통한 CDN 서빙

### ▶ Backend (Spring Boot API Server)

-   Spring Boot 3.x (Java 17)
-   Spring Security 기반 JWT 인증 구조
-   Docker 이미지로 빌드 후 EC2에 배포
-   주요 도메인: 회원관리, 일기/AI 분석, 커뮤니티, 통계, 이미지 업로드 등

### ▶ AWS 서비스 역할

| 서비스     | 역할                                           |
| ---------- | ---------------------------------------------- |
| EC2        | Docker Host, Nginx + Backend + Redis           |
| RDS        | 주요 영속 데이터 저장                          |
| Redis      | Refresh Token, 이메일 인증 코드 등 단기 데이터 |
| S3         | 프로필/일기 이미지 및 정적 리소스 저장         |
| CloudFront | 프론트 정적 리소스 CDN                         |
| Route53    | 도메인 관리                                    |
| Certbot    | SSL 인증서 자동 갱신                           |

---

# 2. 요청 흐름

정적 리소스와 동적 API는 서로 다른 경로와 인프라를 통해 처리된다.

---

## 2-1. React 정적 리소스 로딩

```mermaid
sequenceDiagram
    autonumber
    participant User as 사용자
    participant R53 as Route53
    participant CF as CloudFront
    participant S3 as S3_Static
    participant App as React_SPA

    User ->> R53: mindmate_co_kr 접속
    R53 ->> CF: 도메인 라우팅
    CF ->> S3: 정적 파일 요청
    S3 -->> User: index_html 및 JS CSS 전달
    User ->> App: React 앱 실행
```

---

## 2-2. API 요청 흐름

```mermaid
sequenceDiagram
    autonumber
    participant FE as React_App
    participant R53 as Route53_api
    participant EC2 as EC2_Host
    participant NX as Nginx
    participant BE as Spring_Backend
    participant RDS as MySQL
    participant RD as Redis
    participant S3 as S3_Bucket

    FE ->> R53: api_mindmate_co_kr 요청
    R53 ->> EC2: API 트래픽 전달
    EC2 ->> NX: HTTPS 수신
    NX ->> BE: api 경로 리버스 프록시

    BE ->> RD: 토큰 및 인증코드 조회
    BE ->> RDS: 비즈니스 데이터 처리
    BE ->> S3: 이미지 업로드·조회
    BE -->> FE: JSON 응답
```

---

# 3. 인증 아키텍처 (JWT + Redis)

MindMate 인증 흐름은 다음 요소로 구성된다.

### ① Access Token

-   짧은 유효시간
-   요청 시 Authorization 헤더로 전송

### ② Refresh Token

-   Redis에 `userId + accessToken + refreshToken` 형태로 저장
-   HttpOnly 쿠키로 전달
-   재발급 실패 시 즉시 무효 처리

### ③ Redis

-   RT TTL 기반 로그인 유지 관리
-   이메일 인증 코드 TTL 관리

---

## 3-1. 인증·재발급 흐름 요약

```mermaid
sequenceDiagram
    autonumber
    participant FE as React_Axios
    participant BE as Auth_API
    participant RD as Redis

    FE ->> BE: 보호된 API 요청 - Access Token 포함
    BE -->> FE: 401 Access token expired

    FE ->> BE: POST auth_refresh - 만료된 AT 포함
    BE ->> RD: RefreshToken 조회 및 검증
    RD -->> BE: 검증 결과

    alt 성공
        BE -->> FE: 새 AT 반환
        FE ->> FE: 토큰 갱신 및 원래 요청 재전송
    else 실패
        BE -->> FE: 401 인증 무효
        FE ->> FE: 로컬 토큰 삭제 및 로그아웃 처리
    end
```

> 상세 로직은 `auth-jwt.md`, `auth-axios.md` 문서에 존재하므로 이 문서에서는 개요만 제공.

---

# 4. 이미지 처리 아키텍처

업로드 이미지는 **프론트 1차 압축 → 백엔드 리사이즈 → S3 저장**의 통일된 파이프라인으로 처리한다.

### 설계 원칙

-   DB에는 **이미지 URL만 저장**
-   백엔드에서 **해상도 통일(512px)** 및 **최종 JPEG 압축**
-   회원탈퇴 시 `profile/{userId}`, `diary/{userId}` 경로 전체 삭제

---

## 4-1. 이미지 업로드 흐름

```mermaid
sequenceDiagram
    autonumber
    participant FE as 프론트엔드
    participant BE as UserService
    participant S3S as S3Service
    participant S3 as AWS_S3
    participant DB as User_DB

    FE ->> BE: multipart 업로드 요청
    BE ->> S3S: 리사이즈 및 JPEG 압축
    S3S ->> S3: 업로드 - userId 기반 경로
    S3 -->> S3S: 업로드 URL 반환
    S3S -->> BE: 최종 URL 전달
    BE ->> DB: User 프로필 이미지 URL 저장
    BE -->> FE: 저장된 URL 포함한 최신 유저 정보 반환
```

---

# 5. 이메일 인증 아키텍처

-   인증 코드 TTL: 300초
-   요청 제한: 1시간 5회
-   메일 전송: `@Async` 비동기
-   Redis 장애 시 세션 기반 fallback

(상세 로직은 `auth-email.md` 참고)

---

# 6. 서버 인프라 구성 (EC2 + Docker + Nginx)

EC2 내 Docker Compose 구조:

```
EC2
 └─ docker-compose.yml
      ├─ mindmate-nginx        80, 443
      ├─ mindmate-backend      Spring Boot API
      └─ mindmate-redis        Redis
```

### Nginx 역할

-   HTTPS 종단 (SSL Termination)
-   `/api/**` → backend 프록시
-   Certbot 자동 갱신

### Backend 컨테이너

-   포트 8888에서 API 제공
-   `.env.prod` 환경 변수로 RDS/Redis/S3 설정 주입

---

# 7. 데이터 저장 구조 요약

### MySQL RDS

-   User, Account, Social
-   Diary, AIResult
-   Board, Comment, Like 등 핵심 도메인 저장

### Redis

-   Refresh Token
-   이메일 인증 코드
-   OAuth 임시 데이터

### S3

-   프로필 이미지
-   일기 이미지
-   정적 프론트 리소스 버킷

---

# 8. 설계 철학

### 1) 역할 분리

-   정적 리소스: S3 + CloudFront
-   동적 API: EC2 + Docker
-   인증/세션: Redis
-   이미지: S3
-   영속 데이터: RDS

### 2) 일관된 인증 구조

-   JWT + Redis 기반 AT/RT
-   프론트 Axios 인터셉터로 만료 + 재발급 자동 처리

### 3) 운영 효율성

-   Docker Compose로 배포 방식 통일
-   SSL 자동 갱신
-   유저별 S3 경로로 탈퇴 시 전체 청소 용이

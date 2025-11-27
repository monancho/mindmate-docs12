# 회원 관리 · 인증 시스템 (JWT + Refresh Token + Redis)

> 이 문서는 MindMate에서 **이메일/비밀번호 기반 로그인**과  
> **JWT + Refresh Token + Redis**로 구성된 인증 구조를 정리한 기술 문서입니다.  
> 소셜 로그인, 프로필 이미지, Axios 인터셉터는 각각 별도 문서에서 다룹니다.

---

## 1. 담당 범위 및 역할

### 담당 범위

-   이메일/비밀번호 회원가입 및 로그인
-   Access Token / Refresh Token 발급 구조 설계·구현
-   Redis 기반 Refresh Token 저장 및 TTL 관리
-   토큰 재발급 플로우 및 예외 처리
-   로그아웃·재로그인 정책 정의

### 목표

-   세션이 아닌 **JWT 기반 무상태 인증 구조** 설계
-   Redis를 활용한 **장기 로그인 경험 + 보안성** 확보
-   프론트 Axios 인터셉터와 연동되는 **예측 가능한 에러·재발급 플로우** 제공
-   배포 환경(AWS EC2, Docker, Redis, RDS)에서 안정적으로 동작하는 구조

---

## 2. 전체 인증 흐름 개요

### 2.1 사용자 관점 플로우

1. 사용자가 이메일/비밀번호로 로그인 요청
2. 서버에서 계정 검증 후 **Access Token + Refresh Token 발급**
3. 프론트는:
    - Access Token → 메모리/스토리지에 저장
    - Refresh Token → 쿠키를 통해 관리 (서버 재발급용)
4. 이후 모든 보호된 API 요청은 **Access Token**으로 인증
5. Access Token 만료 시:
    - Axios 인터셉터가 401 응답을 감지
    - Refresh Token을 포함해 **재발급 API** 호출
    - 성공 시 새 Access Token으로 원래 요청 재시도
6. 재발급 실패 또는 Refresh Token 문제 발생 시 → **전체 로그아웃**

> 프론트 측 구현(Axios 인터셉터, 요청 큐 처리 등)은  
> `frontend/auth-integration.md` 문서에서 상세히 다룹니다.

### 2.2 서버 관점 컴포넌트

-   `User` 엔티티
    -   이메일, 비밀번호(BCrypt), 닉네임, 프로필 정보 등 보관
-   `JwtUtil`
    -   Access/Refresh Token 생성·검증, 클레임 파싱
-   `JwtAuthenticationFilter`
    -   요청 헤더에서 JWT 추출, 검증, `SecurityContext`에 인증 정보 주입
-   `Redis`
    -   `Refresh Token` 및 이메일 인증 코드 등 **단기 데이터 + TTL** 관리
-   `AuthController`
    -   `/signup`, `/login`, `/reissue`, `/logout` 등 인증 관련 엔드포인트 제공

---

## 3. 토큰 전략 (Access Token / Refresh Token)

### 3.1 구분 및 역할

| 구분          | 역할                                  | 저장 위치 예시             | 만료 시간 예시 |
| ------------- | ------------------------------------- | -------------------------- | -------------- |
| Access Token  | API 요청 인증, 권한 확인              | 브라우저 (메모리/스토리지) | 분 단위 (짧게) |
| Refresh Token | Access Token 재발급 전용, 장기 로그인 | 서버(Redis) + 클라이언트   | 일/주 단위     |

-   Access Token은 **자주 교체되는 단기 토큰**
-   Refresh Token은 **Redis에 저장 + TTL 관리**로 장기 세션 역할 수행

### 3.2 Refresh Token 저장 구조 (Redis)

-   Key 예시: `RT:{userId}`
-   Value: Refresh Token 문자열
-   TTL: Refresh Token 만료 시간과 동일하게 설정

정책:

-   한 사용자는 **가장 최근 로그인 기준 Refresh Token 1개만 유지**하는 것을 기본 정책으로 사용
-   로그아웃 또는 이상 징후 발생 시 해당 키 삭제 → 즉시 무효화

---

## 4. 도메인 및 데이터 구조

### 4.1 User 엔티티(요약)

-   `id` : Long, PK
-   `email` : String, Unique
-   `password` : String, BCrypt 해시
-   `nickname` : String
-   `role` : Enum(USER, ADMIN 등)
-   기타: 프로필 정보(MBTI, 생일, 프로필 이미지 경로 등)

> 프로필 세부 구조와 S3 연동은 `backend/profile-s3.md`에서 별도 설명.

### 4.2 인증 관련 Redis 데이터

1. **Refresh Token**

    - Key: `RT:{userId}`
    - Value: Refresh Token 문자열
    - TTL: Refresh Token 만료 시간

2. **이메일 인증 코드(선택사항)**
    - Key: `EC:{email}`
    - Value: 인증 코드
    - TTL: 짧은 시간(예: 5분)

---

## 5. 주요 API 설계

### 5.1 회원가입: `POST /api/auth/signup`

**요청**

-   Body: `{ email, password, nickname, ... }`

**처리**

1. 이메일 중복 여부 확인
2. 비밀번호 BCrypt 해시 처리
3. User 엔티티 생성 및 DB 저장

**응답**

-   성공 메시지 또는 기본 프로필 정보
-   JWT 발급은 로그인 API에서 처리

---

### 5.2 로그인: `POST /api/auth/login`

**요청**

-   Body: `{ email, password }`

**처리 플로우**

1. 이메일로 사용자 조회
2. BCrypt로 비밀번호 매칭
3. 검증 성공 시:
    - Access Token 생성
    - Refresh Token 생성
    - Redis에 `RT:{userId}` = Refresh Token, TTL 설정
4. 이전 Refresh Token 정책
    - 기존 값이 있으면 덮어쓰기(마지막 로그인 기준 1개 유지)

**응답**

-   Body: `accessToken`, 사용자 기본 정보
-   Cookie: Refresh Token (HttpOnly, Secure(HTTPS 환경), Path 등 정책 반영)

---

### 5.3 토큰 재발급: `POST /api/auth/reissue`

**요청**

-   헤더/바디: (선택) 만료된 Access Token
-   쿠키: Refresh Token

**처리 플로우**

1. 요청에서 Refresh Token 추출
2. Refresh Token 유효성 검증 (서명, 만료)
3. 토큰에서 `userId` 추출
4. Redis에서 `RT:{userId}` 조회 후, **저장된 토큰과 요청 토큰 비교**
5. 일치하면:
    - 새 Access Token 생성
    - (필요 시) Refresh Token 재발급 후 Redis 갱신
6. 불일치 or Redis에 값 없음:
    - 탈취/로그아웃된 토큰으로 판단 → 전체 로그아웃 유도

**응답**

-   새 Access Token (+ 선택적으로 새 Refresh Token)

---

### 5.4 로그아웃: `POST /api/auth/logout`

**처리 플로우**

1. 인증된 사용자의 `userId` 확인
2. Redis에서 `RT:{userId}` 삭제
3. Refresh Token 쿠키 삭제 지시

이후:

-   Access Token은 만료 시점까지는 유효하지만,
-   만료 후 재발급이 불가능하므로 자연스럽게 세션 종료

---

### 5.5 내 정보 조회: `GET /api/auth/me`

-   `JwtAuthenticationFilter`를 통과한 사용자만 접근 가능
-   `@AuthenticationPrincipal` 또는 SecurityContext에서 로그인 사용자 정보를 가져와 응답
-   프론트에서 로그인 상태 유지/복원 등에 활용

---

## 6. Spring Security & JWT 필터 구조

### 6.1 Security 설정 개요

-   `/api/auth/signup`, `/api/auth/login`, 소셜 콜백 등은 `permitAll()`
-   `/api/**` 중 보호가 필요한 URI는 인증 필요
-   `JwtAuthenticationFilter`를 UsernamePasswordAuthenticationFilter 앞단에 등록

### 6.2 JwtAuthenticationFilter 동작 순서

1. HTTP 헤더에서 `Authorization` 값 추출
2. `"Bearer "` 접두사 확인 후 JWT 분리
3. `JwtUtil`로 토큰 유효성 검증
4. 유효하면:
    - 토큰에서 사용자 정보 추출 → Authentication 객체 생성
    - `SecurityContextHolder`에 설정
5. 유효하지 않으면:
    - 재발급 대상(만료)인지, 위조/오류인지 구분
    - 재발급은 `reissue` 엔드포인트에서 처리 (필터 단에서 직접 재발급 X)

---

## 7. 예외 처리 및 에러 응답 정책

### 7.1 Access Token 관련

-   **만료된 토큰**

    -   응답: 401 Unauthorized
    -   메시지 예: `"ACCESS_TOKEN_EXPIRED"`
    -   프론트: 재발급 시도 후 원래 요청 재전송

-   **위조/손상된 토큰**
    -   응답: 401/403
    -   메시지 예: `"INVALID_TOKEN"`
    -   프론트: 즉시 로그아웃 및 재로그인 유도

### 7.2 Refresh Token 관련

-   **Redis에 존재하지 않는 경우**

    -   이미 로그아웃했거나 탈취된 토큰으로 판단
    -   응답: 401, `"REFRESH_TOKEN_NOT_FOUND"`

-   **Redis 값과 불일치**

    -   탈취 가능성 → Redis 키 삭제, 전체 로그아웃 처리
    -   응답: 401, `"REFRESH_TOKEN_MISMATCH"`

-   **만료된 Refresh Token**
    -   응답: 401, `"REFRESH_TOKEN_EXPIRED"`
    -   완전한 재로그인 요구

---

## 8. 보안 및 운영 상 고려사항

-   `JWT_SECRET` 및 DB/Redis/S3 자격 증명은 `.env`로 분리하고 Git에 노출 금지
-   HTTPS 환경에서 Refresh Token 쿠키에 `Secure` 옵션 적용
-   CORS 설정 시, 실제 프론트 도메인만 허용 (로컬/배포 도메인 분리 관리)
-   Redis 장애 시:
    -   토큰 재발급을 막고 재로그인 요구하는 방향으로 설계 (안전 우선)
-   배포 환경에서는:
    -   Docker + EC2 + Nginx 조합으로 백엔드를 운영
    -   Redis 컨테이너와 백엔드 컨테이너는 내부 도커 네트워크로만 통신

---

## 9. 회고 및 확장 포인트

-   JWT + Refresh Token + Redis 구조를 직접 설계·구현하면서:
    -   **토큰 수명 관리**, **예외 상황 처리**, **프론트 연동**까지 전체 흐름을 경험
    -   단순 로그인 기능을 넘어, **운영 가능한 인증 구조**를 구성하는 감각을 쌓음
-   추후 확장 아이디어
    -   기기별 Refresh Token 관리 (멀티 디바이스 정책)
    -   비밀번호 변경/회원탈퇴 시, 전체 토큰 블랙리스트 처리
    -   관리자 전용 권한/역할 구조 세분화

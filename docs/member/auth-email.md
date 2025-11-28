# 이메일 인증 코드 시스템

> MindMate에서 회원가입 시 사용하는 **이메일 인증 코드(6자리 숫자)** 시스템 문서이다.
> 이 문서는 인증 코드 생성·저장·전송·검증·요청 제한·상태 관리까지의 흐름을 정리한다.
> 메일 발송 실패, Redis 장애 등 예외 상황에서의 Fallback 전략도 함께 다룬다.

---

## 1. 개요

이메일 인증 시스템은 다음 요구사항을 기반으로 설계되었다.

-   6자리 숫자 코드 기반 이메일 인증
-   **1시간 안에 최대 5회**까지만 코드 요청 허용

    -   마지막 요청 시점 기준 **슬라이딩 1시간 5회 제한**

-   인증 코드는 **Redis 기반 `Email` 엔티티(@RedisHash, TTL=300초)** 에 저장
-   Redis 장애 시 **HttpSession으로 Fallback**
-   실제 메일 전송은 `@Async` 비동기로 처리
-   발송 상태값 저장 (REQUESTED, SENT, FAILED)

이 구조를 통해, 과도한 인증 요청을 제어하면서도
Redis 장애 시에도 최소 기능은 유지할 수 있도록 설계했다.

> ※ 이메일 발송 상태(`REQUESTED`, `SENT`, `FAILED`)는 서버 단에서 기록되며,
> 현재 UI에서는 직접 조회하지 않지만 향후 전송 상태 표시용으로 활용 가능하다.

---

## 2. 관련 API (이메일 인증용)

`AuthController` 기준, 이메일 인증과 직접 연관된 엔드포인트는 다음과 같다.

| 기능                           | HTTP | 엔드포인트                 | 설명                                                                                     |
| ------------------------------ | ---- | -------------------------- | ---------------------------------------------------------------------------------------- |
| 이메일 중복 확인 + 코드 발송   | GET  | `/api/auth/check_username` | `username`(=이메일) 쿼리 파라미터로 중복 여부 확인 후, 사용 가능하면 인증 코드 메일 발송 |
| 인증 코드 소프트 체크          | POST | `/api/auth/check_code`     | Body의 `email`, `code`로 코드 유효 여부만 확인 (코드는 유지)                             |
| 인증 코드 하드 체크 + 회원가입 | POST | `/api/auth/signup`         | Body의 `email`, `code`를 검증(`isEmailCode`), 성공 시 코드 삭제 후 회원가입 진행         |

> **핵심 요약**
>
> -   인증 코드는 **`/check_username` 호출 시 발급 + 메일 발송**
> -   **UI에서 미리 확인**: `/check_code` (소프트 체크, 코드 유지)
> -   **실제 회원가입**: `/signup` 내부에서 `isEmailCode`로 하드 체크(코드 삭제)

---

## 3. 구성 요소

| 컴포넌트           | 역할                                                     |
| ------------------ | -------------------------------------------------------- |
| `EmailService`     | 코드 생성·저장, 요청 제한, 상태 조회, 비즈니스 로직 핵심 |
| `AsyncMailService` | 메일 전송 비동기 처리, 전송 결과에 따른 상태값 갱신      |
| `RedisService`     | 단순 String 기반 Redis 연산(get/set/delete + TTL) 래핑   |
| `EmailRepository`  | 인증 코드 저장용 Redis 엔티티(`Email`, TTL=300초)        |
| `JavaMailSender`   | 실제 메일 전송(Spring Mail)                              |
| `HttpSession`      | Redis 장애 시 코드·카운트 Fallback 저장소                |

주요 Prefix/Key:

-   요청 횟수: `email:verify:count:{email}`
-   상태 값: `email:verify:status:{email}` (REQUESTED / SENT / FAILED)
-   코드 저장:

    -   Redis: `Email` 엔티티 (key = email, value = code, TTL=300초)
    -   Session Fallback: `emailCode:{email}`

---

## 4. 인증 코드 요청 흐름

### 4.1 전체 흐름 요약

1. 클라이언트가 **`GET /api/auth/check_username?username={email}`** 호출
2. 서버에서:

    - 이메일 null/blank 여부 검사
    - 기존 가입 여부 검사 (`userRepository.findByEmail`)

3. 사용 가능한 이메일이면:

    - 1시간 요청 횟수(최대 5회) 초과 여부 검사 (`isOverLimitAndIncrease`)
    - 6자리 숫자 코드 생성 (`generateCode`)
    - Redis `Email` 엔티티에 코드 저장 (TTL 300초)

        - 실패 시 HttpSession에 저장

    - Redis에 상태값 `REQUESTED` 기록
    - HTML 메일 생성 후, 비동기 메일 발송 요청 (`AsyncMailService.sendEmailAsync`)

4. 응답:

    - 정상: 200 OK `"이메일 전송 요청을 완료 했습니다."`
    - 요청 과다: 429 TOO_MANY_REQUESTS + 안내 메시지

### 4.2 Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant A as AuthController
    participant AS as AccountService
    participant ES as EmailService
    participant R as RedisService/EmailRepo
    participant H as HttpSession
    participant MS as AsyncMailService
    participant M as MailServer

    C ->> A: GET /api/auth/check_username?username={email}
    alt 이메일 null/blank
        A -->> C: 400 BAD_REQUEST ("이메일을 입력해 주세요.")
    else 이메일 형식 정상
        A ->> AS: checkUsername(email)
        alt 이미 존재하는 이메일
            AS -->> A: Optional<User>.present
            A -->> C: 409 CONFLICT ("이미 사용중인 이메일입니다")
        else 사용 가능한 이메일
            AS -->> A: Optional.empty
            A ->> ES: sendMailWithCode(email, request)

            ES ->> R: email:verify:status:{email} = REQUESTED (10분 TTL)
            ES ->> ES: createAndStoreCode(email, request)
            ES ->> ES: generateCode() (6자리)
            ES ->> R: Email(email, code) 저장 (TTL=300초)
            alt Redis 저장 실패
                ES ->> H: 세션에 "emailCode:{email}=code" 저장
            end
            ES ->> ES: createMail(email, code)
            ES ->> MS: sendEmailAsync(email, message)
            ES -->> A: 200 OK ("이메일 전송 요청을 완료 했습니다.")
            A -->> C: 200 OK ("이메일 전송 요청을 완료 했습니다.")

            MS ->> M: 메일 전송
            alt 메일 전송 성공
                MS ->> R: email:verify:status:{email} = SENT (10분)
            else 전송 실패
                MS ->> R: email:verify:status:{email} = FAILED (10분)
            end
        end
    end
```

---

## 5. 인증 코드 생성 및 저장

### 5.1 코드 생성

```java
private String generateCode() {
    // 100000 ~ 999999 범위에서 랜덤 6자리
    return Integer.toString((int) (Math.random() * 900000) + 100000);
}
```

-   6자리 숫자 코드
-   단순 난수 기반

### 5.2 코드 저장 로직

```java
public String createAndStoreCode(String email, HttpServletRequest request) {
    // 1시간 5회 제한 체크
    if (isOverLimitAndIncrease(email, request)) {
        throw new IllegalStateException("인증코드 요청 가능 횟수를 초과했습니다. 잠시 후 다시 시도해주세요.");
    }

    String code = generateCode();

    // Redis에 Email 엔티티 저장 (유효시간 300초)
    try {
        Email emailEntity = new Email(email, code);
        emailRepository.save(emailEntity); // @RedisHash(timeToLive=300)
    } catch (Exception e) {
        // Redis 장애 시 세션에 저장
        HttpSession session = request.getSession(true);
        session.setAttribute("emailCode:" + email, code);
    }

    return code;
}
```

정책:

-   기본 저장소: Redis 기반 `EmailRepository` (`@RedisHash(timeToLive=300)`)
-   예외 발생 시:

    -   HttpSession에 `"emailCode:{email}"` 형태로 코드 저장

> **요청 제한의 의미**
>
> -   `isOverLimitAndIncrease`는 **매 요청마다 TTL을 1시간으로 다시 설정**하기 때문에,
>     “정해진 1시~2시” 같은 고정 구간이 아니라
>     **마지막 요청 시점 기준 1시간 동안 최대 5회**를 허용하는 슬라이딩 윈도우에 가깝다.

---

## 6. 요청 횟수 제한 (1시간 5회)

### 6.1 Redis 기반 카운트

```java
private boolean isOverLimitAndIncrease(String email, HttpServletRequest request) {
    String key = EMAIL_COUNT_KEY_PREFIX + email;
    // 예: email:verify:count:tiger@abc.com

    try {
        String current = redisService.getData(key);
        int count = (current == null) ? 0 : Integer.parseInt(current);

        if (count >= MAX_REQUESTS_PER_HOUR) {
            return true;
        }

        int newCount = count + 1;
        long oneHourSeconds = 60 * 60;
        redisService.setDataExpire(key, String.valueOf(newCount), oneHourSeconds);
        return false;

    } catch (Exception e) {
        // Redis 장애 시 세션으로 대체
        HttpSession session = request.getSession(true);
        String sessionKey = "emailCodeCount:" + email;
        Object value = session.getAttribute(sessionKey);

        int count = 0;
        if (value instanceof Integer) {
            count = (Integer) value;
        }

        if (count >= MAX_REQUESTS_PER_HOUR) {
            return true;
        }

        session.setAttribute(sessionKey, count + 1);
        return false;
    }
}
```

요약:

-   `MAX_REQUESTS_PER_HOUR = 5`
-   카운트 저장:

    -   Redis: `email:verify:count:{email}`, TTL = 1시간
    -   Redis 실패 시: Session `"emailCodeCount:{email}"` 사용

-   카운트가 5 이상이면 `true` 반환 → 호출부에서 `IllegalStateException` 발생

---

## 7. 이메일 상태 관리(REQUESTED / SENT / FAILED)

> ※ 상태 값은 현재 클라이언트에서 사용되지 않으며,
> 향후 "전송 중/완료/실패" UI에 사용 가능하다.

### 7.1 상태 키 및 TTL

```java
private static final String EMAIL_STATUS_KEY_PREFIX = "email:verify:status:";
private static final long EMAIL_STATUS_TTL_SECONDS = 60 * 10; // 10분
```

-   상태 값:

    -   `REQUESTED` : 코드 발송 요청 직후
    -   `SENT` : 비동기 메일 전송 성공
    -   `FAILED` :

        -   메일 발송 실패
        -   또는 요청 제한 초과·기타 예외로 인해 전송 처리가 정상적으로 완료되지 않은 경우

    -   `NONE` : 상태 정보 없음(요청 전 또는 TTL 만료 이후)

### 7.2 상태 조회 API

```java
public ResponseEntity<?> getEmailStatus(String email) {
    String statusKey = EMAIL_STATUS_KEY_PREFIX + email;

    try {
        String status = redisService.getData(statusKey);

        if (status == null) {
            return ResponseEntity.ok("NONE");
        }
        return ResponseEntity.ok(status); // REQUESTED / SENT / FAILED

    } catch (Exception e) {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                .body("이메일 상태를 조회할 수 없습니다.");
    }
}
```

특징:

-   상태 정보가 없으면 `"NONE"` 반환
-   Redis 장애 시 503(Service Unavailable) 응답

> 현재 컨트롤러에 이 메서드를 매핑한 API는 없음.
> 추후 필요 시 `/api/auth/email/status`와 같은 엔드포인트로 노출 가능하다.

### 7.3 비동기 메일 전송과 상태 값 갱신

```java
@Async
public void sendEmailAsync(String email, MimeMessage message) {
    String statusKey = EMAIL_STATUS_KEY_PREFIX + email;

    try {
        javaMailSender.send(message);
        // 전송 성공 => SENT
        redisService.setDataExpire(statusKey, "SENT", EMAIL_STATUS_TTL_SECONDS);
    } catch (Exception e) {
        // 전송 실패 => FAILED
        redisService.setDataExpire(statusKey, "FAILED", EMAIL_STATUS_TTL_SECONDS);
    }
}
```

`EmailService.sendMailWithCode()`에서는 요청 직후 `"REQUESTED"`로 설정하고,
실제 메일 전송 성공/실패에 따라 `AsyncMailService`가 `"SENT"` 또는 `"FAILED"`로 상태를 변경한다.

---

## 8. 코드 검증 방식(소프트 체크 / 하드 체크)

### 8.1 코드 조회

```java
private String findCode(String email, HttpServletRequest request) {
    try {
        Optional<Email> _email = emailRepository.findById(email);
        if (_email.isPresent()) {
            return _email.get().getCode();
        }
    } catch (Exception e) {
        // Redis 장애 => 세션에서 처리
    }

    HttpSession session = request.getSession(false);
    if (session == null) return null;

    Object sessionCode = session.getAttribute("emailCode:" + email);
    if (sessionCode == null) return null;

    return sessionCode.toString();
}
```

### 8.2 코드 삭제

```java
private void deleteCode(String email, HttpServletRequest request) {
    try {
        emailRepository.deleteById(email);
    } catch (Exception e) {
        // 에러 무시
    }

    HttpSession session = request.getSession(false);
    if (session != null) {
        session.removeAttribute("emailCode:" + email);
    }
}
```

### 8.3 소프트 체크 (코드 유지) – `/api/auth/check_code`

```java
public boolean checkEmailCodeOnly(String email, String code, HttpServletRequest request) {
    String savedCode = findCode(email, request);
    if (savedCode == null) return false;
    return savedCode.equals(code);
}
```

-   용도:

    -   화면에서 “코드 확인” 버튼으로 미리 검사할 때
    -   코드가 맞더라도 삭제하지 않는다.

> 컨트롤러에서는 다음과 같이 사용한다:
>
> ```java
> @PostMapping("/check_code")
> public ResponseEntity<?> checkCode(@RequestBody Map<String, String> body, HttpServletRequest request) {
>     String email = body.get("email");
>     String code  = body.get("code");
>     ...
>     boolean isValid = emailService.checkEmailCodeOnly(email, code, request);
>     ...
> }
> ```

### 8.4 하드 체크 (코드 삭제) – `/api/auth/signup` 내부

```java
public boolean isEmailCode(String email, String code, HttpServletRequest request) {
    if (!checkEmailCodeOnly(email, code, request)) {
        return false;
    }
    deleteCode(email, request);
    return true;
}
```

-   용도:

    -   실제 회원가입 완료 시점 등,
        코드가 한 번 사용되면 다시 사용할 수 없어야 할 때

-   코드가 유효하면 삭제 후 `true` 반환

> `AuthController.signup`에서는 다음과 같이 사용한다:
>
> ```java
> boolean isEmailCode = emailService.isEmailCode(email, code, request);
> if (!isEmailCode) {
>     return ResponseEntity
>         .status(HttpStatus.UNPROCESSABLE_ENTITY)
>         .body("이메일 인증코드가 틀렸거나 만료되었습니다.");
> }
> ```

---

## 9. 메일 본문 템플릿

`createMail(email, code)`에서 HTML 템플릿을 사용하여 인증 코드 메일을 생성한다.

특징:

-   제목: `"MindMate 이메일 인증 코드 안내"`
-   본문: MindMate 브랜드 스타일 적용된 HTML
-   코드가 강조된 박스에 삽입되어 사용자에게 안내

핵심 구조:

```java
String body = """
    ... (중략) ...
        <td align="center" ...>
            %s
        </td>
    ... (중략) ...
    """.formatted(code);
```

-   `%s` 위치에 6자리 코드가 삽입된다.
-   컨텐츠 타입은 `"text/html; charset=UTF-8"`로 설정한다.

---

## 10. 장애 및 Fallback 전략

| 상황                             | 처리 방식                                              |
| -------------------------------- | ------------------------------------------------------ |
| Redis 장애 (코드 저장)           | HttpSession에 코드 저장                                |
| Redis 장애 (카운트)              | HttpSession에 `"emailCodeCount:{email}"` 사용          |
| Redis 장애 (상태 조회)           | 503 응답 + `"이메일 상태를 조회할 수 없습니다."`       |
| 메일 발송 실패·요청 제한 실패 등 | 상태를 FAILED로 설정, UI에는 별도 알림 없음(현재 기준) |

-   Redis 사용이 불가능한 경우에도,
    최소한의 기능(코드 생성·검증, 요청 제한)은 Session으로 유지한다.
-   FAILED 상태는
    **“순전히 메일 전송 실패”뿐만 아니라,
    요청 제한 초과 등으로 인해 전송이 정상 완료되지 않은 경우까지 포함**할 수 있다.

---

## 11. 정리

이메일 인증 코드 시스템의 특징:

-   6자리 숫자 코드 기반 간단한 인증 구조
-   Redis + Session Fallback을 이용한 **탄력적인 저장/제한 구조**
-   마지막 요청 기준 **1시간 5회 제한**으로 과도한 인증 요청 방지
-   비동기 메일 발송과 상태 키(`REQUESTED`, `SENT`, `FAILED`)를 통한
    향후 UX 개선 가능성(“전송 중/성공/실패” 표시)
-   소프트 체크(`/check_code`)와
    하드 체크(`/signup` 내부 `isEmailCode`)로
    “검증만 필요한 경우”와 “실제 가입 시 1회용으로 소모해야 하는 경우”를 분리

회원가입 문서에서는 이 인증 시스템을
**“이메일 중복 확인 + 코드 검증(선행 조건)”**으로 간단히 참조하고,
실제 가입 흐름에서는
`/check_username → (/check_code 선택) → /signup` 순서를 사용한다.

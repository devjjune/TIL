# 📌 프로젝트 리팩토링 및 고도화

- SecurityConfig 인가 로직 (Authorization)
- 예외 처리 로직
- JWT 인증 로직 (Authentication)


<br>
<br>

# 🗓️ 2026-08-03 (월)

## 🧩 `SecurityConfig`의 `authorizeHttpRequests` 인가 규칙 리팩토링
>📌 공식 문서: [Authorize HttpServletRequests](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html)  
Spring Security > Servlet Applications > Authorization(인가) > Authorize HTTP Requests

`HttpSecurity`의 `authorizeHttpRequests`는 요청 경로별로 인가 규칙을 설정하는 메서드로, **위에서부터 순서대로 첫 번째로 매치되는 규칙만 적용된다** (first-match-wins).
따라서 좁은 범위의 규칙이 넓은 범위의 규칙보다 항상 먼저 와야, 좁은 규칙이 의도대로 동작한다.

- 기존에는 인가 규칙이 특별한 순서 없이 나열되어 있어 가독성이 떨어지고,
  넓은 범위 규칙이 실수로 위에 추가되면 좁은 범위 규칙이 무시될 위험이 있었음
- `authenticated`, `hasRole` 규칙을 `permitAll` 규칙보다 항상 위에 배치하여, 이후 `permitAll` 규칙이 추가되어도 기존 인증/권한 규칙을 침범하지 못하도록 **일일이 비교하는 방식이 아니라 구조적으로 순서를 강제**함
- 다른 규칙에 이미 포함되어 동작상 의미 없는 중복 규칙(죽은 코드) 제거
- ⚠️ Note: `/api/campings/**` permitAll이 GET 외 메서드까지 열려있는 문제는 서비스 로직 확인 후 별도 처리 예정

<br>
<br>

# 🗓️ 2026-08-04 (화)

## 🧩 SecurityConfig 401/403 응답을 ErrorCode/ApiResponse 기반으로 리팩토링

기존에는 `authenticationEntryPoint`, `accessDeniedHandler`에서 JSON 응답을 문자열 리터럴로 직접 조립하고 있었음.

**문제점**
- 상태코드/메시지만 다르고 구조가 동일한 코드가 두 번 중복
- JSON을 손으로 이스케이프 처리해야 해서, 메시지에 특수문자가 들어가면 깨질 위험
- 프로젝트에 이미 `ErrorCode`(enum) + `ApiResponse` + `GlobalExceptionHandler`로 구성된 전역 에러 응답 체계가 있는데, `SecurityConfig`만 이 체계를 안 쓰고 별도로 메시지 문자열을 하드코딩하고 있었음
  → 컨트롤러 예외 응답과 Security 필터 예외 응답의 포맷이 실질적으로는 같은데 서로 다른 경로로 만들어지던 상태

**리팩토링 내용**
- `writeErrorResponse(HttpServletResponse, ErrorCode)` 공통 메서드 추출
- `ObjectMapper`로 `ApiResponse` 객체를 직접 JSON 직렬화 → 문자열 수작업 제거
- 상태코드/메시지를 하드코딩하지 않고 기존 `ErrorCode`(`ACCESS_TOKEN_MISSING`, `FORBIDDEN`)를 그대로 참조
  → `GlobalExceptionHandler`의 `handleCustomException`과 동일한 방식(`errorCode.name()` + `errorCode.getMessage()`)으로 응답 조립

**배운 점**
- `SecurityConfig`의 `authenticationEntryPoint`/`accessDeniedHandler`는 서블릿 필터 체인 단계에서 실행되기 때문에, `DispatcherServlet` 이후에나 쓸 수 있는 `ResponseEntity`를 못 쓰고 `HttpServletResponse`를 직접 다뤄야 함
- `AccessDeniedException`을 처리하는 지점이 두 곳(`SecurityConfig`의 필터 레벨 vs `GlobalExceptionHandler`의 `@PreAuthorize` 등 MVC 레벨)이라 둘 다 필요하지만, `ErrorCode`를 공통으로 참조하게 만들면서 응답 포맷은 완전히 통일됨

## 🧩 CORS 이해하기 (개념 · 동작 원리 · 인가와의 차이)
### 1. CORS란 무엇이고 왜 필요한가
- **CORS**: 서로 다른 출처 간 리소스 공유를 허용하거나 제한하는 정책
- **출처(origin)**: 프로토콜 + 도메인 + 포트
- 출처가 다른 예시
  - (로컬) 프론트엔드: `localhost:3000` / 백엔드: `localhost:8080`
  - (배포) 프론트엔드: `https://camping-gajat.com` / 백엔드: `https://api.camping-gajat.com`
- 브라우저는 이렇게 도메인/포트가 다르면 서로 다른 출처로 간주하고, 스크립트가 다른 출처의 리소스에 접근하는 것을 **기본적으로 제한**함 → 이 **제한을 완화**하기 위한 정책이 CORS

### 2. CORS 헤더 동작 원리와 CSRF 방어 흐름
- 브라우저가 요청을 보낼 때, **요청을 보낸 페이지의 출처**를 `Origin` 요청 헤더에 자동으로 담아 전송함 (개발자가 설정하는 게 아니라 브라우저가 강제로 붙임)
- 서버는 응답을 만들 때, 이 `Origin` 값을 `allowedOrigins` 목록과 비교해서 응답 헤더에 `Access-Control-Allow-Origin`을 붙일지 결정함
- 최종적으로 이 헤더를 보고 응답을 스크립트에 넘길지 차단할지 판단하는 주체는 **브라우저**임 (서버는 요청 자체는 이미 처리함)

**악성 사이트(evil-site.com)가 요청을 보내는 경우:**
1. evil-site.com의 JS가 `fetch(백엔드 URL)` 실행
2. 브라우저가 요청 전송 시 `Origin: https://evil-site.com` 헤더를 자동으로 붙임
3. 서버(`SecurityConfig`)가 이 `Origin` 값을 `allowedOrigins`와 비교 → 목록에 없음
4. 서버는 `Access-Control-Allow-Origin` 헤더를 안 붙이거나 다른 값으로 응답
5. 브라우저가 응답에서 자신이 기대한 허가 헤더(evil-site.com)를 못 찾음 → JS 코드에 응답 내용 전달 차단

**정상 출처(camping-gajat.com)에서 요청을 보내는 경우:**
- 3번에서 `Origin`이 `allowedOrigins`와 일치 → 서버가 `Access-Control-Allow-Origin: https://camping-gajat.com`을 붙여 응답 → 브라우저가 정상적으로 JS에 응답 전달

### 3. CORS vs authorizeHttpRequests
`authorizeHttpRequests`는 서버가 이 요청을 애초에 처리해도 되는지 판단하는 것이고, CORS는 브라우저가 응답을 자바스크립트에게 전달해도 되는지 판단하는 것이다.   
`authorizeHttpRequests`는 요청을 보낸 사용자가 해당 자원에 접근할 권한이 있는지 서버가 판단하고, 요청이 들어온 직후 이를 검사하고 거부하면 401/403 에러를 통해 요청 자체를 처리하지 않는다.  
CORS는 요청을 보낸 출처(origin)가 안전한지를 판단하는 것이고, 서버로부터 응답을 받은 뒤 자바스크립트에 넘기기 전에 브라우저가 확인한다.   
`authorizeHttpRequests`가 잘못된 사용자로부터 우리 서비스를 보호하는 것이라면, CORS는 잘못된 요청(악성 사이트)으로부터 사용자를 보호하는 것이다.  
또한 curl이나 Postman처럼 브라우저를 거치지 않는 요청에는 CORS 검증 자체가 적용되지 않으므로, 서버 자원을 지키는 실질적인 방어는 `authorizeHttpRequests`(및 인증)가 맡는다.

⚠️ Note: 별도의 `http.cors(...)` 설정이 없음   
(`.cors(cors -> cors.configurationSource(corsConfigurationSource()))`를 securityFilterChain()에 추가)

<br>
<br>

# 🗓️ 2026-08-05 (수) ~ 2026-08-06 (목)

## 🧩 Throwable 계층구조 - 에러, 예외

**Throwable 클래스**: Java에서 `throw`로 던지고 `catch`로 잡을 수 있는 모든 것의 최상위 클래스   
(Throwable과 그 하위 클래스는 모두 **런타임에 발생**하는 문제를 표현. 컴파일 오류는 여기서 논외고 애초에 throw-catch 대상도 아님)

### 1. Error
: JVM 레벨의 심각한 문제. **애플리케이션 코드로 복구를 시도하지 않는 게 원칙**
- `OutOfMemoryError`: 힙 메모리 부족
- `StackOverflowError`: 재귀 호출 등으로 스택 깊이 초과
- `NoClassDefFoundError`: 컴파일 땐 있었는데 런타임에 클래스 못 찾음

### 2. Exception
: 프로그램 로직에서 발생하고, **처리 가능한** 문제

**Checked Exception**: 컴파일러가 `try-catch` 또는 `throws` 처리를 강제, 코드가 맞아도 외부 요인으로 발생 가능해서 실행 중 처리해야 함 (재시도, 대체 로직 등)
- `IOException`: 입출력 작업 중 문제
- `FileNotFoundException`: 파일을 찾을 수 없음 (IOException 하위)
- `SQLException`: DB 작업 중 문제
- `ClassNotFoundException`: `Class.forName()` 등으로 클래스 로드 실패

**Unchecked Exception (RuntimeException 계열)**: 컴파일러가 강제 안 함, 코드 자체의 버그이므로 애초에 버그가 나지 않도록 작성해야 함  
- `NullPointerException`: null 객체 참조 시도
- `ArrayIndexOutOfBoundsException`: 배열 범위 벗어난 접근
- `IllegalArgumentException`: 메서드에 부적절한 인자 전달
- `ClassCastException`: 잘못된 타입 캐스팅
- `ArithmeticException`: 0으로 나누기 등 산술 오류
- `NumberFormatException`: 문자열→숫자 변환 실패 (IllegalArgumentException 하위)

```
Throwable
 ├── Error (복구 시도 X, 잡지 않는 게 관례)
 │    ex) OutOfMemoryError, StackOverflowError
 │
 └── Exception (복구 가능, 처리 대상)
      ├── Checked Exception (컴파일러가 처리 강제)
      │    ex) IOException, FileNotFoundException, SQLException
      │
      └── RuntimeException = Unchecked Exception (강제 안 함)
           ex) NullPointerException, IllegalArgumentException, ClassCastException
```

Error는 "복구할 수 있냐 없냐"로 판단, Checked/Unchecked는 "컴파일러가 처리 여부를 검사하냐 안 하냐"로 판단 

## 🧩 캠핑가잣 예외 처리 클래스 검토 및 리팩토링
현재 우리 프로젝트의 예외 처리 클래스는 3개로 이루어져 있다.
- **`ErrorCode`(enum)**: 에러의 종류를 정의. HTTP 상태코드 + 메시지를 세트로 관리.
- **`CustomException` (`extends RuntimeException`)**: ErrorCode를 담아서 런타임 예외를 던진다.
  - `super(errorCode.getMessage())`로 Throwable에 메시지 저장
  - `this.errorCode = errorCode`로 자기 필드에도 보관
  - unchecked 예외인 RuntimeException을 상속받으므로 컴파일러 강제X
- **`GlobalExceptionHandler` (`@RestControllerAdvice`)**: 던져진 예외를 최종적으로 잡아서 HTTP 응답으로 변환

### 설계 포인트 💡
- 서비스 로직에서 발생하는 예외 케이스를 개별 클래스로 분리하지 않고, `ErrorCode` Enum에 담아둔 뒤 `CustomException` 하나가 이를 꺼내 던지는 구조로 설계했다.
- `CustomException`이 Unchecked 예외인 `RuntimeException`을 상속하도록 하여, 모든 계층에 `throws`를 전파하지 않아도 되게 함으로써 보일러플레이트를 줄였다.
- `GlobalExceptionHandler`에서 `@ExceptionHandler`로 `CustomException`과 Spring 프레임워크가 던지는 예외들(`MethodArgumentNotValidException`, `AccessDeniedException` 등)을 각각 잡아 처리하고, 어디에도 안 걸리는 나머지 예외는 `Exception.class` 핸들러가 최종적으로 한 번에 잡도록 했다.

### GlobalExceptionHandler 리팩토링

- `ResponseEntity.status(...).body(new ApiResponse<>(...))` 반복되던 패턴 → `buildErrorResponse()` 헬퍼 2개(오버로드)로 통합
- `@Slf4j` 추가, 로그 레벨 구분해서 적용
  - `handleException` (catch-all) → `log.error` (필수: 예상 못한 버그 추적용)
  - `handleCustomException`, `handleAccessDeniedException` → `log.warn` (빈도 파악/비정상 접근 탐지용)
  - 검증 관련 핸들러는 사용자 실수 성격이라 로깅 생략
- `handleException(Exception.class)`을 맨 마지막으로 이동 (관례: 구체적 타입 → 포괄적 타입 순서)
- `ResponseEntity<ApiResponse>` → `ResponseEntity<ApiResponse<?>>` (raw type 경고 제거)

<br>
<br>

# 🗓️ 2026-08-06 (목) ~ 2026-08-07 (금)
## 🧩 캠핑가잣 JWT 인증 로직 검토
### 1. JwtFilter 클래스
- `JwtFilter`는 스프링 시큐리티의 필터 체인에 끼워 넣는 필터 중 하나다.
- 필터로 동작하기 위해, 관례적으로 `OncePerRequestFilter`을 상속받는다.
  - 한 요청당 한 번만 실행 보장 - by. final 메서드 `doFilter()`
  - (DB 조회, SecurityContext 설정이 중복 실행되면 안 됨)
- 상속받으면 OncePerRequestFilter 추상 클래스에 정의되어 있는 `doFilterInternal()`이라는 추상 메서드를 오버라이드한다.

#### doFilterInternal 메서드 흐름
1. `/api/auth` 경로 → 무조건 통과 (인증 로직 자체를 안 탐)
   - 로그인된 상태라 토큰이 있지만 인증 로직 거칠 필요 없을 때 (logout, refresh) 
2. 쿠키에 토큰 없음 (signup, login) → 통과 (비로그인 사용자, 뒤의 authorizeHttpRequests가 판단)
3. 토큰 있음 → 검증
   - 검증 실패(만료 등) → 예외 발생 → catch에서 GlobalExceptionHandler로 위임
   - 검증 성공 → userId/role 추출 → DB 조회(탈퇴 확인) → Authentication 생성 (인증된 사용자 정보 + 권한 목록 담음) → SecurityContext에 저장
4. 어느 경우든 마지막엔 filterChain.doFilter(...)로 다음 필터로 진행 (예외 안 났다면)

### 2. JwtProvider 클래스

#### 1) 생성자 — SecretKey 준비
- `@Value`로 `jwt.secret`, 만료 시간(access/refresh) 주입받음
- `secret` 문자열 → UTF-8 바이트 배열 → `Keys.hmacShaKeyFor(...)`로 `SecretKey` 생성 (키 길이에 따라 HS256/384/512 자동 결정, 너무 짧으면 `WeakKeyException`으로 앱 시작 자체 실패)
- 매 요청마다 변환하지 않도록 생성자에서 한 번만 만들어 필드로 재사용

#### 2) 토큰 생성 — createAccessToken / createRefreshToken
- `Jwts.builder()`로 클레임 조립: `subject`(userId), `claim("role", ...)`(커스텀 클레임), `issuedAt`, `expiration`
- `signWith(key)`로 서명 → `compact()`로 `xxx.yyy.zzz` 문자열 직렬화
- Access/Refresh 구조는 동일, 만료 기간만 다름 (accessExpiration vs refreshExpiration)

#### 3) 토큰에서 값 추출 — getRole / getUserId
- `Jwts.parser().verifyWith(key).build().parseSignedClaims(token)`으로 파싱 + 서명 검증
- `getPayload()`로 Claims 꺼낸 뒤 `getRole`은 커스텀 클레임 "role", `getUserId`는 `getSubject()`(→ Long 변환)
- ⚠️ 각각 독립적으로 파싱을 수행 → 같은 토큰에 대해 파싱이 중복됨

#### 4) 검증 — validateToken / validateRefreshToken
- 둘 다 동일한 파싱 로직으로 서명/만료 검증만 수행 (결과값 Claims는 버림)
- `validateToken`: 만료 → `CustomException(ACCESS_TOKEN_EXPIRED)` 던짐 / 그 외 실패(서명 불일치, 형식 오류, null) → `false` 리턴 (예외와 boolean이 혼재된 이중적 신호 방식)
- `validateRefreshToken`: 만료 → `REFRESH_TOKEN_EXPIRED`, 그 외 → `REFRESH_TOKEN_INVALID` (실패를 항상 예외로 통일 — validateToken과 대조됨)

<br>
<br>

# 🗓️ 2026-08-10 (월)
## 🧩 JWT 토큰 구조 이해하기 (헤더·페이로드·서명, 파싱)
### 1. JWT 토큰의 구조
JWT는 점(.)으로 구분된 세 부분으로 이루어진 문자열이다. → `[헤더].[페이로드(=클레임)].[서명]`

- **헤더 (JSON)**: 어떤 알고리즘으로 서명됐는지, 타입이 뭔지에 대한 메타데이터. 실제 사용자 정보와는 무관.
- **페이로드 (JSON) = 클레임**: `sub`, `role`, `exp` 같은 key-value 쌍. 이 각각을 "클레임"이라 부른다. 인코딩만 되어 있을 뿐 암호화는 아니라서 누구나 디코딩해서 내용을 읽을 수 있다 — 그래서 민감정보는 절대 담으면 안 된다.
- **서명**: 헤더 + 페이로드를 시크릿 키로 계산한 결과값. 토큰의 위변조 여부와 발급 주체를 증명하는 용도이지, 내용을 숨기는 용도가 아니다.

### 2. 시크릿 키 vs 서명
- **시크릿 키**: 서버만 알고 있으며 서명을 만들고 검증하는 데 쓰임. 토큰에는 포함되지 않고, 클라이언트로 절대 전달되지 않는다.
- **서명**: 그 키로 헤더+페이로드를 계산해서 나온 결과값. 토큰의 세 번째 부분에 실제로 포함되어 클라이언트까지 전달된다.
- 서명값만 보고 시크릿 키를 역산하는 것은 불가능(단방향 연산)하므로, 토큰이 노출돼도 시크릿 키 자체는 안전하다.

### 3. 파싱이란?
문자열로 된 토큰을 구조화된 객체로 변환하는 작업. 헤더/페이로드/서명으로 분해해서 이해 가능한 형태(JSON → 자바 객체)로 바꾼다.  
파싱 결과는 `Claims` 객체에 담기며, 이 JSON(페이로드) 전체를 Map처럼 다룰 수 있게 감싼 자바 객체다. `claims.getSubject()`는 `sub` 클레임을, `claims.get("role", String.class)`는 `role` 클레임을 꺼내는 식.

### 4. 파싱이 "서명 검증"까지 같이 하는 이유
JWT 파싱은 Base64 디코딩만 하는 게 아니라, 서명 검증까지 같이 수행한다.

1. 문자열을 세 부분으로 분해
2. 헤더+페이로드를 갖고 서버가 가진 시크릿 키로 서명을 다시 계산
3. 새로 계산한 서명과 토큰에 원래 붙어있던 서명을 비교
4. 일치하면 "발급 주체가 맞고 조작 안 됐다" → Claims 반환
5. 불일치하면 예외 발생 (`JwtException`)

```
JWT 문자열 (xxx.yyy.zzz)
    ↓ 파싱 (문자열 → 구조화된 데이터로 변환 + 서명 검증)
Claims 객체 (= 클레임들의 모음, {sub, role, exp, ...})
    ↓ 개별 클레임 꺼내기
userId, role 등
```

### 5. 여러 번 파싱하면 안 좋은 이유
서명 검증은 암호 연산이 포함돼 상대적으로 비용이 드는 작업. 같은 토큰에 대해 `validateToken`, `getUserId`, `getRole`을 각각 호출하면 동일한 서명 재계산+비교를 매번 반복하게 됨.  
→ 서비스가 마비될 정도의 성능 문제는 아니지만, 트래픽이 많아질수록 낭비가 누적되고 무엇보다 "같은 검증을 여러 번 한다"는 설계 자체가 비효율적. `parseClaims()`로 통합해 한 번만 파싱하고 결과(Claims)를 재사용하도록 리팩토링.

<br>
<br>

# 🗓️ 2026-08-12 (수)
## 🧩 JwtProvider 리팩토링: 파싱 로직 통합 및 토큰 생성 메서드 정리
기존에는 validateToken/getUserId/getRole이 각각 독립적으로 토큰을
파싱(서명 검증 포함)하고 있어, 같은 토큰에 대해 파싱이 3회 반복됨.
또한 validateToken은 만료 시 예외, 그 외 실패 시 false를 리턴하는
이중적인 실패 신호 방식을 사용하고 있었음.

- validateToken(String) 제거, parseClaims(String)으로 대체
  - 파싱 결과(Claims)를 그대로 반환하여 호출부에서 재사용 가능하게 함
  - 실패 시 항상 예외를 던지도록 통일 (ACCESS_TOKEN_EXPIRED / INVALID_TOKEN)
- getUserId, getRole이 String token 대신 Claims를 받도록 변경
  → 파싱은 parseClaims에서 한 번만 수행, 이후 결과를 재사용
- createAccessToken/createRefreshToken을 createToken(userId, role, expirationMs)
  헬퍼로 통합 (만료 시간 값만 다르고 나머지 로직 동일)
- JwtFilter도 parseClaims 기반으로 함께 수정 (if(validateToken) 분기 제거,
  검증 실패는 catch에서 일괄 처리)

Note: validateToken이 false를 리턴하던 케이스(서명 불일치 등)는 기존에
authorizeHttpRequests가 처리해 ACCESS_TOKEN_MISSING으로 응답했으나,
이제 parseClaims가 즉시 INVALID_TOKEN 예외를 던져 응답 코드가 달라짐
(둘 다 401이지만 에러 메시지 값 변경 - 프론트 영향 확인 필요)

## 🧩 AOP 개념 정리 — self-invocation 버그의 원인
애플리케이션 로직은 크게 **핵심 로직**과 **부가 로직**으로 나뉜다. 
- **핵심 로직**: 객체가 제공하는 기능 (비즈니스 로직)
- **부가 로직**: 핵심 기능을 보조하는 기능으로, 단독으로 쓰이지 않고 핵심 기능과 함께 쓰인다. (예: 트랜잭션 관리, 로그 추적, 권한 체크 등)

(본래 부가 로직도 개발자가 직접 작성해야 하지만, AOP를 사용하면 Spring이 프록시를 통해 자동으로 앞뒤에 끼워 넣어주므로 개발자는 핵심 로직만 작성하면 됨)

부가 기능은 일반적으로 **횡단 관심사(cross-cutting concern)** 다. 즉, 하나의 클래스에만 속하지 않고 여러 클래스에 걸쳐 동일하게 반복된다.  

따라서 같은 부가 기능을 여러 곳에 적용하면 반복되는 로직이 많아지고 부가 기능 로직이나 적용 대상을 변경할 때 수정이 번거롭다. 

이러한 부가 기능 도입의 문제를 해결하기 위해 **AOP(Aspect-Oriented Programming, 관점 지향 프로그래밍)** 라는 개념이 도입되었다.  
부가 기능을 핵심 기능으로부터 **분리**해서 한 곳에서 관리하고, 어디에 적용할지도
선언적으로 지정할 수 있게 한다. OOP를 대체하는 게 아니라, OOP를 보완·확장하는 방법이다.  

| 용어 | 정의 | ReservationService 예시 |
|---|---|---|
| **Aspect** | 부가 기능과, 그 부가 기능을 어디에 적용할지를 함께 정의한 모듈 | 트랜잭션 관리 자체가 대표적인 aspect |
| **Join point** | 프로그램 실행 중 advice가 적용될 수 있는 지점. Spring AOP에서는 항상 **메서드 실행 시점**을 의미 | `cancelReservation()`이 호출되는 순간 |
| **Advice** | 특정 join point에서 aspect가 실제로 수행하는 동작. Before/After/Around 등 여러 종류가 있음 | `@Transactional`이 트랜잭션을 열고 닫는 동작 |
| **Pointcut** | 어떤 join point에 advice를 적용할지 걸러내는 규칙(predicate) | "`@Transactional`이 붙은 모든 public 메서드" |
| **Target object** | advice가 적용되는 대상 객체. Spring AOP는 런타임 프록시 방식이므로, target object는 항상 **프록시로 감싸진 객체** | `reservationService` 빈의 원본 인스턴스 |
| **AOP proxy** | aspect의 동작(advice 실행)을 실제로 구현하기 위해 프레임워크가 만드는 객체. Spring에서는 **JDK 동적 프록시** 또는 **CGLIB 프록시** | Spring이 `ReservationService`를 감싸서 만든 프록시 객체 |
| **Weaving** | aspect를 실제 대상 객체에 연결해서 advised object(프록시 적용된 객체)를 만드는 과정. 컴파일 타임/로드 타임/런타임에 할 수 있는데, **Spring AOP는 런타임에 weaving** | Spring 컨테이너가 뜰 때 `ReservationService` 빈을 CGLIB 프록시로 감싸는 과정 |


### 왜 이게 중요한가 (Spring AOP의 특징)
Spring AOP는 **프록시 기반**으로 동작한다. 즉, advice가 실행되려면 반드시
**프록시 객체를 거쳐서** 메서드가 호출돼야 한다.

→ 이 특징 때문에, 클래스 내부에서 `this.method()`처럼 자기 자신의 메서드를 직접 호출하면
프록시를 거치지 않고 원본 객체가 바로 실행되어 **advice(예: `@Transactional`)가 적용되지 않는
self-invocation 문제**가 생긴다. (`cancelReservation()` 내부에서
`cancelPaymentOutsideTransaction(payment)`를 직접 호출한 게 바로 이 케이스)

<br>
<br>

# 🗓️ 2026-08-13 (목)
## 🧩 AOP 프록시 교체 시점과 Pointcut의 역할
### 1. 프록시는 언제, 어떻게 만들어질까?

빈 하나가 생성될 때마다 그 빈이 어떤 aspect의 pointcut에 매칭되는지 BeanPostProcessor 단계에서 확인한다.

- 매칭되면 → 원본 빈 대신 **프록시 객체**를 만들어 반환
- 이 프록시 객체가 실제로 **컨테이너에 등록**되는 빈이 됨 (원본은 프록시 내부에 감싸진 채로만 존재)
- 참고: 스프링에는 프록시 생성 이외에도 다른 목적을 가진 BeanPostProcessor들이 여러 개 있다 (예: `@Autowired` 필드를 채워주는 처리기 등)

### 2. Join point vs Pointcut

- **Join point**: advice(부가 기능)가 낄 수 있는 **후보 지점 전체**. Spring AOP에서는 모든 메서드 실행 하나하나가 join point
- **Pointcut**: 그 수많은 join point 중에서 **실제로 advice를 적용할 대상을 걸러내는 규칙**

### 3. `@Transactional`, `@Async`, `@Cacheable` = 미리 만들어진 (advice + pointcut) 세트

이 어노테이션들은 스프링이 이미 다음 두 가지를 세트로 구현해서 내장해둔 것이다.

- **Advice**: 실제로 실행할 부가 로직
- **Pointcut**: "이 어노테이션이 붙은 메서드/클래스"를 찾아내는 매칭 규칙

즉 **"부가 로직 + 적용 대상 규칙"** 이 이미 하나로 묶여서 제공되기 때문에,
개발자는 어노테이션만 붙이면 별도로 pointcut을 작성할 필요가 없다.

**예: `@Transactional`**
| 구성 요소 | 내용 |
|---|---|
| Advice (부가 로직) | 메서드 실행 전 트랜잭션 시작, 정상 종료 시 커밋, 예외 발생 시 롤백 |
| Pointcut (적용 대상 규칙) | `@Transactional`이 붙은 메서드(또는 클래스) |

<br>
<br>
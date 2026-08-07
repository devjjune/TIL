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
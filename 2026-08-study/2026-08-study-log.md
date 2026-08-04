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
- Note: `/api/campings/**` permitAll이 GET 외 메서드까지 열려있는 문제는 서비스 로직 확인 후 별도 처리 예정

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

<br>
<br>
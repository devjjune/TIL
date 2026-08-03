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
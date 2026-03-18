# 🗓️ 2026-03-01 (일)
## 스프링 블로그 프로젝트 ver2.0 진행 시작
### 🧩 Java 버전 업그레이드 및 Gradle과의 호환성 해결
블로그 기록: [[Gradle] Java 25 업데이트: sourceCompatibility 대신 Toolchain을 선택한 이유 (feat. Gradle 버전 에러, Gradle 공식 문서)](https://velog.io/@hjy648012/Gradle-sourceCompatibility-%EB%A7%90%EA%B3%A0-toolchain%EC%9D%84-%EC%8D%A8%EC%95%BC-%ED%95%98%EB%8A%94-%EC%9D%B4%EC%9C%A0-feat.-%EA%B3%B5%EC%8B%9D-%EB%AC%B8%EC%84%9C)

`build.gradle` 파일에서 자바와 스프링 버전을 상위 버전으로 업그레이드했다.  
자바 버전을 명시할 땐 **직접 Gradle 공식 문서를 찾아보며 toolchain 방식과 Source Compatibility 방식을 비교**했고, 그 결과 toolchain 방식을 택했다. 

인텔리제이 설정까지 마치고 build 파일을 새로고침했을 때, 오래된 Gradle 버전과 자바 25 간에 호환이 안 돼서 에러가 발생했다. Gradle 공식 홈페이지에서 **가장 최신 gradle 버전을 찾아 적용**해주니 에러를 해결할 수 있었다. 

이를 통해 Gradle 버전마다 지원하는 자바 버전의 상한선이 정해져 있으므로, 자바 버전을 올릴 때 Gradle wrapper 버전도 함께 체크해야 한다는 것을 알게 되었다. 

### 🧩 스프링부트 버전 업그레이드 및 시큐리티 에러 해결
블로그 기록: [[트러블슈팅] Spring Boot 4.0 업데이트: AntPathRequestMatcher 제거 대응 및 시큐리티 7.0 마이그레이션](https://velog.io/@hjy648012/%ED%8A%B8%EB%9F%AC%EB%B8%94%EC%8A%88%ED%8C%85-Spring-Boot-4.0-%EC%97%85%EB%8D%B0%EC%9D%B4%ED%8A%B8-AntPathRequestMatcher-%EC%A0%9C%EA%B1%B0-%EB%8C%80%EC%9D%91-%EB%B0%8F-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0-7.0-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98)

스프링부트 버전을 4.0으로 올리니 `AntPathRequestMatcher`를 찾을 수 없다는 에러가 발생했다.   
스프링부트를 업그레이드하며 스프링 시큐리티도 7.0으로 올라갔는데, 7.0에서는 해당 클래스가 제거된 것이 원인이었다. 아래와 같은 방식으로 에러 난 코드를 수정했다. 
- AntPathRequestMatcher 제거에 따른 PathPatternRequestMatcher 도입
- authorizeRequests를 최신 authorizeHttpRequests 문법으로 변경
- H2 콘솔 및 정적 리소스 설정 경로 직접 지정 방식으로 수정

또한 **스프링 공식 문서 (docs.spring.io)에서 시큐리티 버전과의 호환성이 명시된 부분**을 직접 찾아보며, 라이브러리 버전을 올릴 때 공식 문서나 마이그레이션 가이드를 참고해야 한다는 점을 배웠다.  

<br>
<br>

# 🗓️ 2026-03-02 (월)
## 스프링 프로젝트 패키지 구조 리팩토링 - 도메인 중심으로
### 🧩 레이어드를 품은 도메인 구조
소규모 스프링 프로젝트로 처음 시작할 땐 패키지를 계층형 구조로 구성했었는데,    
이후 기능 확장을 고려해서 상위 패키지를 도메인 기반으로 구성하고 각 내부를 다시 계층으로 나눈 '레이어드를 품은 도메인 구조'로 리팩토링했다. 

이전에 블로그에 관련 개념을 정리해서 작성한 글이 있었는데, 이번 프로젝트 리팩토링을 그 예시로 추가했다.   
→ [[자바 OOP 아키텍처] 패키지 구조 - 레이어드 vs 도메인보단 레이어드를 품은 도메인](https://velog.io/@hjy648012/%EC%9E%90%EB%B0%94-OOP-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98-%ED%8C%A8%ED%82%A4%EC%A7%80-%EA%B5%AC%EC%A1%B0-%EB%A0%88%EC%9D%B4%EC%96%B4-vs-%EB%8F%84%EB%A9%94%EC%9D%B8)

<br>
<br>

# 🗓️ 2026-03-03 (화)
## 🧩 배열은 참조로 전달되고, 기본형은 값으로 전달된다
> **[문법 요소]**  
>배열 정렬: `Arrays.sort(array);`  
>중앙값: `array[length / 2]`

`Arrays.sort(arr)`를 호출하면 **매개변수로 받은 배열 자체가 바뀌며** 따라서 따로 반환값을 받을 필요가 없다. (`sort`는 `void`)

```java
public void solution(int[] arr) {
    Arrays.sort(arr);
}
```

자바에서 **배열은 참조 타입**이기 때문에, 여기서 `arr`는 배열의 **주소(참조값)** 를 복사해서 받은 것이고 그 주소가 가리키는 **같은 배열을 공유**한다.   
따라서 내부에서 배열의 요소를 바꾸면 원본도 바뀐다.

```java
static void change(int x) {
    x = 100;
}
```

하지만 위와 같은 기본형의 경우에는 원본이 바뀌지 않는다. 왜냐하면 `int`는 기본형이라 값이 복사돼서 전달되기 때문이다. 

## 🧩 JPA Fetch 전략
JPA에서 연관관계를 조회할 때 데이터를 가져오는 시점 (Fetch) 
- EAGER : 즉시 로딩, 엔티티 조회 시 연관된 엔티티도 즉시 함께 조회
- LAZY : 지연 로딩, 연관 엔티티를 실제로 사용하는 시점에 조회 (실무에서 더 권장됨)
- 실무에서는 대부분 LAZY로 설정하고, 필요한 경우 fetch join으로 조회한다. (JPA에서 지연로딩(LAZY)을 유지하면서도, 특정 조회 시점에 연관 엔티티를 한 번에 같이 가져오는 방법)

<br>
<br>

# 🗓️ 2026-03-04 (수) ~ 2026-03-05 (목)
## 🧩 JwtFactory를 이용한 토큰 테스트 흐름

블로그 정리: [[Spring Security/JWT] JwtFactory를 이용한 토큰 테스트 흐름](https://velog.io/@hjy648012/JwtFactory-%ED%85%8C%EC%8A%A4%ED%8A%B8-%ED%9D%90%EB%A6%84)

스프링 프로젝트에서 자바와 스프링 버전을 업그레이드하고, 패키지 구조를 도메인 중심으로 개편하는 일련의 과정을 거쳤다.   
이제 Test 코드도 Main 패키지 구조와 동일하게 맞춰야 하는데, 이를 수정하며 작성했던 테스트 코드들을 뜯어보았다.   

가짜 사용자의 가짜 토큰을 생성하는 `JwtFactory (Test)`는 토큰과 관련된 인가 기능을 테스트할 때 매번 로그인 로직을 구현해야 하는 번거로움을 없애준다.   
또한 정상적인 토큰 뿐만 아니라 비정상적인 토큰도 인위적으로 만들어서 에러 상황을 테스트해 볼 수 있다.    
이 클래스는 실제 서비스 로직이 아니라 테스트 상황에서만 사용되는 클래스이기 때문에 Test 패키지 안에 작성해야 하며, 단위 테스트를 넘어 통합 테스트에서도 사용되므로 domain이 아닌 global 패키지에 넣었다. 

이후 본격적으로 테스트 코드를 살펴보았는데, `TokenProviderTest`에서는 `TokenProvider (Main)` 에서 정상적인 토큰을 제대로 발행하는지, 그리고 JwtFactory에서 발행한 비정상적인 토큰을 제대로 걸러내는지, 그리고 토큰 정보를 제대로 가져오는지 검사하는 로직이 포함되어 있다.    
`TokenProvider (Main)`와 `JwtFactory (Test)`는 둘 다 토큰을 생성하지만 전자는 실제 서비스에서 로그인을 성공했을 때 호출되어 정상적으로 실행되는 토큰을 , 후자는 개발자가 만드는 비정상 토큰을 생성한다는 점에서 차이가 있다.   

마지막으로 단위 테스트였던 TokenProviderTest을 거친 후, 리프레시 토큰과 토큰 필터 로직 등은 `TokenApiControllerTest (API 테스트)`, `BlogApiControllerTest (통합 테스트)` 등의 상위 테스트에서 검증한다.

<br>
<br>

# 🗓️ 2026-03-06 (금)
## 🧩 Spring Boot 4.0 환경에서의 토큰 테스트 코드 트러블슈팅
### 1. 문제 상황
- Java 25, Spring Boot 4.0으로 프로젝트 환경 업그레이드
- 이전에 작성했던 테스트 코드에 `ObjectMapper`, `@AutoConfigureMockMvc` 등 주요 클래스의 패키지 경로를 찾지 못하는 컴파일 에러 발생
- 컴파일 에러 해결 후 테스트 실행 시, OAuth2 및 Security 설정 미비로 인한 `UnsatisfiedDependencyException` 발생 및 테스트 실패

### 2. 원인 분석
- **라이브러리 메이저 업데이트**: Jackson 라이브러리가 업데이트되면서 패키지 경로가 `com.fasterxml.jackson`에서 `tools.jackson`으로 변경됨을 확인
- **스프링 부트 구조 재편**: 최신 버전에서는 테스트 자동 설정 패키지가 `webmvc.test.autoconfigure` 등으로 세분화되거나 변경되어 기존 `import` 문이 유효하지 않게 됨
- **보안 필터 간섭**: `@SpringBootTest` 실행 시 전체 애플리케이션 컨텍스트를 로드하면서, 테스트 환경에 설정되지 않은 OAuth2 클라이언트 정보로 인해 빈(Bean) 생성에 실패함

### 3. 해결을 위한 시도 및 학습 내용
- **라이브러리 직접 탐색**: IntelliJ의 `External Libraries`를 직접 뒤져 변경된 클래스 패키지 경로를 추적하는 법을 익힘
- **수동 주입(Manual Setup)의 활용**: 어노테이션 기반의 자동 설정(`@AutoConfigureMockMvc`)이 고장 났을 때, `WebApplicationContext`를 이용해 `MockMvc`를 직접 빌드하는 **Low-level 제어 방식**을 학습함
- **공식 문서 읽기**: Spring Framework와 Spring Boot 문서의 차이를 이해하고, 최신 API 명세서에서 `@interface` 및 패키지 주소를 확인하는 법을 체득함

### 4. 회고
- **Keep**: 에러 메시지의 `Caused by`를 끝까지 추적하여 문제의 본질(Security/OAuth2)을 찾아내려 노력한 점
- **Problem**: 환경 변화(버전 업)에 따른 패키지 경로 변경 가능성을 간과하고 기존 코드에만 의존하려 했던 점
- **Try**: 다음에는 새로운 기술 스택을 도입할 때 공식 문서의 **Release Notes**를 먼저 훑어보고, 환경 설정(Configuration) 문제를 격리하기 위해 최소 단위의 테스트부터 시작할 것

<br>
<br>

# 🗓️ 2026-03-07 (토)
## 김영한의 스프링 핵심 원리 - 기본편
### 🧩 싱글톤 컨테이너
**싱글톤 패턴**은 어떤 클래스의 인스턴스가 오직 1개만 생성되도록 보장하는 디자인 패턴으로, 메모리 낭비와 성능 저하를 방지해준다.   
스프링은 객체를 일일이 싱글톤 객체로 만들지 않고도 싱글톤으로 관리해주는 **싱글톤 컨테이너**의 역할을 수행한다. 싱글톤 컨테이너를 쓸 때 주의해야 할 점은 객체 안에 상태를 저장하지 않는 '무상태'를 유지해야 한다는 것이다.   

설정 클래스(`AppConfig`)에 **`@Configuration`** 어노테이션을 붙이면 CGLIB라는 바이트코드 조작 라이브러리를 사용해, 우리가 만든 클래스를 상속받은 가짜(프록시) 객체(`AppConfig@CGLIB`)를 만들고 이를 스프링 빈으로 등록한다.  
이 가짜 프록시 객체는 우리가 만든 메서드를 그대로 실행하지 않고, 내부적으로 아래와 같은 조건문 형식의 로직을 거치게 한다. 
```Java
1. @Bean 메서드가 호출될 때, 이미 스프링 컨테이너에 해당 객체가 있는지 확인
1. 있다면? 기존에 있던 객체를 반환
2. 없다면? 직접 생성해서 컨테이너에 등록하고 반환

// 스프링이 조작한 프록시 객체의 내부 로직 예시
if (이미 스프링 컨테이너에 'memberRepository'가 등록되어 있는가?) {
    return 컨테이너에서 기존 객체를 찾아서 반환;
} else {
    // 최초 1회만 실행
    return super.memberRepository(); // 실제 우리가 작성한 메서드 호출 후 컨테이너에 등록
}
```

### 🧩 컴포넌트 스캔과 의존관계 자동 주입
컴포넌트 스캔: 스프링이 제공하는 기능으로, 설정 정보에 일일이 스프링 빈을 등록하지 않아도 자동으로 등록할 수 있게 해준다. (`@ComponentScan`)  
- `@ComponentScan` 은 `@Component` 가 붙은 모든 클래스를 스프링 빈으로 등록한다. 이때 스프링 빈의 기본 이름은 클래스명을 사용하되 맨 앞글자만 소문자를 사용한다.

`@Autowired`를 통해 의존관계를 자동 주입해준다. 
- 생성자에 `@Autowired` 를 지정하면, 스프링 컨테이너가 자동으로 해당 스프링 빈을 찾아서 주입한다. 이때 기본 조회 전략은 타입이 같은 빈을 찾아서 주입한다.

<br>
<br>

# 🗓️ 2026-03-08 (일)
## 김영한의 스프링 MVC 1편 - 웹 애플리케이션 이해
### 🧩 웹 서버 & 웹 애플리케이션 서버
웹은 **HTTP**를 기반으로 이루어져 있고 거의 모든 형태의 데이터는 HTTP 메시지를 통해 전송될 수 있다. 

웹 서버와 웹 애플리케이션 서버(WAS) 또한 HTTP를 기반으로 동작한다. **웹 서버**는 주로 정적 리소스를 제공하고, **웹 애플리케이션 서버(WAS)** 는 프로그램 코드를 실행해서 애플리케이션 로직을 수행하지만 정적 리소스 또한 제공할 수 있다. 

따라서 WAS와 DB만으로도 시스템을 구성할 순 있지만, WAS에게 너무 많은 역할이 부여되면 서버에 과부하가 올 수도 있고, WAS 장애가 발생했을 때 사용자에게 오류 화면을 노출하는 것도 불가능해진다. 

따라서 보통 **정적 리소스는 웹 서버가 처리하고, 애플리케이션 로직 같은 동적 처리는 WAS가 담당**하는 식으로 구성된다. 

### 🧩 서블릿
서블릿은 자바 코드로 이루어진 웹 처리 컴포넌트로, **WAS의 핵심 기능인 동적 처리를 담당**하며 개발자가 직접 코딩해야 하는 복잡한 서버 로직을 상당 부분 대신 처리해준다. 

서블릿을 통해 개발자는 HTTP 스펙을 매우 편리하게 사용할 수 있다. HTTP 요청과 응답 정보를 각각 HttpServletRequest과 HttpServletResponse 객체에 넣어 전달할 수 있다.  

톰캣처럼 서블릿을 지원하는 WAS를 **서블릿 컨테이너**라고 한다. 서블릿 컨테이너는 서블릿 객체를 생성, 초기화, 호출, 종료하는 생명주기를 관리한다. 서블릿 객체는 **싱글톤**으로 관리되기 때문에 **공유 변수 사용에 주의**해야 한다. 서블릿 컨테이너의 가장 큰 특징은 **동시 요청을 위한 멀티 쓰레드 처리를 지원**한다는 것이다. 

### 🧩 멀티 스레드
스레드는 운영체제의 프로세스 내에서 실행되는 작은 작업 단위로, 애플리케이션 코드를 하나하나 순차적으로 실행한다. 

요청이 발생할 때마다 스레드를 생성하면 스레드를 생성하거나 컨텍스트 스위칭을 할 때 비용이 많이 발생하고 제한 없이 생성하다가 서버가 죽을 수도 있다. 

따라서 **스레드 풀 안에서 스레드를 보관하고 관리**한다. 스레드를 사용할 땐 풀에서 사용하고 사용이 끝나면 반납한다. 스레드의 최대치가 정해져 있고 그 이상이 필요하다면 대기하도록 설정할 수 있다. 

참고) 실무에서는 스레드 풀 안의 **최대 스레드 수**를 적당하게 설정하는 게 중요하다. 

### 🧩 웹 서비스의 데이터 전달 및 렌더링 방식
웹 서비스의 응답 데이터 형식
1. **정적 리소스** → 고정된 HTML 파일/CSS/JS/이미지/영상 등 제공
2. **동적 HTML 파일** → 서버에서 동적 HTML 파일을 생성해서 전달 (뷰 템플릿), 서버에서 렌더링을 마친 HTML을 브라우저는 단순히 그려줌 **(SSR)**
3. **HTTP API** → HTML 파일이 아니라 데이터를 전달, 주로 JSON 형식 사용
- 데이터만 주고받고 UI 화면이 필요하면 클라이언트가 별도로 처리한다.
- 예: 앱/웹 클라이언트 - 서버 **(CSR)**, 서버 to 서버

**SSR(서버 사이드 렌더링)**: HTML 최종 결과를 서버에서 만들어서 웹 브라우저에게 전달, 주로 정적인 화면에 사용

**CSR(클라이언트 사이드 렌더링)**: HTML 결과를 자바스크립트를 사용해 웹 브라우저에서 동적으로 생성해 적용, 주로 동적인 화면에 사용

<br>
<br>

# 🗓️ 2026-03-09 (월)
## 코테 - 문자열 파싱 문제
#### 패턴 접근법 
```
1. A → B 변환 문제: 매핑 + 반복문 
2. 입력을 어떻게 쪼갤지: 공백 기준 split 
3. 매핑 구조: 대응관계 → 배열, Map 
```
#### 실제 사고 흐름 
```
모스부호 
→ 변환 문제 

공백 있음 
→ split 

대응표 있음 
→ 배열 또는 map 

→ 반복문으로 붙이기
```

## 김영한의 스프링 MVC 2편 - 서블릿
### 🧩 직접 서블릿 등록하고 사용해보기
- 서블릿 자체는 스프링과 별개의 기술이다. 원래는 톰캣 같은 WAS를 직접 설치하고, 그 위에 서블릿 코드를 빌드해서 올리는데 스프링 부트는 톰캣 서버를 내장하고 있으므로 별도의 서버를 설치할 필요 없다. 
- JSP 실습을 하기 위해 War 방식으로 프로젝트를 생성했다. JAR는 내장 서버를 활용한 단순한 배포에 최적화된 방식이고, WAR는 JSP 같은 전통적인 웹 기술 지원이나 외장 WAS 환경과의 호환성을 위해 사용하는 방식이다. 
- 최상위 패키지에 `@ServletComponentScan`을 붙여 직접 서블릿을 등록하고 서블릿 클래스 위에는 `@WebServlet`을 붙인다. 

### 🧩 HttpServletRequest
서블릿은 개발자 대신에 HTTP 요청 메시지를 파싱해서 그 결과를 HttpServletRequest 객체 안에 담에 전달한다. 

HTTP 요청 메시지를 통해 클라이언트에서 서버로 데이터를 전달하는 방법에는 3가지가 있다. 
#### 1. GET - 쿼리 파라미터
- `/url?username=hello&age=20` (`?`을 시작으로, `&`로 추가)
- 메시지 바디 X, URL의 쿼리 파라미터에 데이터를 포함해 전달
- 예) 검색, 필터, 페이징 등에서 많이 사용
- HttpServletRequest에서는 다음 메서드를 제공해 쿼리 파라미터를 편리하게 조회할 수 있게 한다. 
```
// 1. 단일 파라미터 조회
String username = request.getParameter("username"); 

// 2. 파라미터 이름들 모두 조회 (Enumeration 타입)
Enumeration<String> parameterNames = request.getParameterNames(); 

// 3. 파라미터를 Map으로 조회 (모든 데이터를 한꺼번에 관리할 때 유용)
Map<String, String[]> parameterMap = request.getParameterMap(); 

// 4. 복수 파라미터 조회 (동일한 이름의 파라미터가 여러 개일 때, 예: username=kim&username=lee)
String[] usernames = request.getParameterValues("username");
```

#### 2. POST - HTML Form
- content-type: `application/x-www-form-urlencoded`
- 메시지 바디에 쿼리 파라미터 형식으로 전달 `username=hello&age=20`
- 예) 회원 가입, 상품 주문, HTML Form 사용 등
- `application/x-www-form-urlencoded` 형식은 앞서 GET에서 살펴본 쿼리 파라미터 형식과 같다. `request.getParameter()`는 1,2번 모두 지원하므로 여기에서도 쿼리 파라미터 조회 메서드를 그대로 사용하면 된다. 
- content-type은 HTTP 메시지 바디의 데이터 형식을 지정한다. 1번에서는 HTTP 메시지 바디를 사용하지 않았기 때문에 content-type이 따로 없다. 하지만 2번에서는 바디에 해당 데이터를 포함해서 보내기 때문에 반드시 데이터의 형식을 지정해줘야 한다. 

#### 3. HTTP message body에 데이터를 직접 담아서 요청
- HTTP API에서 주로 사용 (JSON, XML, TEXT)
- 데이터 형식은 주로 JSON 사용
- POST, PUT, PATCH
- 단순한 텍스트 데이터를 담을 수도 있고, JSON 형식으로 데이터를 지정할 수 있다. 
```
- 문자 전송
POST http://localhost:8080/request-body-string content-type: text/plain
message body: hello
결과: messageBody = hello

- JSON 형식 전송
POST http://localhost:8080/request-body-json
content-type: application/json
message body: {"username": "hello", "age": 20}
결과: messageBody = {"username": "hello", "age": 20}
```
- JSON 결과를 파싱해서 사용할 수 있는 자바 객체로 변환하려면 Jackson, Gson 같은 JSON 변환 라이브러리를 추가해서 사용해야 한다. 스프링 부트로 Spring MVC를 선택하면 기본으로 Jackson 라이브러리
( `ObjectMapper` )를 함께 제공한다.
- HTML form 데이터도 메시지 바디를 통해 전송되므로 직접 읽을 수 있다. 하지만 편리한 파리미터 조회 기능 ( `request.getParameter(...)` )을 이미 제공하기 때문에 파라미터 조회 기능을 사용하면 된다.

<br>
<br>

# 🗓️ 2026-03-10 (화)
## 🧩 코테 - char 산술 연산과 문자 ↔ 숫자 변환
자바에서 `char`은 내부적으로 유니코드 정수값을 가진다. `char`에 산술 연산(`+ - * /`)이 붙으면 `int` 타입으로 바꿔서 계산할 수 있다. 그러나 `String`은 숫자가 아니라 객체이므로 `+`로 문자열 연결하는 것 이외에는 연산자가 통하지 않는다. 

자주 나오는 패턴
```
문자 → 숫자
int num = c - '0';

숫자 → 문자
char c = (char)(num + '0');

예)
int num = 7;
char c = (char)(num + '0');  // '7'
```
이를 활용해 문자를 배열 인덱스로 바꿀 수 있다. 
```
char → 숫자 변환 → 배열 인덱스 → 값 꺼내기

result.append(win[c - '0']);
```
## 🧩 MockMvc와 DispatcherServlet: 컨트롤러 테스트의 동작 원리
웹 앱을 실행하면 서블릿 컨테이너(톰캣)이 **DispatcherServlet**에게  브라우저의 요청을 전달하고, DispatcherServlet이 해당 요청에 적절한 컨트롤러를 찾아 전달해준다. (**프론트 컨트롤러** 역할)

**MockMvc**는 실제 서블릿 컨테이너를 띄우지 않고 그 동작을 흉내내는 모킹 환경을 구성하는 테스트 도구이다. MockMvc는 **실제 HTTP 서버를 거치지 않고 DispatcherServlet에 직접 요청을 전달하여 스프링 MVC 흐름을 테스트**한다.    

결과적으로 테스트 환경에서 서버를 실행하지 않고 컨트롤러의 로직을 빠르게 확인할 수 있다. 

MockMvc는 Spring MVC 계층을 테스트하기 위한 도구로 주로 **컨트롤러 테스트**에 사용되지만, @SpringBootTest와 함께 사용하면 서비스와 리포지토리까지 포함한 **애플리케이션 전체 흐름을 테스트**할 수도 있다.

모킹 환경을 만드는 어노테이션에는 두 가지가 있다. 
- `@WebMvcTest`: 슬라이스 테스트. 전체 스프링 컨텍스트를 로드하지 않고, MVC 계층과 관련된 빈들만 제한적으로 로드한다. @Service나 @Repository는 로드되지 않으므로, 이들은 보통 @MockBean을 사용하여 가짜로 대체된다. 필요한 부분만 떼어내서 테스트하므로 가볍고 빠르다. 

- `@AutoConfigureMockMvc`: 통합 테스트. 보통 @SpringBootTest와 함께 사용되며, 전체 스프링 컨텍스트를 로드한 뒤 MockMvc 객체를 자동으로 설정해준다. 서비스, 리포지토리, DB까지 포함된 전체 애플리케이션 흐름을 테스트할 수 있다.

<br>
<br>

# 🗓️ 2026-03-11 (수) ~ 2026-03-12 (목)
### 🧩 [트러블슈팅] Spring Boot 4.0 & JJWT 0.13 마이그레이션: JWT 인증 테스트 에러 해결

- [[트러블슈팅] Spring Boot 4.0 & JJWT 0.13 마이그레이션: JWT 인증 테스트 에러 해결](https://velog.io/@hjy648012/%ED%8A%B8%EB%9F%AC%EB%B8%94%EC%8A%88%ED%8C%85-Spring-Boot-4.0-JJWT-0.13-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98-JWT-%EC%9D%B8%EC%A6%9D-%ED%85%8C%EC%8A%A4%ED%8A%B8-%EC%97%90%EB%9F%AC-%ED%95%B4%EA%B2%B0)
- [JJWT 0.10.0 이후 signWith(SignatureAlgorithm, String) 메서드 deprecated](https://velog.io/@hjy648012/JJWT-0.10.0-%EC%9D%B4%ED%9B%84-signWithSignatureAlgorithm-String-%EB%A9%94%EC%84%9C%EB%93%9C-deprecated)

1. 스프링 업그레이드에 따른 의존성 수정 : Mock 관련 어노테이션 인식 불가 (TokenApiControllerTest)
2. jjwt 버전 마이그레이션 : 빌더 메서드 명칭 변경, Key 생성 방식 변경 (TokenProvider, TokenProviderTest)
3. 설정 파일 (application.yml) 오타

<br>
<br>

# 🗓️ 2026-03-13 (금)

프로젝트 확장 기획: [project-plan.md](project-plan.md)

<br>
<br>

# 🗓️ 2026-03-15 (일) ~ 2026-03-16 (월)
### 🧩 깃허브 Action CI 도전과 트러블슈팅

블로그 기록:  
[[트러블슈팅] GitHub Actions CI 실패 – spring-boot-starter-security-test 의존성 groupId 오류](https://velog.io/@hjy648012/%ED%8A%B8%EB%9F%AC%EB%B8%94%EC%8A%88%ED%8C%85-GitHub-Actions-CI-%EC%8B%A4%ED%8C%A8-spring-boot-starter-security-test-%EC%9D%98%EC%A1%B4%EC%84%B1-groupId-%EC%98%A4%EB%A5%98)

<br>
<br>
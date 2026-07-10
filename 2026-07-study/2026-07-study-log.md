# 🗓️ 2026-07-08 (수)
## 🧩 @Configuration과 @Bean, 그리고 싱글톤이 유지되는 원리

**📌 학습 계기:** 프로젝트 SecurityConfig의 `@Configuration` 어노테이션이 정확히 뭘 하는지 궁금해서 찾아봄  

**📌 공식 문서 참고:** [Basic Concepts: @Bean and @Configuration](https://docs.spring.io/spring-framework/reference/core/beans/java/basic-concepts.html#page-title)

### 1. 스프링 컨테이너에 빈을 등록하는 두 가지 방식
- **Annotation-based Container Configuration** = Component Scan 방식 (자동 등록, `@Component`/`@Service` 등)
- **Java-based Container Configuration** = `@Configuration` + `@Bean` 방식 (수동 등록)

### 2. @Configuration 클래스와 @Bean 메서드
- `@Configuration` 클래스는 `@Bean` 메서드를 정의하는 설정 클래스다. `@Bean` 메서드는 주로 `@Configuration` 클래스 안에서 정의된다.
- `@Bean` 메서드가 반환하는 객체가 스프링 빈으로 등록된다.
- 같은 `@Configuration` 클래스 안에서 `@Bean` 메서드끼리는 서로를 호출하며 의존관계를 형성할 수 있다.
- 이때 이미 등록된 빈이 있다면 새 객체를 생성하지 않고 컨테이너에 등록된 기존 빈을 반환한다 (싱글톤 유지).
- 이 역할은 스프링이 만드는 **CGLIB 프록시 객체**가 수행한다. `@Configuration` 클래스 자체가 프록시로 감싸져서, 빈 메서드 호출을 스프링이 가로챌 수 있게 되는 것.

### 3. Lite Mode (프록시가 없는 경우)
- 빈 메서드가 `@Configuration`이 붙지 않은 클래스, 또는 `@Configuration(proxyBeanMethods = false)`가 붙은 클래스 안에 있으면 **Lite Mode**로 동작하며 프록시를 만들지 않는다.
- 프록시가 없으면 빈 메서드를 직접 호출해도 스프링이 가로채지 못해서 **매번 새 객체가 생성**된다. 따라서 이 경우엔 빈끼리 서로 메서드로 호출하지 말고, 필요한 빈을 **메서드 파라미터로 받아야** 한다.
- 대신 프록시를 안 만드니 메모리 사용량 감소, 시작 속도 향상, 런타임 오버헤드 감소라는 장점이 있다.

### 오늘 실제로 본 코드
`SecurityConfig`는 `jwtFilter`를 필드 주입(`@RequiredArgsConstructor`)으로 받는데, 
이건 `@Bean` 메서드끼리 서로 호출하는 것과는 별개의 메커니즘이다.
`jwtFilter`는 `SecurityConfig` 안에 `@Bean`으로 등록된 게 아니라 이미 다른 곳
(`@Component` 등)에서 빈으로 등록된 걸 생성자로 주입받는 것뿐이라, 
"메서드 호출로 새 객체가 중복 생성되는 문제" 자체가 발생할 상황이 아니다.

우리 프로젝트 Config 클래스(SecurityConfig, QueryDslConfig, RedisClientConfig, 
RestTemplateConfig, SwaggerConfig, WebSocketConfig)는 전부 `@Bean` 메서드끼리 
서로 호출하는 코드가 없다. 그래서 `@Configuration(proxyBeanMethods = false)`를 
붙여도 안전하게 동작하며, CGLIB 프록시 생성 비용(런타임 바이트코드 생성, 
클래스 로딩, 메모리)을 절약할 수 있다. 단, 이 비용은 애플리케이션 시작 시점에 
한 번만 발생하는 미미한 수준이라 실제 성능 개선 체감은 거의 없다 — 그래도 
"왜 이 옵션을 켰는지" 설명할 수 있다는 점에서 학습 목적으로는 
붙여볼 가치가 있다.

예시:  
````java
@Configuration(proxyBeanMethods = false)
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {
    ...
}
````

<br>
<br>

# 🗓️ 2026-07-09 (목)
## 🧩 Spring IoC 컨테이너와 Bean
- **IoC**: 객체가 의존성을 직접 만들지 않고, 컨테이너가 대신 생성해서 주입
- **DI**: IoC를 실현하는 방법. 생성자 인자 / 팩토리 메서드 인자 / setter로 의존성을 "정의"만 하면 컨테이너가 주입
- **핵심 인터페이스**
  - `BeanFactory`: 기본 컨테이너 기능
  - `ApplicationContext`: BeanFactory + AOP, 국제화, 이벤트 발행 등 (실무 표준)
- **Bean**: 컨테이너가 생성·조립·관리하는 객체. 설정 메타데이터(XML, @Configuration 등)에 Bean과 의존관계가 정의됨

```
설정 메타데이터 (@Component, @Bean...)
                  │
                  ▼
          ApplicationContext (IoC 컨테이너)
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
    Bean 생성/조립        의존성 주입(DI)
        └─────────┬─────────┘
                  ▼
            생명주기 관리
        (초기화 → 사용 → 소멸)
```

<br>
<br>
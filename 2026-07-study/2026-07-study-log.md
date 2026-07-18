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

### 💥 우리 프로젝트 코드
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

# 🗓️ 2026-07-09 (목) ~ 2026-07-10 (금)
## 🧩 Spring IoC 컨테이너와 Bean

**📌 공식 문서 참고:** [Container Overview](https://docs.spring.io/spring-framework/reference/core/beans/basics.html)  
**📌 강의 참고:** 김영한의 스프링 핵심 원리 - 기본편

- **IoC**: 객체가 의존성을 직접 만들지 않고, 컨테이너가 대신 생성해서 주입
- **DI**: IoC를 실현하는 방법. 생성자 인자 / 팩토리 메서드 인자 / setter로 의존성을 "정의"만 하면 컨테이너가 주입
- **핵심 인터페이스**
  - `BeanFactory`: 기본 컨테이너 기능
  - `ApplicationContext` (실무 표준): BeanFactory + 부가 기능
    - BeanFactory의 기능을 상속받음
    - 부가 기능: 국제화, 이벤트 발행, 환경 변수, 리소스 조회, AOP 처리기(BeanPostProcessor) 등록 등
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
### ApplicationContext 생성 방식  
스프링 컨테이너(ApplicationContext)는 XML 설정 또는 애노테이션 기반 자바 설정 클래스(@Configuration)를 읽어들여 만들어지며, 어떤 방식의 설정 메타데이터를 쓰느냐에 따라 컨테이너의 구현체가 달라진다. 요즘은 애노테이션 기반 자바 설정 클래스를 주로 사용한다. 

### 각 방식별 구현체

| 설정 방식 | ApplicationContext의 구현체 |
|---|---|
| XML 설정 | `ClassPathXmlApplicationContext` |
| 애노테이션 기반 자바 설정 클래스 | `AnnotationConfigApplicationContext` |
| 애노테이션 기반 + 웹 애플리케이션 (Spring Boot) | `AnnotationConfigServletWebServerApplicationContext` |

```
ApplicationContext (인터페이스)
    ├── ClassPathXmlApplicationContext        ← XML 읽음
    ├── AnnotationConfigApplicationContext    ← @Configuration 클래스 읽음
    └── AnnotationConfigServletWebServerApplicationContext
              ← @Configuration 클래스 읽음 + 내장 웹 서버 통합 (Spring Boot 웹 앱이 실제로 쓰는 것)
```
**어떤 형식의 설정 메타데이터를 쓰느냐가 구현체를 결정**한다.  
우리 프로젝트(Spring Boot 웹 앱)에는 `AnnotationConfigServletWebServerApplicationContext`가 내부적으로 쓰인다. (컴포넌트 스캔/자동설정 읽는 로직 + 내장 톰캣 구동)

### BeanDefinition — 설정 메타데이터의 통일된 내부 표현
컨테이너 내부에서 설정 메타데이터(XML, @Configuration, @Component 스캔 등)는 **BeanDefinition 인터페이스**로 통일되어 표현되며, @Bean(또는 `<bean>`)당 각각 하나씩 생성된다. **컨테이너 로직은 BeanDefinition 추상화에만 의존**한다.

<br>
<br>

# 🗓️ 2026-07-10 (금)
## 🧩 Spring 애플리케이션을 실행하면 무슨 일이 벌어질까?

### 1단계 — `SpringApplication.run()`
- `main()`에서 호출되면 `ApplicationContext` 구현체 생성 (웹 앱이면 내장 톰캣 통합 버전)
- `Environment` 준비 (profile, application.yml 값 읽기)

### 2단계 — 컨테이너 초기화 (`refresh()`)

1. **BeanDefinition 수집**
   - `@ComponentScan`으로 `@Component`/`@Service`/`@Repository`/`@Controller` 스캔
   - `@EnableAutoConfiguration`으로 클래스패스 기준 자동설정 클래스 조건부 등록
   - 이 시점엔 메타데이터만 존재, 실제 객체 없음

2. **BeanPostProcessor 등록**
   - `@Autowired` 처리기, AOP 프록시 생성기 등 개입용 특수 빈 먼저 준비

3. **싱글톤 빈 인스턴스화**
   - BeanDefinition 기반으로 실제 객체 생성 + 의존성 주입 (생성자 주입이면 필요한 빈부터 먼저 생성)
   - `@Transactional` 등 AOP 대상이면 이 과정에서 프록시로 교체
   - 완성된 빈은 내부 캐시에 저장

4. **내장 웹 서버 시작** (웹 앱인 경우)

### 3단계 — 애플리케이션 준비 완료
- `ApplicationRunner`/`CommandLineRunner` 실행 → 요청 받을 준비 완료

## 🧩 요약 : 컴포넌트 스캔 → 빈 정의 → 인스턴스화 → 프록시 → 서버 시작

- `SpringApplication.run()` → 컨테이너 생성 → 컴포넌트 스캔/자동설정으로 빈 정의 수집 → 실제 빈 생성 + 의존성 주입 → AOP 대상은 프록시로 교체 → (웹 앱이면) 내장 톰캣 시작 → 요청 처리 준비 완료

- **BeanDefinition:** 빈의 메타정보(타입, 스코프, 생성자 인자, 의존관계 등)를 담은 "설계도". 실제 인스턴스 아님
- **프록시 생성 시점:** `@Transactional`, `@Async` 등 AOP 대상 빈이 생성된 직후, `BeanPostProcessor` 단계에서 원본 대신 프록시로 교체
- **싱글톤이 기본인 이유:** stateless 서비스/리포지토리는 매번 새로 만들 필요 없이 하나만 공유하는 게 효율적
- **`@SpringBootApplication`:** `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` 조합
- **기타 스위치성 애노테이션:** `@EnableJpaAuditing`, `@ConfigurationPropertiesScan`, `@EnableScheduling`, `@EnableAsync` — 각각 특정 기능을 켜는 용도, 컨테이너 부팅 흐름 자체와는 무관

<br>
<br>

# 🗓️ 2026-07-11 (토) ~ 2026-07-13 (월)
## 🧩 싱글톤 패턴과 스프링 컨테이너

**📌 블로그 정리:** [스프링 빈은 왜 싱글톤인가 — 그리고 왜 상태를 가지면 안 되는가
](https://velog.io/@hjy648012/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%B9%88%EC%9D%80-%EC%99%9C-%EC%8B%B1%EA%B8%80%ED%86%A4%EC%9D%B8%EA%B0%80-%EA%B7%B8%EB%A6%AC%EA%B3%A0-%EC%99%9C-%EC%83%81%ED%83%9C%EB%A5%BC-%EA%B0%80%EC%A7%80%EB%A9%B4-%EC%95%88-%EB%90%98%EB%8A%94%EA%B0%80)

<br>

## 🧩 `@Primary` / `@Qualifier`로 프로젝트 개선 아이디어

`@Component`는 "이 구현체를 쓰겠다"는 선택이 아니라 **"컨테이너 관리 대상으로 등록해라"** 는 의미. 같은 인터페이스에 여러 구현체가 `@Component`로 등록되면, 스프링은 어떤 걸 주입해야 할지 몰라서 `NoUniqueBeanDefinitionException` 발생.

| 상황 | 도구 |
|---|---|
| 기본 구현체가 명확, 예외적으로만 다른 걸 씀 | `@Primary` (+ 필요한 곳만 `@Qualifier`) |
| 컴파일 시점에 "이 필드는 항상 이 구현체" | `@Qualifier` |
| 런타임에 값(사용자 선택 등)에 따라 구현체가 바뀜 | `Map<String, T>` 주입 + 라우팅 로직 |

**적용 가능 지점 (프로젝트 확장 시)**
- 결제수단이 여러 개로 늘어난다면: `Map<String, PaymentService>` 주입받는 라우터를 만들어 사용자가 고른 타입에 따라 분기 (`@Qualifier`는 런타임 분기엔 부적합, 컴파일 시점 고정 관계에만 적합)
- 알림 발송(이메일/SMS/푸시)처럼 상황별로 정해진 채널을 고정적으로 쓸 때 → `@Qualifier`
- 운영/로컬 환경별 구현체 스위칭(Toss 실결제 vs Mock) → `@Primary` + 테스트에서만 `@Qualifier`로 명시

예약(Reservation) 자체는 "구현체 선택" 문제라기보다 책임 분리·트랜잭션 경계·동시성 문제에 가까워서 이 패턴이 덜 어울림.

<br>
<br>

# 🗓️ 2026-07-14 (화)
## 🧩 DI 복습하다 발견한 리팩토링 후보: 생성자 vs @PostConstruct

**📌 강의 참고:** 김영한의 스프링 핵심 원리 - 기본편

### 1. 컴포넌트 스캔 & 의존관계 주입
- `@SpringBootApplication`에 `@ComponentScan` 포함 (관례상 루트 위치)
- 기본 스캔 대상: `@Component`, `@Controller`, `@Service`, `@Repository`, `@Configuration`
- 의존관계 주입 4가지 중 **생성자 주입이 최근 표준**
  - 불변/필수 의존관계에 적합, `final` 사용 가능한 유일한 방식
  - 생성자 1개면 `@Autowired` 생략 가능 + `@RequiredArgsConstructor` 조합이 요즘 관행
- 빈이 2개 이상 매칭될 때 해결 순서: `@Autowired` 타입/이름 매칭 → `@Qualifier` → `@Primary`

### 2. 빈 라이프사이클 리팩토링 후보
- **핵심 원칙**: 생성자는 "객체 생성"만, 주입값을 활용한 초기화는 `@PostConstruct`에서
- 체크리스트:
  - `PaymentService`의 `TossPaymentsClient` 생성 위치 (필드/생성자에서 바로 `new` 하면 `secretKey` 주입 전에 실행될 위험)
  - `CampingImageService`의 `S3Client` 생성 위치 + `@PreDestroy`로 `close()` 정리 로직 유무
- 판단 기준: "이 필드 준비에 다른 주입값이 필요한가?" → Yes면 `@PostConstruct` 후보

### 3. 로깅
- `println`과 로깅 프레임워크(Logback 등) 차이: 레벨 조절, 설정 파일로 on/off, 부가정보 자동 포함
- 로그 레벨: `TRACE < DEBUG < INFO < WARN < ERROR` (기본 INFO 이상)
- 적용 예정: `application.yml`에 `org.hibernate.SQL: DEBUG` 켜서 QueryDSL 실제 SQL 확인 (N+1 이슈 탐지용)

---

**다음 할 일**: `PaymentService`/`CampingImageService` 코드 직접 열어서 `@PostConstruct` 적용 여부 확인 → 적용 후 서버 재시작 테스트, 로깅 설정 추가해서 QueryDSL 쿼리 확인

<br>
<br>

# 🗓️ 2026-07-16 (목)
## 🧩 클라우드/AWS 기초 개념 정리

**📌 도서 참고:** AWS 교과서

클라우드: 인터넷상 IT 자원들의 집합  
클라우드 컴퓨팅: 인터넷상 IT 자원들을 원격으로 빌려 쓰는 서비스

### 클라우드 컴퓨팅 서비스 유형
- **IaaS** (Infrastructure as a Service): 인프라(서버, 네트워크, 스토리지)를 서비스로 제공
  - 예: EC2, S3 → 운영체제/미들웨어 설정은 사용자 몫
- **PaaS** (Platform as a Service): 애플리케이션 실행 플랫폼까지 제공
  - 예: RDS (DB 엔진 설치/패치는 AWS가 관리, 사용자는 데이터/쿼리에만 집중)
  - Railway, Render도 PaaS에 가까움 (배포 프리티어 대안 찾을 때 봤던 것들)
- **SaaS** (Software as a Service): 완성된 소프트웨어 자체를 제공
  - 예: Gmail, Notion — 인프라/플랫폼 신경 안 쓰고 바로 사용

밑으로 내려갈수록(IaaS) 내가 직접 관리할 게 많아지고 자유도가 높음, 위로 올라갈수록(SaaS) 관리 부담은 없지만 커스터마이징 자유도가 낮음

### 클라우드 배포 모델
- 퍼블릭 클라우드 (AWS, GCP, Azure) — 우리 프로젝트가 쓰는 방식
- 프라이빗 클라우드 — 회사 자체 데이터센터, 보안/규제 이유로 씀
- 하이브리드 클라우드 — 민감 데이터는 프라이빗, 나머지는 퍼블릭에 분산

### AWS 서비스 종류
- 컴퓨팅: EC2
- 네트워킹: CloudFront
- 스토리지: S3
- 데이터베이스: RDS
- 보안 자격: IAM   
...

<br>
<br>
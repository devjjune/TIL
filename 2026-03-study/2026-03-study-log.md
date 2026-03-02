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
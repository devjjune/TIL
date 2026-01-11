# 🗓️ 2026-01-01 (목)
**1. spring-boot-blog**
- JWT 토큰 API 구현 완료
- 리프레시 토큰 도메인 구현 → 토큰 필터 구현 → 토큰 API 구현

**2. 문제 해결 회고 작성**
- [Spring 애플리케이션 에러 로그 읽는 법](https://velog.io/@hjy648012/Spring-%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98-%EC%97%90%EB%9F%AC-%EB%A1%9C%EA%B7%B8-%EC%9D%BD%EB%8A%94-%EB%B2%95)

<br>
<br>

# 🗓️ 2026-01-02 (금)
**1. spring-boot-blog**
- **OAuth2 로그인 구현 전 '인증/인가 흐름' 학습: [OAuth 관련 용어와 인증 흐름](https://velog.io/@hjy648012/OAuth%EB%9E%80-%EA%B4%80%EB%A0%A8-%EC%9A%A9%EC%96%B4%EC%99%80-%EC%9E%91%EB%8F%99-%EB%B0%A9%EC%8B%9D)**
- 기존에 작성했던 글에 **‘권한 부여 코드 승인 타입(Authorization Code Grant)’의 동작 과정**을 추가했다.
- 특히 인증과 권한 부여가 분리되어 이루어지는 과정, 인증 코드를 액세스 토큰으로 교환하는 과정에 초점을 맞춰 정리했다.
- 책에 있는 OAuth 흐름 그림이 직관적으로 이해되지 않아 내가 이해한 방식대로 도식과 설명을 수정했다.
- 이 과정에서 예상보다 시간이 오래 걸렸지만 그만큼 더 기억에 오래 남을 것 같다. 

<br>
<br>

# 🗓️ 2026-01-03 (토)
**1. spring-boot-blog**
- **OAuth2 로그인을 구현**하며, 책에 있는 코드를 그대로 옮기는 데서 생각보다 많은 문제가 발생했다.
- 내 프로젝트로 옮기자 스프링 버전 차이, 빈 순환 의존성, 보안 설정 충돌 등의 문제가 발생했다. 
- 당장은 배포까지 한 사이클을 빠르게 돌리는 게 목표지만, 단순히 동작하게 만드는 데서 그치지 않고 스프링 구조를 정말로 내 것으로 만들어야겠다는 생각이 들었다.

<br>
<br>

# 🗓️ 2026-01-04 (일)
1. 문제 해결 회고 작성: [[Spring][JWT] base64-encoded secret key 오류 - 알고 보니 YAML 들여쓰기](https://velog.io/@hjy648012/Spring-Boot-JWT-base64-encoded-secret-key-%EC%98%A4%EB%A5%98-%EC%95%8C%EA%B3%A0-%EB%B3%B4%EB%8B%88-YAML-%EB%93%A4%EC%97%AC%EC%93%B0%EA%B8%B0)
2. 데브코스 지원서 작성
3. 기술 블로그 정리: '글의 성격 + 핵심 기술 스택' 별로 태그(카테고리) 구성

<br>
<br>

# 🗓️ 2026-01-06 (화)
**1. 기술 블로그 태그 기준 정리**  
- 블로그 글을 **태그(카테고리) 기준**으로 다시 정리했다. 태그는 나중에 검색해서 바로 찾을 수 있도록, 기술 스택을 나열하거나 범위가 너무 넓어지지 않게 **핵심 소재 위주**로 구성했다.  
  
**2. spring-boot-blog**
- AWS 배포 전 관련 개념 학습: [[클라우드 서버] AWS 서비스 구조와 일래스틱 빈스토크](https://velog.io/@hjy648012/%ED%81%B4%EB%9D%BC%EC%9A%B0%EB%93%9C-%EC%84%9C%EB%B2%84-AWS-%EC%84%9C%EB%B9%84%EC%8A%A4-%EA%B5%AC%EC%A1%B0%EC%99%80-%EC%9D%BC%EB%9E%98%EC%8A%A4%ED%8B%B1-%EB%B9%88%EC%8A%A4%ED%86%A0%ED%81%AC)

<br>
<br>

# 🗓️ 2026-01-07 (수) ~ 2026-01-10 (토)
**1. spring-boot-blog : AWS에 프로젝트 배포**
- Elastic Beanstalk로 서버 환경 구성
  - Elastic Beanstalk 서비스 생성
  - RDS(MySQL) 생성 및 DB 접속 정보 저장
  - 애플리케이션에서 사용할 RDS 정보 환경 변수로 관리
- 로컬에서 RDS 연결 및 초기 세팅
  - RDS 보안 그룹 인바운드 규칙 수정 (로컬 접속 허용)
  - IntelliJ Database Navigator 플러그인으로 로컬에서 RDS 연결
  - SQL 작성하여 초기 테이블 생성
  - 프로젝트에 MySQL 의존성 추가
- Elastic Beanstalk에 애플리케이션 배포
  - 프로젝트 빌드 후 실행 가능한 JAR 파일 생성
  - Elastic Beanstalk에 JAR 업로드 및 배포
  - Google OAuth 서비스에 승인된 Redirect URI 추가

**2. 스프링부트 복습 - 김영한의 스프링 입문 강의 복습**

<br>
<br>

# 🗓️ 2026-01-11 (일)
**1. spring-boot-blog : AWS에 프로젝트 배포**
- 배포 중 발생한 502 에러에 대해 트러블슈팅 회고 작성: [[AWS] Elastic Beanstalk 배포 후 502 오류 원인과 해결](https://velog.io/@hjy648012/AWS-Elastic-Beanstalk-%EB%B0%B0%ED%8F%AC-%ED%9B%84-502-%EC%98%A4%EB%A5%98-%EC%9B%90%EC%9D%B8%EA%B3%BC-%ED%95%B4%EA%B2%B0)

**2. 스프링부트 복습 - 김영한의 스프링 입문 강의 복습**

<br>
<br>
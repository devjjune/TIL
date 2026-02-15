# 🗓️ 2026-01-28 (수)
**1. 작은 프로젝트에도 MVC가 꼭 필요할까?**
- 자바 명언 게시판을 구현하면서, **요구사항이 비교적 간단하고 단순한 경우에도 MVC 패턴을 사용하는 게 오버엔지니어링이 아닐지** 우려되었다. 
- 하지만 초반부터 MVC 패턴으로 설계하는 것이 기능을 클래스별로 분리해서 넣기에도 편하고, 이후에 기능이 확장될 때 유지보수하기 용이하다는 점을 느꼈다. 

<br>
<br>

# 🗓️ 2026-01-29 (목)
### 🧩 언제 static을 사용할까?
static은 멤버 변수나 메서드를 클래스에 종속시킬 때 사용한다.  

**상태 없는 유틸성 클래스**인 경우 static을 사용하는 게 더 자연스럽다.  
(_상태를 가지지 않는다: 객체 내부에서 값을 저장하지 않고, 단순히 값을 입력받고 처리해서 반환하는 기능만을 수행한다_)

MVC 구조에서 View는 흔히 static 기반의 유틸리티 클래스로 남겨두지만 그 외(Controller 등)에선 static을 지양하고 인스턴스(객체)로 사용하는 것이 권장된다. 

**static을 지양해야 하는 이유: '확장성'과 '테스트 독립성'**  
**static이 불리한 경우: 상태를 내부 필드에 저장해야 할 때, 객체마다 다른 상태/데이터를 가져야 할 때**

### 🧩 Scanner의 위치
`Scanner scanner = new Scanner(System.in)`  

`System.in`(시스템 자원)과 이를 가져오는 `Scanner` 객체는 전체 프로그램 내에서 하나만 생성해 공유하는 것이 메모리 관리와 테스트 측면(자동화 테스트(입력값 미리 넣어두기))에서 유리하다. 

- Application: Scanner 생성 → Controller 생성자 호출 시 전달
- Controller: 전달받은 Scanner를 자기 필드에 보관
- Controller → View: 입력이 필요할 때 InputView.read(scanner) 식으로 빌려줌. 

<br>
<br>

# 🗓️ 2026-01-30 (금)
### 🧩 프로그래머스 코테 문법 정리
#### 1. 배열 관련
&rightarrow; [자바 배열 문법 정리](https://velog.io/@hjy648012/%EC%BD%94%ED%85%8C-%EC%9E%90%EB%B0%94-%EB%B0%B0%EC%97%B4-%EB%AC%B8%EB%B2%95-%EC%A0%95%EB%A6%AC)  
- 특히 `Arrays.sort()`는 최댓값, 최솟값을 찾거나 순서를 세울 때 유용하게 쓸 수 있다.  
- 예) 최댓값 두 개 곱하기, 삼각형 조건 (변 길이 비교)

#### 2. 향상된 for문
- 인덱스(i) 없이 요소만 순차적으로 꺼낼 때 사용한다.  
- 사용 가능 대상: 배열, 리스트, 셋 등 Iterable 인터페이스를 구현한 클래스에선 모두 사용 가능하다.  
- 단, String과 Map은 직접 사용할 수 없어 변환 과정이 필요하다. 

#### 3. 삼항 연산자
- 문법: `(조건식) ? 참일_때_값 : 거짓일_때_값;`
- 용도: 단순한 if-else 문을 리턴해야 할 때 코드를 간결하게 만들어 준다.

#### 4. 메서드 체이닝 기법과 자바 Stream
- 메서드가 '객체 자신'을 반환한 다음 점(.)을 찍고 이어서 다음 할 일을 시키는 **기법**이다.   
    ```java
    // 단순히 문자를 뒤집는 '과정'을 체이닝으로 표현
    String result = new StringBuilder("abc").reverse().append("d").toString();
    ```
- 자바의 Stream 문법은 메서드 체이닝 기법을 활용하여, 데이터를 '흐름'에 따라 설계할 때 사용하는 **도구**이다.  
    ```java
    // 여러 데이터 중 '조건에 맞는 것만 골라내는 과정'을 체이닝으로 표현
    long count = Arrays.stream(numbers) // 1. 스트림 생성
                    .filter(n -> n > 10) // 2. 가공 (체이닝)
                    .count();            // 3. 결과 (체이닝)
    ```

#### 5. 가변 문자열 클래스 (StringBuilder & StringBuffer)
- String은 '불변'이라 수정 시 매번 새로운 객체를 생성(참조값 교체)해야 하지만, 위 클래스들은 내장된 메모리 공간(버퍼)을 활용해 하나의 객체 안에서 데이터를 직접 수정한다. 
- **StringBuilder**: 단일 스레드 환경(일반적인 상황)에서 가장 빨라 주로 사용된다. 
- **StringBuffer**: 멀티스레드 환경에서 데이터 안전성(동기화)을 보장한다. 
- 핵심 메서드: `.append()`, `.reverse()`, `.insert()`, `.delete()`, `.toString()`

<br>
<br>

# 🗓️ 2026-01-31 (토) ~ 2026-02-01 (일)
### 🧩 Java Stream 개념 및 관련 문법 정리

데이터의 흐름을 의미하는 자바 스트림은 배열이나 컬렉션을 표준화된 함수형 연산(람다식)으로 간단하게 처리할 수 있는 기술이다. 스트림은 항상 '생성 -> 가공 -> 결과'의 세 단계를 거치며 크게 '중간 연산'과 '최종 연산'이라는 두 가지 연산을 제공한다. 

블로그 정리: [Java Stream 개념과 문법 정리](https://velog.io/@hjy648012/Java-Stream-%EB%AC%B8%EB%B2%95-%EC%A0%95%EB%A6%AC)

<br>
<br>

# 🗓️ 2026-02-02 (월)
## 🧩 수업 내용 정리
### 1. Java Stream

- 정리 문서: [02_수업내용_정리(자바Stream)](02_수업내용_정리(자바Stream).md)

### 2. Git 설치와 설정 방법

- 블로그 정리: [[Mac] Git 설치와 설정법 (Homebrew, GitHub CLI, .zshrc, Docker 등)
](https://velog.io/@hjy648012/Mac-Git-%EC%84%A4%EC%B9%98%EC%99%80-%EC%84%A4%EC%A0%95%EB%B2%95-Homebrew-GitHub-CLI-.zshrc-Docker-%EB%93%B1)


### 3. Git 문법
#### git log —oneline

- git log에 —online 옵션을 붙이면 간략하게 커밋 기록들을 볼 수 있다.
- alias를 이용해 편하게 쓰는 법

```bash
alias gl="git log --oneline"
```

#### git checkout

- 커밋 간 이동 (시간 여행, 특정 save point를 불러오기)

```bash
cd ~/Custom/GitProjects/work1
git log --oneline

git checkout 최초커밋코드 # 과거로 돌아감
ls # 파일 a 만 존재함

git checkout main # 현재로 돌아옴
ls # 파일 a, b, c
```
<br>
<br>

# 🗓️ 2026-02-03 (화)
### 🧩 리액트 간단하게 정리
수업 내용에 살을 붙여 간단히 핵심만 정리했다.   
→ [03_리액트_정리](03_리액트_정리.md)

<br>
<br>

# 🗓️ 2026-02-04 (수)
## 🧩 명언 게시판 리팩토링
### 반복문 & 조건문 → 자바 Stream
크게 복잡한 로직은 아니지만 들여쓰기 depth도 줄일 겸 Stream으로 바꾸는 연습을 했다. 
```java
// 리팩토링 전
private WiseSaying findByTargetId(int targetId) {
        for (WiseSaying ws : wiseSayingList) {
            if (ws.getId() == targetId) {
                return ws;
            }
        }
        return null;
    }
```
```java
// 리팩토링 후
private WiseSaying findByTargetId(int targetId) {
        return wiseSayingList.stream()
                .filter(ws -> ws.getId() == targetId)
                .findFirst()
                .orElse(null);
    }
```

<br>
<br>

# 🗓️ 2026-02-05 (목)
## 🧩 명언 게시판 리팩토링: Rq 클래스 도입
### Rq 클래스에 입력값 분석 로직 분리
사용자 입력값을 분석해서 명령어(actionName)과 id 번호를 파싱하는 작업을 모두 컨트롤러에서 수행했었는데, Rq 클래스를 생성해 이를 분리했다. 
```
Controller → 사용자와 상호작용 담당
사용자의 요청을 분석/처리 → 다른 클래스(Rq)로 분리
```
- id는 숫자와 매핑해서 Map에 저장 → 직관적, 데이터 확장 용이
- 객체를 생성해서 매개변수로 넘겨주는 방식(의존성 주입):   
  Scanner처럼 프로그램 전체에서 공용으로 돌려 쓰는 경우, 또는 사용자가 입력할 때마다 내용이 바뀌는 Rq처럼 일회용 데이터인 경우 

<br>
<br>

# 🗓️ 2026-02-06 (금)
### 🧩 예외 처리 할 만한 부분 찾기
- 내가 직접 사용자가 되어 실수할 수 있는 상황을 모두 가정해보기
- 프로그램 내부에서 내가 직접 만든 데이터가 아닌 '외부 데이터'가 들어오는 입구를 찾기
- 예) 키보드 입력값 (Scanner), URL/명령어 파라미터(Rq), 파일/데이터베이스 (삭제, 형식 깨짐 등)  

<br>
<br>

# 🗓️ 2026-02-07 (토)
## 🧩 프로그래머스 - 코딩테스트 50 days 강좌
### Day 0. 연산 횟수

> 코드 실행 시간 → 연산 횟수로 예상   
> 입력값(매개변수, 표준 입력) `n`에 따라 총 몇 번의 연산이 반복되는가? 

```python
def example_function(n):
    count = 0        # (1) 대입 연산 1회
    for i in range(n):
        count += 1   # (2) 덧셈 및 대입 연산 (n번 반복)
    return count     # (3) 반환 연산 1회
```

→ **입력값 n에 비례해서 급격하게 늘어나는 핵심 연산이 있다면, 영향력이 미미한 상수항은 무시하고 n의 연산 횟수만 고려한다. (n은 대명사처럼 사용됨)**  
→ 상수번 반복하는 횟수는 그냥 1이라고 하기도 한다.

<br>
<br>

# 🗓️ 2026-02-08 (일)
## 🧩 프로그래머스 - 코딩테스트 50 days 강좌
### Day 1. 해시(Hash) 및 문자열 해싱

해시(Hash): 임의의 데이터를 고정된 크기의 ‘정수’로 변환하는 것  
예) 문자열을 정수로 바꾸기  
→ 아스키 코드 등을 이용해 문자를 숫자로 매칭하여 처리   
(자바에선 문자(char)에 작은 따옴표를 붙이면 연산자를 만나는 순간 숫자로 자동 변환됨)

컴퓨터 데이터 저장 방식 : 정수/문자 → 이진수로 저장 , 이를 이용해 해시값을 생성  
문자가 아닌 문자열 → 문자열 전체를 특정 정수값으로 치환  
문자열이 길어질 경우 → 복잡한 해시를 직접 구현할 필요 없는 **해시맵(HashMap)**, **해시셋(HashSet)** (해시를 사용한 컨테이너 자료형, 언어별로 제공)

## 🧩 자바 OOP 아키택처 - 패키지 구조 : 계층형(레이어)과 도메인형
블로그 정리: [[자바 OOP 아키텍처] 패키지 구조 : 레이어 vs 도메인](https://velog.io/@hjy648012/%EC%9E%90%EB%B0%94-OOP-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98-%ED%8C%A8%ED%82%A4%EC%A7%80-%EA%B5%AC%EC%A1%B0-%EB%A0%88%EC%9D%B4%EC%96%B4-vs-%EB%8F%84%EB%A9%94%EC%9D%B8)  

이전까지는 컨트롤러, 서비스, 리포지터리로 이루어진 **레이어드 아키텍처와 MVC 패턴**을 이용해 패키지 구조를 설계했는데, 이번에 명언 게시판을 구현하며 **도메인 구조**를 도입했다.   
더 정확히 말하면 최상위는 도메인으로 나누고 그 안에서 레이어드로 설계하는 **도메인이 레이어를 품는 구조**이다.   
각각의 장단점을 비교하는 글도 인터넷 상에서 많이 봤는데, 둘 중 하나만을 택하기보단 양쪽을 적절히 섞어서 장점을 모두 활용하는 방식이 유용한 것 같다.  
또한 레이어와 도메인의 개념을 정리하며 그동안 혼동이 있었던 **레이어드 아키텍처와 MVC 패턴과의 관계** 또한 비교해 정리했다. 

<br>
<br>

# 🗓️ 2026-02-09 (월)
## 🧩 명언 게시판 리팩토링
- Rq 클래스 생성 이후 Controller 로직을 Service, Repository에 차례로 분리

- 패키지에 도메인 구조 도입 → system 패키지 생성 및 SystemController에 종료 로직 분리

- 이전까진 Application 클래스가 Controller의 run 메서드를 호출하면서 프로그램을 시작했는데, 가장 바깥에 Main 클래스를 추가하고 run 메서드를 Application 클래스로 옮겼다.   
  → **Application 클래스가 도메인별 컨트롤러를 호출해 관리하고, 각 컨트롤러는 자신이 속한 도메인 내부의 레이어드 클래스들을 관리하도록 했다.**

<br>
<br>

# 🗓️ 2026-02-10 (화)
### 🧩 리액트 복습 및 정리
→ [[React] 백엔드를 위한 리액트 간단 정리](https://velog.io/@hjy648012/React-%EB%B0%B1%EC%97%94%EB%93%9C%EB%A5%BC-%EC%9C%84%ED%95%9C-%EB%A6%AC%EC%95%A1%ED%8A%B8-%EA%B0%84%EB%8B%A8-%EC%A0%95%EB%A6%AC)

<br>
<br>

# 🗓️ 2026-02-11 (수)
## 🧩 프로그래머스 - 코딩테스트 50 days 강좌
### Day 2. 해시 셋
#### 배열 vs 해시 셋

    `{”apple”, “banana”, “list”}`

- 배열: 데이터를 순서대로 빠짐없이 저장, 순서 O, 인덱싱 가능
- 해시 셋: 해시를 사용, 데이터를 해싱해서 나온 정수의 위치에 저장, 순서 X, 인덱싱 불가능

#### 해시 셋의 특징

1. **데이터 중복 제거**
    
    셋은 **데이터 중복을 허용하지 X**, 이미 존재하는 값을 또 추가하면 데이터나 구조가 바뀌지 않고 boolean 값으론 false 반환 (자바 Collection들은 보통 자료구조가 실제로 변경됐는지를 boolean으로 알려줌)
    
    **배열 → 해시 셋으로 바꾸면 배열의 중복이 제거됨**
    
2. **빠른 데이터 검색**
    
    배열: 처음부터 나올때까지 하나하나 검사 vs 해시 셋: 찾으려는 데이터를 해싱한 값에 데이터가 존재하는지만 검사
    
    ```java
    import java.util.HashSet;
    
    class Solution {
    	public void solution() {
    		String[] arr = {"apple", "apple", "banana"};
    		
    		HashSet<String> set = new HashSet<>();
    		for (int i = 0; i < arr.length; i ++) {
    			set.add(arr[i]);
    		}
    		System.out.println(set.contains("hash"); // 출력: false
    	}
    }
    ```
<br>
<br>

# 🗓️ 2026-02-12 (목)
### 🧩 테일윈드란?
- 유틸리티 기반 CSS 프레임워크로, **별도의 CSS 파일을 많이 작성하지 않고 JSX 파일의 HTML 태그 안에 유틸리티 클래스를 직접 작성**하여 스타일을 적용한다. 이를 통해 더 직관적이고 간결한 코드를 작성할 수 있다.
- 메인 CSS 파일에는 Tailwind의 지시어(@tailwind base, components, utilities 등)를 작성하고, 이를 메인 JS 파일에서 import하여 사용한다.
- 빌드 과정에서 실제로 사용된 클래스만 모아 최종 CSS 파일이 생성되며, 브라우저는 이를 일반 CSS처럼 해석한다.
- 테일윈드 적용 예시 : 
```jsx
export function NaverLink() {
  return (
    <a 
      href="https://www.naver.com/" 
      target="_blank"
      className="text-green-600 hover:text-green-800 font-bold underline"
    >
      네이버
    </a>
  );
}
```

<br>
<br>

# 🗓️ 2026-02-13 (금)
### 🧩 스프링부트 프로젝트 환경 세팅 오류 해결
- **상황**:   
  `build.gradle.kts`파일 전체에 빨간 줄이 표시되었고, `Cannot access script base class 'org.gradle.kotlin.dsl.KotlinBuildScript'. Check your module classpath for missing or conflicting dependencies`라는 오류 메시지가 나타났다. 그러나 터미널에서 `./gradlew clean build`는 정상적으로 실행되었다.

- **해결 시도**:   
  - 인텔리제이 캐시 삭제
  - SDK 및 Gradle JVN 버전 확인
  - 프로젝트 삭제 후 재설치

- **진짜 원인**:  
  인텔리제이 업데이트를 하지 않았던 것. IntelliJ 2023.3 버전에서 JDK 25 + Spring Boot 4 조합을 사용할 때
Gradle Kotlin DSL 스크립트 모델을 IDE가 정상적으로 인식하지 못하는 문제가 발생한 것으로 보인다.
(CLI 빌드는 정상이나 IDE 내부 모델링 단계에서 오류 발생)

- **해결**:  
  IntelliJ를 최신 버전(2025.3)으로 업데이트한 뒤 프로젝트를 재임포트하자
`build.gradle.kts`의 빨간 줄이 사라지고 정상적으로 인식되었다. 

- **배운 점**:  
최신 JDK 및 Spring Boot 버전을 사용할 경우, IDE 버전과의 호환성도 함께 고려해야 함을 깨달았다.

<br>
<br>

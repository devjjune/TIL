# 🗓️ 2026-09-01 (화) ~ 2026-09-10 (목)
## 🧩 예약 취소 트랜잭션 분리와 동시성·실패 복구 개선
### 1. completeCancel() 상태 검증 추가
```java
Payment payment = paymentRepository.findById(paymentId)
        .orElseThrow(...);

payment.updateStatus(PaymentStatus.CANCELLED);
```
기존 코드는 결제 성공 후 현재 상태를 따로 확인하지 않고 `CANCELLED`로 바꾼다. 이렇게 했을 때 `prepareCancel()`에서 선점해놓은 결제가 아닌 다른 상태의 결제 내역이 변경될 가능성이 있다. 
```java
if (payment.getStatus() != PaymentStatus.CANCEL_IN_PROGRESS) {
            throw new CustomException(ErrorCode.INVALID_PAYMENT_STATUS);
        }
```
이 문제는 상태를 바꾸기 전 위의 검증 로직 하나를 더 추가함으로써 해결했다.    

결제가 다른 상태일 땐 취소가 안 되고, 오직 `CANCEL_IN_PROGRESS` 상태일 때만 취소할 수 있게 했다. 

---

### 2. 예약 취소 동시 요청 시 에러 처리 개선

기존 `prepareCancel()`에서는 `DONE` 상태의 결제를 바로 조회했다.
```java
paymentRepository.findByReservation_IdAndStatus(
        reservationId,
        PaymentStatus.DONE
)
```
문제는 동시에 두 개의 취소 요청이 들어오는 경우였다.

첫 번째 요청이 먼저 결제 상태를 `DONE → CANCEL_IN_PROGRESS`로 변경하면, 두 번째 요청에서는 더 이상 `DONE` 상태의 결제를 찾을 수 없다.  

기존 코드에서는 이 경우 결제 자체가 없는 것으로 판단해서 `PAYMENT_NOT_FOUND` 예외가 발생했다. 하지만 실제로는 결제가 없는 것이 아니라, 이미 다른 요청이 취소를 진행 중인 상태였다.  

따라서 결제를 `DONE` 조건으로 바로 조회하지 않고, 해당 예약에 연결된 모든 결제를 먼저 조회하도록 변경하고 그 후 상태를 나누어 판단했다. 
```java
List<Payment> payments =
        paymentRepository.findAllByReservation_Id(reservationId);
```
- `CANCEL_IN_PROGRESS`가 존재 → 이미 결제 취소가 진행 중
- `DONE`이 존재 → 현재 요청이 취소 진행 가능
- 둘 다 없음 → 취소 가능한 결제가 없음

이를 통해 동시 취소 요청이 들어왔을 때 기존의 `PAYMENT_NOT_FOUND` 대신 `PAYMENT_CANCEL_IN_PROGRESS`를 반환하도록 개선했다.

즉, 이번 변경의 목적은 동시성 자체를 새로 해결한 것이 아니라, 이미 비관적 락으로 중복 취소를 막고 있는 상황에서 **두 번째 요청에게 현재 상태에 맞는 정확한 에러를 반환하도록 개선한 것**이다.

---

### 3. 단위/통합/동시성 테스트 추가

#### `ReservationCancelTest` Toss 실패 복구 테스트 보강 (API/통합 테스트)
  - 정상 취소 성공 시 Payment와 Reservation의 DB 상태까지 검증
  - 예약 취소 실패(Toss 취소 실패) 시 결제 상태가 DONE으로 복구되는지 검증
  - 이미 `CANCEL_IN_PROGRESS`인 결제를 취소 요청하면 실패하는지 검증

#### `ReservationCancelTransactionServiceTest` 취소 트랜잭션 상태 전이 테스트
  - `prepareCancel()` 성공: `DONE → CANCEL_IN_PROGRESS`
    - 이미 `CANCEL_IN_PROGRESS`인 결제가 있으면 `PAYMENT_CANCEL_IN_PROGRESS` 에러 반환
  - `completeCancel()` 성공: `CANCEL_IN_PROGRESS → CANCELLED`
    - `completeCancel()` 호출 시 Payment가 `CANCEL_IN_PROGRESS`가 아니면 `PAYMENT_INVALID_STATUS` 에러 반환
  - `releaseCancel()` 성공: `CANCEL_IN_PROGRESS → DONE`
    - `releaseCancel()` 호출 시 이미 다른 상태면 함부로 `DONE`으로 바꾸지 않는지

#### `ReservationCancelConcurrencyTest` (실제 동시성 테스트)
처음에는 같은 예약에 취소 요청 2개를 거의 동시에 보내고 Toss mock에서 `Thread.sleep(300)`을 사용했다. 
```java
willAnswer(invocation -> {
    Thread.sleep(300);
    return null;
}).given(tossPaymentClient)
        .cancel(anyString(), anyString());
```
의도한 결과는 다음과 같았다.
```
첫 번째 요청
→ DONE → CANCEL_IN_PROGRESS
→ Toss 호출 중 대기

두 번째 요청
→ CANCEL_IN_PROGRESS 확인
→ PAYMENT_CANCEL_IN_PROGRESS
```
하지만 `Thread.sleep()`은 첫번째 요청을 일정 시간 지연시킬 뿐, 두번째 요청이 그 시간 안에 반드시 `prepareCancel()`까지 도달한다는 것을 보장하지 않는다. 

실제로 정상 요청이 너무 빨리 처리되어서 아래와 같은 상황이 발생했다. 
```
첫 번째 요청
→ 취소 완료
→ Reservation = CANCELLED

두 번째 요청
→ 이미 취소된 예약 조회
→ RESERVATION_CANNOT_BE_CANCELLED
```
따라서 `CountDownLatch`를 사용해 첫번째 요청을 정확히 Toss 호출 지점에서 멈추도록 수정했다. 
```java
CountDownLatch tossEnteredLatch = new CountDownLatch(1);
CountDownLatch releaseTossLatch = new CountDownLatch(1);
```
흐름은 다음과 같다. 
```
첫 번째 요청
→ prepareCancel()
→ DONE → CANCEL_IN_PROGRESS
→ Toss 호출 지점 도착
→ releaseTossLatch.await()에서 대기

메인 테스트 스레드
→ 첫 번째 요청이 Toss까지 도착한 것을 확인

두 번째 요청
→ prepareCancel()
→ Payment = CANCEL_IN_PROGRESS 확인
→ PAYMENT_CANCEL_IN_PROGRESS

메인 테스트 스레드
→ 두 번째 요청 차단 확인
→ 첫 번째 요청의 Toss 대기 해제

첫 번째 요청
→ completeCancel()
→ Reservation = CANCELLED
→ Payment = CANCELLED
```
최종적으로 다음을 검증했다. 
- Toss 취소는 한 번만 호출
- 한 요청만 정상 취소
- 다른 요청은 `PAYMENT_CANCEL_IN_PROGRESS`로 차단
- 최종 DB 상태는 `Reservation = CANCELLED`, `Payment = CANCELLED`

---

### 4. 트랜잭션을 분리했는데도 비관적 락이 해제되지 않던 문제

동시성 테스트를 실행하는 과정에서 다음 에러가 발생했다.

```text
Timeout trying to lock table "RESERVATION"

is locked by tx 1 and can not be updated by tx 2
within allocated time interval 2000 ms
```

의도한 구조는 다음과 같았다.

```text
prepareCancel()
→ 트랜잭션 시작
→ Reservation 비관적 락 획득
→ DONE → CANCEL_IN_PROGRESS
→ commit
→ 락 해제

Toss 호출
→ 트랜잭션 없음

completeCancel()
→ 별도 트랜잭션 시작
→ 취소 완료 상태 반영
```

하지만 실제로는 첫 번째 요청이 Toss 호출 중에도 Reservation 락을 계속 가지고 있었고, 두 번째 요청이 같은 Reservation의 비관적 락을 얻지 못해 타임아웃이 발생했다.

원인은 `ReservationService`의 클래스 레벨 `@Transactional`이었다.

```java
@Service
@RequiredArgsConstructor
@Transactional
@Slf4j
public class ReservationService {
```

`cancelReservation()` 자체에는 `@Transactional`이 없었지만, 클래스 레벨의 `@Transactional`이 적용되어 메서드 전체가 하나의 트랜잭션으로 실행되고 있었다.

또한 `prepareCancel()`의 `@Transactional` 기본 전파 속성은 `REQUIRED`이므로, 이미 바깥 트랜잭션이 존재하면 새로운 트랜잭션을 생성하지 않고 기존 트랜잭션에 참여한다.

따라서 실제 동작은 다음과 같았다.

```text
cancelReservation() 트랜잭션 시작

→ prepareCancel()
   기존 트랜잭션에 참여
   Reservation 비관적 락 획득

→ prepareCancel() 종료
   메서드는 끝났지만 트랜잭션은 종료되지 않음
   락도 유지

→ Toss 호출

→ completeCancel()

→ cancelReservation() 종료
   최종 commit
   락 해제
```

즉 메서드는 분리했지만 실제 트랜잭션 경계는 분리되지 않은 상태였다.

또한 `PaymentCancelService`에 적용한 `NOT_SUPPORTED`는 기존 트랜잭션을 종료하는 것이 아니라 잠시 중단한다.

```text
suspend ≠ commit
suspend ≠ lock 해제
```

따라서 클래스 레벨 `@Transactional`을 제거하고, 실제로 트랜잭션이 필요한 메서드에 개별적으로 `@Transactional`을 적용했다.

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ReservationService {
```

```java
@Transactional
public ReservationResponse create(...) {
    ...
}
```

```java
@Transactional
public ReservationResponse createTimeDealReservation(...) {
    ...
}
```

조회 메서드는 기존처럼 `@Transactional(readOnly = true)`를 유지했다.

반면 `cancelReservation()`에는 트랜잭션을 적용하지 않았다.

수정 후에는 의도한 대로:

```text
prepareCancel() TX
→ commit
→ 락 해제

Toss 호출
→ TX 없음

completeCancel() TX
→ commit
```

구조가 실제로 적용되었고, 동시성 테스트도 정상 통과했다.

이번 과정을 통해 **메서드 분리와 트랜잭션 분리는 동일하지 않으며, 실제 트랜잭션 경계는 Spring의 전파 속성과 바깥 트랜잭션 존재 여부까지 확인해야 한다**는 점을 알게 되었다.

---

### 5. Toss 성공 후 `completeCancel()` 실패 복구 전략

Toss cancel이 성공했지만 우리 DB에서 결제 취소 완료 상태를 저장하던 중 에러가 발생한 경우, 실제 결제는 취소되었지만 DB에서는 결제 상태가 `CANCEL_IN_PROGRESS`, 예약 상태가 `CONFIRMED`로 남아있을 수 있다.

`releaseCancel()`은 Toss 자체가 실패했을 때만 사용한다.

```text
1. 우리 DB: "취소 시작"
   DONE → CANCEL_IN_PROGRESS

2. Toss: 실제 결제 취소

3. 우리 DB: "취소 완료"
   CANCEL_IN_PROGRESS → CANCELLED
   Reservation CONFIRMED → CANCELLED
```

문제는 2번까지 성공했지만 3번에서 실패한 경우다.

이때 `releaseCancel()`을 실행해 `DONE`으로 되돌리면:

```text
Toss = 이미 취소됨
DB Payment = DONE
```

이라는 더 잘못된 상태가 된다.

따라서 Toss 성공 이후에는 `releaseCancel()`을 호출하지 않고 `completeCancel()`만 재시도하도록 했다.

```text
Toss 성공
→ completeCancel()

성공
→ CANCELLED

실패
→ completeCancel() 재시도
→ 최대 3회
```

모든 재시도가 실패하면 `CANCEL_IN_PROGRESS` 상태를 유지하고 예외와 로그를 남긴다.

```text
Toss = CANCELLED
DB Payment = CANCEL_IN_PROGRESS
```

완전한 정합성 복구는 아니지만, 최소한 실제로 취소된 결제를 다시 `DONE`으로 되돌리는 잘못된 복구는 방지할 수 있다.

또한 `completeCancel()` 내부에 있던 채팅방 종료 로직을 분리했다.

기존에는:

```java
payment.updateStatus(PaymentStatus.CANCELLED);
reservation.updateStatus(ReservationStatus.CANCELLED);

chatService.closeByReservationId(reservationId);
```

처럼 채팅방 종료까지 같은 트랜잭션 안에 있었기 때문에, 채팅방 종료에 실패하면 결제와 예약 상태 변경까지 함께 롤백될 수 있었다.

따라서 `completeCancel()`에서는 결제/예약 상태만 확정하고, 채팅방 종료는 그 이후 별도로 수행하도록 변경했다.

---

### 6. Toss 성공 후 DB 반영 재시도 테스트 추가

최소 복구 전략을 검증하기 위해 다음 두 가지 테스트를 추가했다.

#### 1. `completeCancel()` 1회 실패 후 재시도 성공

```text
prepareCancel()
→ Toss 성공
→ completeCancel() 1차 실패
→ completeCancel() 재시도
→ 성공
```

검증:

* Toss 취소 호출은 1회
* `completeCancel()`은 2회 호출
* `releaseCancel()`은 호출되지 않음
* 최종적으로 취소 응답 정상 반환

#### 2. `completeCancel()` 재시도 모두 실패

```text
prepareCancel()
→ Toss 성공
→ completeCancel() 실패
→ 재시도
→ 총 3회 모두 실패
```

검증:

* Toss 취소 호출은 1회
* `completeCancel()`은 최대 재시도 횟수만큼 호출
* `releaseCancel()`은 호출되지 않음
* 예외 반환

즉 Toss가 이미 성공한 이후에는 **외부 결제 취소를 다시 호출하지 않고 DB 반영만 재시도하며, 실패했다고 `DONE` 상태로 되돌리지 않는 것**을 확인했다.

---

### 전체 흐름 정리

최종 예약 취소 흐름은 다음과 같다.

```text
prepareCancel()
→ Reservation 비관적 락 획득
→ Payment DONE → CANCEL_IN_PROGRESS
→ commit
→ 락 해제

↓

Toss cancel()

├─ Toss 실패
│  → releaseCancel()
│  → CANCEL_IN_PROGRESS → DONE
│
└─ Toss 성공
   ↓
   completeCancel()

   ├─ 성공
   │  → Payment CANCELLED
   │  → Reservation CANCELLED
   │
   └─ 실패
      → completeCancel() 재시도
      → 최대 3회

      ├─ 재시도 성공
      │  → CANCELLED
      │
      └─ 모두 실패
         → CANCEL_IN_PROGRESS 유지
         → 로그 및 예외 처리
```

이번 리팩토링에서는 단순히 Toss API 호출을 트랜잭션 밖으로 빼는 것에서 끝나지 않고, 트랜잭션 경계가 실제로 어떻게 적용되는지 확인하고, 동시 요청과 외부 API 실패 및 DB 반영 실패까지 고려해 예약 취소 흐름을 보완했다.

<br>
<br>

# 🗓️ 2026-09-06 (토)
## 🧩 `ReservationCancelConcurrencyTest`의 스레드 동기화 구조
해당 테스트 클래스에는 세 개의 스레드가 관여하고 있다.  
→ **메인 테스트 스레드 1개**  
→ **ExecutorService 내부 스레드 2개**

```text
메인 테스트 스레드
→ for문 실행
→ 작업 2개 submit
→ readyLatch.await()
→ startLatch.countDown()
→ doneLatch.await()

ExecutorService의 스레드풀 (submit(() -> { ... }) 내부 코드 실행)
├─ Thread 1 → 첫 번째 취소 요청 작업 실행
└─ Thread 2 → 두 번째 취소 요청 작업 실행
```
하나의 스레드만 실행되고 있을 땐 코드가 위에서 아래로 내려가며 순차적으로 실행되지만, 여러 스레드가 동시에 실행될 땐 **서로 다른 스레드 사이의 실행 순서는 코드의 순서와 같지 않을 수 있다.**

따라서 각 스레드가 실행되어 요청 직전의 준비 지점까지 도착할 때마다 `readyLatch.countDown();`으로 카운트를 하나씩 감소시키고, 해당 스레드는 `startLatch.await();`에서 대기한다.  

그러는 동안 메인 스레드는 `readyLatch.await();`로 `submit` 내부의 두 스레드가 모두 준비 지점까지 도착할 때까지 기다린다.
(`readyLatch`가 `countDown()`되어 0이 될 때까지 기다리는 것임)

그리고 두 스레드가 모두 준비되면 메인 스레드가 `startLatch.countDown();`을 호출하여, `startLatch.await()`에서 대기하고 있던 두 스레드를 동시에 실행시킨다.

마지막으로 메인 스레드는 `doneLatch.await();`으로 두 요청이 모두 끝날 때까지 기다린 뒤 테스트 결과를 검증한다.

※ `CountDownLatch`의 `countDown()`과 `await()`은 보통 서로 맞물려 사용된다고 생각하면 이해가 쉽다.  
`countDown()`으로 카운트를 1씩 줄이고, `await()`은 카운트가 0이 될 때까지 현재 스레드를 대기시킨다.
단, 두 메서드가 반드시 1:1로 호출되는 것은 아니다.

<br>
<br>

# 🗓️ 2026-09-17 (목)
## 🧩 Docker 기반 CI/CD 배포 파이프라인 복습
- CI (Continuous Integration) = 코드 변경 시 **빌드 + 테스트** 등을 자동으로 수행
- CD (Continuous Delivery/Deployment) = CI를 통과한 결과물을 **배포**하는 과정을 자동화

```
Dockerfile (이미지 설정 작성)
   ↓ build
Docker Image 
   ↓ run
Container 생성 및 실행 (실행 중인 앱 환경)
```
- Docker Compose: 여러 개의 컨테이너를 한 번에 관리할 수 있는 도구
- Docker Compose를 실행 → compose.yml에 정의된 컨테이너들이 한 번에 실행
    - (기존 이미지 없으면) 이미지 빌드/다운로드 → 컨테이너 생성 → 실행


**캠핑가잣 프로젝트 구조**
```
코드 작성
   ↓
GitHub push
   ↓
────── CI ──────
Gradle Build
테스트 실행
Checkstyle
JaCoCo
   ↓
────── CD ──────
Docker Image 생성
서버에 전달
EC2에서 새 버전 실행
   ↓
────── 배포 완료 ──────
사용자가 접속 가능
```

#### 배포 실습 과정
: 로컬 실행 확인 → 2. AWS 인프라 구성 → 3. 수동 배포 성공 → 4. CI/CD 연결
```
[1] 로컬
Docker Compose로 프로젝트 정상 실행
        ↓
[2] AWS
EC2 생성
RDS(MySQL) 생성
보안그룹/환경변수 설정
        ↓
[3] 수동 배포
EC2에 접속
Docker 설치
코드/이미지 가져오기
Spring Boot 컨테이너 실행
        ↓
API 요청해서 정상 동작 확인
        ↓
[4] CI/CD
GitHub Actions
        ↓
Build + Test            ← CI
        ↓
Docker Image Build
        ↓
EC2에 새 버전 배포      ← CD
```

<br>

## 🧩 캠핑가잣 프로젝트 CI/CD 배포 흐름 (GitHub Actions 이용)

```
feature/refactor 브랜치
        │
        │ PR → dev
        ▼
┌──────────────────────────────┐
│ ci.yml                       │
│                              │
│ 테스트                         │
│ JaCoCo 커버리지                │
│ Checkstyle                   │
│ PR에 결과 코멘트                │
└──────────────────────────────┘
        │
        │ PR merge
        ▼
       dev
        │
        │ push 발생
        ▼
┌──────────────────────────────┐
│ deploy.yml                   │
│                              │
│ ① 다시 테스트                  │
│       ↓                      │
│ ② Docker Image build        │
│       ↓                      │
│ ③ Docker Hub push           │
│       ↓                      │
│ ④ EC2 SSH 접속               │
│       ↓                      │
│ ⑤ EC2가 최신 Image pull       │
│       ↓                      │
│ ⑥ Container 재생성            │
│       ↓                      │
│ ⑦ Health Check              │
└──────────────────────────────┘
```

### 1. 로컬 개발 환경

로컬에서는 개발/디버깅 편의를 위해 Spring Boot를 IDE에서 직접 실행하고,
Docker Compose로 인프라만 실행한다.  

- Spring Boot → IntelliJ에서 실행
- MySQL → Docker
- Prometheus → Docker
- Grafana → Docker

(반면 운영에서는 Spring Boot를 Docker 이미지로 만들어 EC2의 컨테이너에서 실행한다. 따라서 로컬과 운영 환경에서 Docker로 관리하는 대상은 다를 수 있다.)

### 2. CI - `ci.yml`

`dev` 브랜치로 PR이 생성되거나 업데이트되면 CI가 실행된다.

GitHub Actions Runner라는 임시 Ubuntu 환경에서 다음 작업을 수행한다.

1. Repository 코드 Checkout
2. JDK 설치
3. Gradle Test 실행
4. JaCoCo 테스트 커버리지 측정
5. Checkstyle 검사
6. 결과를 PR에 표시

즉, CI는 코드 변경 후 테스트와 코드 품질 검사를 자동으로 수행하는 과정이다.

### 3. CD - `deploy.yml`

PR이 `dev`에 merge되면 `push` 이벤트가 발생하고 `deploy.yml`이 실행된다.

#### 전체 흐름
```
dev push
→ Test
→ Docker Image Build
→ Docker Hub Push
→ EC2 접속
→ Docker Image Pull
→ Container 재생성
→ Health Check
```
`needs: test`를 사용하여 테스트가 성공한 경우에만 배포가 진행된다.

### 4. Docker Image는 어디서 만들어지는가?

Docker Image는 EC2가 아니라 GitHub Actions Runner에서 만들어진다.
```
GitHub Actions Runner
→ `docker build` → Docker Image 생성
→ `docker push` → Docker Hub에 Image 저장

EC2
→ `docker pull` → Docker Hub에서 Image 다운로드
→ `docker compose up` → Container 생성 및 실행
```

따라서 EC2에는 Spring Boot 소스코드가 없어도 된다.
실행에 필요한 Docker Image와 `docker-compose.prod.yml`만 있으면 된다.

### 5. 운영 환경변수 관리

DB 비밀번호, JWT Secret 등의 민감정보는 코드에 저장하지 않고 GitHub Secrets에 저장한다.
```
GitHub Secrets
→ 배포 시 SSH를 통해 EC2로 전달
→ EC2의 `.env` 생성
→ `docker-compose.prod.yml`에서 환경변수로 사용
```

### 6. 배포 후 Health Check

Container를 실행했다고 바로 배포 성공으로 처리하지 않는다.

Spring Boot Actuator의 `/actuator/health`를 일정 간격으로 호출하여 애플리케이션이 실제로 정상 실행됐는지 확인한다.

현재 구조는 Health Check 실패를 감지할 수 있지만,
이전 버전으로 자동 Rollback하는 기능은 없다.

### 7. 전체 구조
```
로컬
→ 코드 작성
→ PR

GitHub Actions
→ CI(Test / JaCoCo / Checkstyle)
→ Docker Image Build

Docker Hub
→ Docker Image 저장

EC2
→ Image Pull
→ Container 실행

RDS
→ 운영 DB
```

<br>
<br>
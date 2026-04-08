# Spring Boot + Virtual Thread — 실전 통합

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `spring.threads.virtual.enabled=true` 설정이 어떤 컴포넌트를 Virtual Thread로 전환하는가?
- `@Async`와 Virtual Thread를 조합하는 올바른 방법은?
- HikariCP 커넥션 풀이 Virtual Thread와 호환되는 방식은?
- Pinning 발생 지점을 Spring Boot 애플리케이션에서 찾는 실용적인 방법은?
- Virtual Thread 전환 후 성능 차이를 어떻게 측정하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring Boot 3.2부터 한 줄의 설정으로 모든 요청 처리 스레드를 Virtual Thread로 전환할 수 있다. 하지만 아무 것도 모르고 전환하면 Pinning, ThreadLocal 이슈, 잘못된 스레드 풀 설정으로 오히려 성능이 떨어질 수 있다. 이 문서는 안전하고 효과적으로 VT를 도입하는 실전 가이드다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 설정 하나만 바꾸면 모든 것이 자동 최적화된다는 기대
  spring.threads.virtual.enabled=true  # 이거면 다 되겠지?
  → 내부 synchronized Pinning은 여전히 존재
  → JDBC 드라이버 호환성 확인 필요
  → ThreadLocal 사용 코드 검토 필요

실수 2: @Async를 기존 방식으로 사용
  @Async
  public CompletableFuture<String> asyncTask() { ... }
  // 기본 설정: SimpleAsyncTaskExecutor (각 호출마다 OS 스레드 생성!)
  // → VT 환경에서 @Async는 별도 설정 필요

실수 3: HikariCP 커넥션 수 조정 없이 전환
  # 기존: 스레드 풀 200개 → 커넥션 풀 200개로 설정
  # VT 전환 후: 수천 VT → 커넥션 수는 여전히 제한됨
  # → 커넥션 고갈 후 대기 (정상 동작이나 의도 확인 필요)
  spring.datasource.hikari.maximum-pool-size=10
  # VT 환경에서는 커넥션 수를 줄여도 처리량 유지 (VT가 대기 중 다른 작업)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```yaml
# application.yml 기본 설정

# 1. Virtual Thread 활성화 (Spring Boot 3.2+)
spring:
  threads:
    virtual:
      enabled: true  # Tomcat 요청 처리 + @Async 기본 executor → VT

# 2. HikariCP 설정 (VT 환경에 맞게 조정)
spring:
  datasource:
    hikari:
      maximum-pool-size: 20         # VT는 커넥션 대기 중 Unmount → 적은 수로도 OK
      connection-timeout: 30000      # 30초 (커넥션 대기 타임아웃)
      idle-timeout: 600000
      max-lifetime: 1800000

# 3. Tomcat 설정 (VT 사용 시 스레드 풀 크기 의미 없음)
server:
  tomcat:
    threads:
      max: 200   # VT 활성화 시 이 값이 무시됨 (VT는 무제한)
```

```java
// Spring Boot + VT 실전 설정

@Configuration
public class VirtualThreadConfig {

    // @Async용 VT 실행기
    @Bean(name = "virtualThreadExecutor")
    public Executor virtualThreadExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }

    // 기본 @Async executor를 VT로 설정
    @Bean
    public TaskExecutor applicationTaskExecutor() {
        return new VirtualThreadTaskExecutor("app-vt-");
    }
}

@Service
public class UserService {

    // @Async + VT (spring.threads.virtual.enabled=true 시 자동)
    @Async
    public CompletableFuture<User> findUserAsync(Long id) {
        return CompletableFuture.completedFuture(userRepository.findById(id).orElseThrow());
    }

    // 여러 VT 병렬 실행 (StructuredTaskScope 활용)
    public UserProfile getFullProfile(Long userId) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            var userTask    = scope.fork(() -> userRepository.findById(userId));
            var ordersTask  = scope.fork(() -> orderRepository.findByUserId(userId));
            var reviewsTask = scope.fork(() -> reviewRepository.findByUserId(userId));

            scope.join().throwIfFailed();  // 셋 다 완료될 때까지 대기

            return new UserProfile(
                userTask.get(),
                ordersTask.get(),
                reviewsTask.get()
            );
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. spring.threads.virtual.enabled=true의 내부 동작

```
Spring Boot 3.2 VT 설정의 영향 범위:

① Tomcat 요청 처리 스레드:
   기본: ThreadPoolExecutor (OS 스레드 풀, 기본 200개)
   VT 활성화: VirtualThreadExecutor (요청마다 새 VT 생성)
   → TomcatVirtualThreadsProtocolHandlerCustomizer가 자동 적용

② @Async 기본 Executor:
   기본: SimpleAsyncTaskExecutor (OS 스레드, 매번 새 생성)
   VT 활성화: VirtualThreadTaskExecutor (매번 새 VT 생성)
   → AsyncConfigurationSelector가 VT executor 자동 사용

③ Spring MVC 비동기 처리:
   DeferredResult, Callable, CompletableFuture + @Async
   모두 VT executor로 실행

④ Spring WebFlux:
   영향 없음 (자체 이벤트 루프 사용)
   VT와 WebFlux 혼용은 일반적으로 불필요 (둘 다 비슷한 목적)

영향 없는 영역:
   @Scheduled 작업: 별도 executor (ThreadPoolTaskScheduler)
   Kafka 컨슈머: 자체 스레드 관리
   배치 처리: Spring Batch 자체 설정

auto-configuration 코드 흐름:
  VirtualThreadsProperties:
    spring.threads.virtual.enabled → VirtualThreadsAutoConfiguration
  VirtualThreadsAutoConfiguration:
    TomcatVirtualThreadsProtocolHandlerCustomizer 등록
    SimpleAsyncTaskExecutorBuilder 설정
    ...
```

### 2. HikariCP와 Virtual Thread 상호작용

```
HikariCP 커넥션 풀 + VT 동작:

VT가 DB 쿼리 실행 시:
  ① JDBC 드라이버가 커넥션 풀에서 커넥션 요청
  ② 커넥션 여유분 있음 → 즉시 반환
  ③ 커넥션 고갈 → HikariCP의 connectionTimeout 동안 대기
     대기 내부: LockSupport.parkNanos() → VT Unmount!
     → 캐리어는 다른 VT 실행 (커넥션 기다리는 동안 CPU 낭비 없음)
  ④ 커넥션 반납 → 대기 VT unpark → 커넥션 획득

이것이 VT + 적은 커넥션으로도 높은 처리량이 가능한 이유:
  500 VT가 동시에 DB 쿼리 실행
  커넥션 풀 20개
  480 VT는 커넥션 대기 중 Unmount (캐리어 해방)
  20 VT만 실제 쿼리 실행
  쿼리 완료 → 커넥션 반납 → 대기 VT 하나씩 실행

커넥션 수 설정 가이드:
  VT 이전: threads.max == hikari.max-pool-size (1:1 권장)
  VT 이후: hikari.max-pool-size = DB 서버 최적 연결 수
           (보통 CPU 코어 수 × 2~3, Hikari 공식 권장)
           예: DB 서버 8코어 → 16~24개 커넥션

HikariCP 내부의 Pinning 주의:
  구형 HikariCP: ConcurrentBag에 synchronized 있음 → Pinning
  HikariCP 5.1.0+: synchronized 최소화 → VT 친화적
  → 최신 버전 사용 필수
```

### 3. StructuredTaskScope 실전 활용

```java
// Java 21 StructuredTaskScope: 구조화된 동시성
// 여러 VT 작업을 안전하게 조율

// 패턴 1: 모두 성공해야 할 때 (ShutdownOnFailure)
public ProductDetails getProductDetails(Long productId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        // 세 VT가 동시에 실행
        var product   = scope.fork(() -> productRepo.findById(productId));
        var inventory = scope.fork(() -> inventoryService.check(productId));
        var reviews   = scope.fork(() -> reviewService.getReviews(productId));

        scope.join()          // 셋 다 완료까지 대기
             .throwIfFailed(); // 하나라도 실패 시 예외

        // 모두 성공
        return new ProductDetails(product.get(), inventory.get(), reviews.get());
    }
    // scope.close(): 아직 실행 중인 VT 취소 (하나 실패 시)
}

// 패턴 2: 가장 빠른 하나만 있으면 될 때 (ShutdownOnSuccess)
public String getFastestPrice(Long productId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
        // 여러 가격 API 동시 호출, 가장 빠른 것 사용
        scope.fork(() -> priceApi1.getPrice(productId));
        scope.fork(() -> priceApi2.getPrice(productId));
        scope.fork(() -> priceApi3.getPrice(productId));

        scope.join();  // 하나라도 성공하면 나머지 취소
        return scope.result();  // 가장 빠른 결과
    }
}

// 패턴 3: ScopedValue와 함께 컨텍스트 전달
static final ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();

public void processWithContext(String reqId) throws Exception {
    ScopedValue.where(REQUEST_ID, reqId).run(() -> {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            // 자식 VT에 REQUEST_ID 자동 상속
            scope.fork(() -> {
                log.info("자식 VT: reqId={}", REQUEST_ID.get()); // reqId 자동 전파
                return doWork1();
            });
            scope.fork(() -> {
                log.info("자식 VT: reqId={}", REQUEST_ID.get()); // reqId 자동 전파
                return doWork2();
            });
            scope.join().throwIfFailed();
        } catch (Exception e) { throw new RuntimeException(e); }
    });
}
```

### 4. Pinning 진단 및 수정 실전 워크플로우

```bash
# Step 1: VT 활성화 후 Pinning 스캔
java -Djdk.tracePinnedThreads=full \
     -jar myapp.jar

# 출력 예시 (Pinning 발생 시):
# Thread[#30,ForkJoinPool-1-worker-1,5,CarrierThreads]
#     com.example.dao.UserDao.findById(UserDao.java:45) <== monitors:1
#         com.example.dao.UserDao@7f3d (owner: 0)

# → UserDao.java:45 에 synchronized 블록이 있고 그 내부에서 블로킹 발생

# Step 2: JFR으로 운영 환경 Pinning 측정
jcmd $(pgrep -f myapp.jar) JFR.start \
     duration=60s \
     filename=/tmp/pinning.jfr \
     settings=profile

# Step 3: JFR 분석
jfr print --events jdk.VirtualThreadPinned /tmp/pinning.jfr | \
    grep -E "(stackTrace|duration|method)" | head -100

# Step 4: 수정 후 확인
# synchronized → ReentrantLock으로 교체
# JFR 재수집 → jdk.VirtualThreadPinned 이벤트 없음 확인

# Step 5: 성능 지표 비교
# wrk 또는 k6로 부하 테스트
wrk -t4 -c100 -d30s http://localhost:8080/api/users
# VT 이전 vs 이후 처리량(RPS), 응답 시간(p99) 비교
```

---

## 💻 실전 실험

### 실험 1: Spring Boot VT 설정 전후 비교

```java
// 부하 테스트용 Controller
@RestController
@RequestMapping("/api")
public class TestController {

    @Autowired
    private UserRepository userRepository;

    // 동기 API (VT 효과 측정)
    @GetMapping("/user/{id}")
    public User getUser(@PathVariable Long id) throws InterruptedException {
        // DB 쿼리 시뮬레이션 (50ms I/O)
        Thread.sleep(50);
        return userRepository.findById(id).orElseThrow();
    }

    // 현재 스레드 정보 (VT 여부 확인)
    @GetMapping("/thread-info")
    public Map<String, Object> threadInfo() {
        Thread t = Thread.currentThread();
        return Map.of(
            "name", t.getName(),
            "isVirtual", t.isVirtual(),
            "id", t.threadId()
        );
    }

    // 병렬 API 호출 (StructuredTaskScope)
    @GetMapping("/profile/{userId}")
    public UserProfile getProfile(@PathVariable Long userId) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            var user    = scope.fork(() -> { Thread.sleep(30); return findUser(userId); });
            var orders  = scope.fork(() -> { Thread.sleep(50); return findOrders(userId); });

            scope.join().throwIfFailed();
            return new UserProfile(user.get(), orders.get());
        }
        // 순차 실행: 80ms, 병렬 VT: ~50ms
    }
}
```

```yaml
# 테스트 설정 파일 1: VT 비활성화
# application-no-vt.yml
spring:
  threads:
    virtual:
      enabled: false
server:
  tomcat:
    threads:
      max: 200
```

```yaml
# 테스트 설정 파일 2: VT 활성화
# application-vt.yml
spring:
  threads:
    virtual:
      enabled: true
  datasource:
    hikari:
      maximum-pool-size: 20  # 줄여도 됨
```

### 실험 2: JFR 기반 Pinning 탐지 자동화

```bash
#!/bin/bash
# pinning_check.sh: Pinning 발생 여부 자동 체크

APP_PID=$(pgrep -f "java.*myapp")

# 30초간 JFR 기록
jcmd $APP_PID JFR.start duration=30s filename=/tmp/check.jfr

sleep 35

# Pinning 이벤트 수 카운트
PINNING_COUNT=$(jfr print --events jdk.VirtualThreadPinned /tmp/check.jfr 2>/dev/null | \
    grep -c "VirtualThreadPinned")

echo "Pinning 이벤트 수 (30초): $PINNING_COUNT"

if [ $PINNING_COUNT -gt 0 ]; then
    echo "⚠️ Pinning 발생! 상세 분석:"
    jfr print --events jdk.VirtualThreadPinned /tmp/check.jfr 2>/dev/null | \
        grep -A 10 "startTime"
else
    echo "✅ Pinning 없음"
fi
```

---

## 📊 성능/비용 비교

```
Spring Boot VT 전환 전후 처리량 비교 (I/O 바운드 API):

시나리오: 50ms DB 쿼리, 동시 요청 500개

설정                    | RPS      | p99 응답시간  | 메모리
───────────────────────┼──────────┼─────────────┼───────────
OS 스레드 풀 200개        | ~2,000   | ~300ms      | ~1GB
OS 스레드 풀 500개        | ~10,000  | ~60ms       | ~2.5GB (위험)
Virtual Thread         | ~10,000  | ~55ms       | ~500MB
VT + Pinning 있음       | ~2,000   | ~300ms      | ~500MB

VT 장점이 드러나는 조건:
  I/O 대기 비율 > 80%
  동시 요청 수 >> 스레드 풀 크기
  메모리 효율 중요

VT 장점이 적은 조건:
  CPU 바운드 (이미지 처리, 암호화)
  Pinning이 많은 레거시 코드
  이미 Reactive(WebFlux)로 최적화된 시스템

HikariCP 설정 변화:
  이전: maximum-pool-size = OS 스레드 수와 비슷하게 (예: 200)
  이후: maximum-pool-size = DB 최적 연결 수 (예: 20~50)
        더 적은 커넥션으로 더 높은 처리량 가능
```

---

## ⚖️ 트레이드오프

```
Spring Boot VT 도입 트레이드오프:

이점:
  설정 한 줄로 Tomcat 요청 처리 VT 전환
  @Async가 VT executor 자동 사용
  기존 동기 코드 그대로 유지
  I/O 바운드 API 처리량 대폭 향상
  메모리 효율 (OS 스레드 수 감소)

위험:
  Pinning (synchronized 내 I/O): 진단 및 수정 필요
  ThreadLocal 사용 코드 검토 필요
  서드파티 라이브러리 호환성 확인 필요
  운영 모니터링 지표 변경 (스레드 수 → VT 수)

마이그레이션 전략:
  1단계: 개발 환경에서 VT 활성화
  2단계: -Djdk.tracePinnedThreads=full 로 Pinning 스캔
  3단계: Pinning 지점 수정 (ReentrantLock 교체)
  4단계: 부하 테스트 (wrk, k6)로 처리량/지연 비교
  5단계: 운영 배포 + JFR 모니터링

모니터링 변화:
  기존: JVM Thread Count (200개 정도)
  VT 이후: Platform Thread Count (CPU 코어 수) + VT Count (수천~수만)
  Grafana/Micrometer: jvm.threads.live + jvm.threads.virtual.live
```

---

## 📌 핵심 정리

```
Spring Boot + VT 핵심:

활성화:
  spring.threads.virtual.enabled=true (Spring Boot 3.2+)
  → Tomcat 요청 처리 VT
  → @Async 기본 executor VT
  → 기존 코드 변경 없음

HikariCP:
  VT 대기 중 Unmount → 커넥션 대기 비용 낮음
  maximum-pool-size = DB 최적 연결 수 (줄여도 됨)
  최신 버전(5.1.0+) 사용으로 Pinning 최소화

StructuredTaskScope:
  여러 VT 작업의 구조화된 조율
  ShutdownOnFailure: 하나 실패 → 모두 취소
  ShutdownOnSuccess: 가장 빠른 하나 성공 → 나머지 취소
  ScopedValue 자동 상속

Pinning 진단 워크플로우:
  개발: -Djdk.tracePinnedThreads=full
  운영: JFR jdk.VirtualThreadPinned
  수정: synchronized → ReentrantLock

모니터링:
  jvm.threads.live (OS 스레드 = 캐리어 수)
  jvm.threads.virtual.live (VT 수)
  jdk.VirtualThreadPinned (JFR)
```

---

## 🤔 생각해볼 문제

**Q1.** `spring.threads.virtual.enabled=true` 설정 시 Spring WebFlux 기반 애플리케이션에도 영향이 있는가?

<details>
<summary>해설 보기</summary>

Spring WebFlux는 Netty나 Undertow 같은 Non-Blocking 서버를 사용하며, 자체 이벤트 루프(Reactor Netty의 Event Loop Group)를 보유한다. `spring.threads.virtual.enabled=true`는 주로 Tomcat과 Spring MVC의 스레드 모델에 영향을 준다.

WebFlux + VT 조합:
- 서버 레벨: Netty의 이벤트 루프는 VT가 아닌 자체 OS 스레드 사용 (영향 없음)
- 비즈니스 로직: WebFlux의 `Mono/Flux` 체인은 Reactor 스케줄러에서 실행 (VT와 무관)
- `@Async`와 조합: VT executor가 적용될 수 있음 (혼용 주의)

권장: WebFlux와 VT를 같은 애플리케이션에 혼용하는 것은 복잡성만 증가시킨다. I/O 바운드 서비스는 VT + Spring MVC, 또는 WebFlux 중 하나를 선택한다.

</details>

---

**Q2.** `@Transactional` 메서드가 Virtual Thread에서 실행될 때 트랜잭션 전파는 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

Spring의 `@Transactional`은 `TransactionSynchronizationManager`가 `ThreadLocal`로 트랜잭션 컨텍스트를 관리한다. VT는 고유한 `ThreadLocal`을 가지므로 트랜잭션 컨텍스트도 VT별로 독립적이다.

단일 VT 내의 트랜잭션 전파는 정상 동작한다:
```java
@Transactional  // VT에서 호출
public void serviceA() {
    repo.save(entity1);  // 같은 트랜잭션
    serviceB();          // 같은 VT → 같은 ThreadLocal → 트랜잭션 전파 OK
}
```

다른 VT로 넘기면 트랜잭션이 끊어진다:
```java
@Transactional
public void serviceA() {
    repo.save(entity1);  // 트랜잭션 T1

    // 새 VT에서 실행 → 별도 ThreadLocal → T1과 무관
    Thread.ofVirtual().start(() -> {
        repo.save(entity2);  // 새 트랜잭션 또는 트랜잭션 없음
    });
}
```

`@Async`와 `@Transactional` 조합: `@Async`는 새 VT에서 실행하므로 트랜잭션이 새로 시작된다. 이것은 OS 스레드와 동일한 동작이다.

</details>

---

**Q3.** Virtual Thread 환경에서 Actuator의 `/actuator/threaddump`는 어떻게 보이는가?

<details>
<summary>해설 보기</summary>

`/actuator/threaddump`는 기본적으로 플랫폼 스레드(OS 스레드) 덤프를 반환한다. VT 활성화 후:

- 플랫폼 스레드: 캐리어 스레드 수(CPU 코어 수) + 기타 시스템 스레드 (훨씬 적음)
- VT는 기본적으로 스레드 덤프에 포함되지 않음 (수백만 개가 될 수 있으므로)

VT를 덤프에 포함하려면:
```bash
# jcmd로 VT 포함 덤프
jcmd <pid> Thread.dump_to_file -format=json /tmp/dump.json

# Spring Actuator 설정 (application.yml)
management:
  endpoint:
    threaddump:
      enabled: true
# Spring Boot 3.2+: 자동으로 VT 포함 여부 표시
```

VT 포함 덤프의 특징:
- 활성 VT만 표시 (park된 VT는 표시 안 될 수 있음)
- 캐리어 스레드 + 연결된 VT 구조로 표시
- VT의 스택 트레이스는 동기 코드처럼 읽기 쉬움

실무 모니터링: VT 수는 `Micrometer`의 `jvm.threads.virtual.live` 메트릭으로 추적한다.

</details>

---

<div align="center">

**[⬅️ 이전: ThreadLocal과 ScopedValue](./05-threadlocal-scopedvalue.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 — 데드락 분석 ➡️](../real-world-patterns/01-deadlock-analysis.md)**

</div>

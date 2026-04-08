# Spring 동시성 이슈 — @Transactional·@Async·싱글톤 Bean

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Transactional`이 스레드에 바인딩되는 원리와 멀티스레드에서의 주의사항은?
- `@Async` 스레드 풀 설정이 누락될 때 `SimpleAsyncTaskExecutor`가 일으키는 문제는?
- 싱글톤 Bean의 인스턴스 변수를 여러 스레드가 공유할 때 Race Condition이 발생하는 패턴은?
- 이 이슈들을 운영 환경에서 어떻게 진단하는가?
- 각 문제를 예방하는 코드 패턴은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring의 편리한 어노테이션(`@Transactional`, `@Async`)이 내부에서 스레드 기반 메커니즘을 사용한다. 이를 모르고 멀티스레드 코드를 작성하면 트랜잭션 누락, OOM, 데이터 손상이 발생한다. 싱글톤 Bean은 Spring의 기본 스코프이므로, 인스턴스 변수의 공유가 미치는 영향을 이해하는 것은 Spring 개발자의 필수 지식이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: @Transactional 메서드에서 새 스레드 생성
  @Transactional
  public void processOrder(Order order) {
      orderRepo.save(order);
      new Thread(() -> {
          // 이 스레드는 부모 트랜잭션과 무관!
          inventoryService.deduct(order);  // 트랜잭션 없이 실행
      }).start();
  }
  // 주문 저장 롤백 시 재고 차감은 롤백 안 됨 → 데이터 불일치

실수 2: @Async 기본 설정 그대로 사용
  @Async
  public void sendEmail(String to, String content) {
      emailService.send(to, content);
  }
  // 기본 executor: SimpleAsyncTaskExecutor
  // → 호출마다 새 OS 스레드 생성!
  // → 초당 수천 호출 → OOM

실수 3: 싱글톤 Bean에 인스턴스 변수 사용
  @Service
  public class StatisticsService {
      private int requestCount = 0;  // 모든 요청 스레드가 공유!
      
      public void track(Request req) {
          requestCount++;  // Race Condition! (원자적이지 않음)
      }
  }
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// @Transactional 올바른 패턴

// 패턴 1: 다른 스레드에서 트랜잭션 전파 불가 → 이벤트 기반으로 분리
@Transactional
public void processOrder(Order order) {
    orderRepo.save(order);
    // 새 스레드 대신 이벤트 발행 (동일 트랜잭션 내)
    eventPublisher.publishEvent(new OrderCreatedEvent(order));
    // 리스너가 트랜잭션 커밋 후 실행 (@TransactionalEventListener)
}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Async("emailExecutor")  // 트랜잭션 커밋 후 비동기 실행
public void onOrderCreated(OrderCreatedEvent event) {
    inventoryService.deduct(event.getOrder());
}

// 패턴 2: StructuredTaskScope (Java 21, 구조화된 동시성)
@Transactional
public OrderResult processOrderParallel(Order order) throws Exception {
    // 트랜잭션 내에서 병렬 처리 (동일 스레드 컨텍스트 유지)
    // StructuredTaskScope는 부모 VT의 트랜잭션을 공유하지 않음
    // → 각 스코프 작업은 독립 트랜잭션
    orderRepo.save(order);  // 메인 트랜잭션
    return new OrderResult(order);
}

// === @Async 올바른 설정 ===
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Bean("emailExecutor")
    public Executor emailExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("email-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    // Virtual Thread 기반 Async executor (Spring Boot 3.2+)
    @Bean("vtExecutor")
    public Executor vtExecutor() {
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }

    @Override
    public Executor getAsyncExecutor() {
        return emailExecutor();  // 기본 @Async executor
    }
}

// === 싱글톤 Bean 상태 안전 패턴 ===
@Service
public class StatisticsService {
    // Bad: private int requestCount = 0;

    // Good 1: AtomicInteger 사용
    private final AtomicInteger requestCount = new AtomicInteger(0);
    // Good 2: LongAdder (고경쟁 시 더 빠름)
    private final LongAdder totalRequests = new LongAdder();

    public void track(Request req) {
        requestCount.incrementAndGet();  // 스레드 안전
        totalRequests.increment();       // 스레드 안전
    }

    // Bad: 메서드 내 인스턴스 변수 임시 저장
    // private String tempResult;  // ← 절대 안 됨!

    // Good: 메서드 로컬 변수 사용 (스택에 저장 → 스레드 안전)
    public String processRequest(Request req) {
        String tempResult = compute(req);  // 로컬 변수 → 스레드 안전
        return format(tempResult);
    }
}
```

---

## 🔬 내부 동작 원리

### 1. @Transactional의 스레드 바인딩 원리

```java
// TransactionSynchronizationManager 내부 (단순화)
public class TransactionSynchronizationManager {
    // ThreadLocal로 현재 스레드의 트랜잭션 정보 저장
    private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");

    private static final ThreadLocal<Boolean> actualTransactionActive =
        new NamedThreadLocal<>("Actual transaction active");

    private static final ThreadLocal<String> currentTransactionName =
        new NamedThreadLocal<>("Current transaction name");
}

// @Transactional 동작 흐름:
// 1. AOP 프록시가 메서드 진입 전 트랜잭션 시작
//    → DataSource.getConnection()
//    → TransactionSynchronizationManager.bindResource(dataSource, connection)
//    → ThreadLocal에 커넥션 저장

// 2. 메서드 실행 중
//    → 모든 Repository/JPA 호출이 ThreadLocal에서 커넥션 가져옴
//    → 같은 커넥션 = 같은 트랜잭션

// 3. 새 스레드에서 DB 접근
//    → 새 스레드는 다른 ThreadLocal 공간 보유
//    → ThreadLocal에서 커넥션을 찾지 못함
//    → 새 커넥션 생성 (=새 트랜잭션, 또는 트랜잭션 없음)
//    → 부모 트랜잭션 롤백 시 자식 스레드의 작업은 롤백 안 됨!

// 자식 스레드 전파 방법 (권장하지 않음):
// TransactionTemplate을 사용하거나
// 부모 트랜잭션 커밋 후 이벤트로 처리

// @Transactional 자가 호출 문제 (동일 클래스 내):
@Service
public class OrderService {
    @Transactional
    public void outer() {
        inner();  // 프록시 없이 직접 호출 → @Transactional 무시!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void inner() {
        // 별도 트랜잭션으로 실행 기대
        // 하지만 outer()에서 직접 호출 시 REQUIRES_NEW 무시됨!
    }
}
// 해결: inner()를 별도 Bean으로 분리 또는 ApplicationContext.getBean()
```

### 2. @Async와 SimpleAsyncTaskExecutor 문제

```java
// SimpleAsyncTaskExecutor 내부 (단순화)
public class SimpleAsyncTaskExecutor implements TaskExecutor {
    @Override
    public void execute(Runnable task) {
        // 매번 새 Thread 생성! 풀 없음!
        Thread thread = new Thread(task);
        thread.start();
    }
}

// 문제 시나리오:
// 초당 1,000회 @Async 호출
// → 초당 1,000개 OS 스레드 생성
// → 각 1MB 스택 → 초당 1GB 할당
// → OOM 또는 시스템 한계 초과
// → 실무에서 갑자기 서버 멈추는 주요 원인 중 하나

// 기본 executor 설정 (Spring Boot 자동 설정):
// spring.task.execution.pool.core-size=8
// spring.task.execution.pool.max-size=200  (기본 매우 큼)
// spring.task.execution.pool.queue-capacity=2147483647  (사실상 무한)

// 올바른 설정:
spring:
  task:
    execution:
      pool:
        core-size: 10
        max-size: 50
        queue-capacity: 500        # 500개 대기 후 거부
        keep-alive: 60s
      thread-name-prefix: async-  # 스레드 덤프에서 식별 용이

// VT 기반 @Async (Spring Boot 3.2+):
// spring.threads.virtual.enabled=true
// → @Async 기본 executor가 VT executor로 자동 설정

// @Async 예외 처리:
@Configuration
public class AsyncExceptionConfig implements AsyncConfigurer {
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("@Async 예외: method={}, params={}", method, params, ex);
            // 알림 발송 등
        };
    }
}
```

### 3. 싱글톤 Bean Race Condition 패턴과 진단

```java
// Race Condition 발생 패턴들

// 패턴 1: 인스턴스 변수를 메서드 간 상태 공유
@Service
public class ReportService {
    private List<String> currentReportLines;  // ← Race Condition!

    public void startReport() {
        currentReportLines = new ArrayList<>();  // 한 스레드가 초기화
    }

    public void addLine(String line) {
        currentReportLines.add(line);  // 다른 스레드가 다른 리스트에 접근
    }

    public String finishReport() {
        return String.join("\n", currentReportLines);
    }
}
// → 두 요청이 동시에 오면 리스트가 혼재됨

// 해결: 메서드 파라미터 또는 반환 값으로 상태 전달
@Service
public class ReportService {
    public String generateReport(List<String> lines) {
        // 상태가 없는 순수 메서드 → 스레드 안전
        return String.join("\n", lines);
    }
}

// 패턴 2: 캐시 구현 시 non-atomic 복합 연산
@Service
public class CacheService {
    private final Map<String, Object> cache = new HashMap<>();

    // 위험: check-then-act (두 스레드가 동시에 null 확인 가능)
    public Object get(String key) {
        if (!cache.containsKey(key)) {      // 스레드 A, B 모두 false
            Object value = compute(key);
            cache.put(key, value);           // 둘 다 put → 하나가 덮어씀
        }
        return cache.get(key);
    }

    // 해결: ConcurrentHashMap.computeIfAbsent
    private final ConcurrentHashMap<String, Object> safeCache = new ConcurrentHashMap<>();

    public Object safeGet(String key) {
        return safeCache.computeIfAbsent(key, k -> compute(k));  // 원자적
    }
}

// 패턴 3: 날짜/시간 포맷터 (non-thread-safe 객체 공유)
@Service
public class DateService {
    // Bad: SimpleDateFormat은 스레드 안전하지 않음!
    private final SimpleDateFormat formatter = new SimpleDateFormat("yyyy-MM-dd");

    public String format(Date date) {
        return formatter.format(date);  // Race Condition → 잘못된 날짜 출력
    }

    // Good 1: ThreadLocal
    private static final ThreadLocal<SimpleDateFormat> FORMATTER =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

    public String safeFormat(Date date) {
        return FORMATTER.get().format(date);
    }

    // Good 2: DateTimeFormatter (Java 8+, 스레드 안전)
    private static final DateTimeFormatter DTF =
        DateTimeFormatter.ofPattern("yyyy-MM-dd");

    public String modernFormat(LocalDate date) {
        return DTF.format(date);  // 스레드 안전
    }
}
```

### 4. 운영 환경 진단 방법

```bash
# 문제 1: @Async OOM 진단
# 증상: 메모리 급증, java.lang.OutOfMemoryError: unable to create native thread

# Thread Dump에서 @Async 스레드 수 확인
jstack <pid> | grep -E "(task-|async-|SimpleAsync)" | wc -l
# 수백~수천 개면 SimpleAsyncTaskExecutor 의심

# 스레드 이름 패턴 확인
jstack <pid> | grep "thread state" -A 3 | grep "task-"

# 문제 2: 트랜잭션 누락 진단
# JFR 또는 AOP 로깅
@Aspect
@Component
public class TransactionLoggingAspect {
    @Before("@annotation(org.springframework.transaction.annotation.Transactional)")
    public void logTransaction() {
        log.debug("트랜잭션 활성: {}, 스레드: {}",
            TransactionSynchronizationManager.isActualTransactionActive(),
            Thread.currentThread().getName());
    }
}

# 문제 3: 싱글톤 Race Condition 진단
# FindBugs/SpotBugs: IS2_INCONSISTENT_SYNC 경고
# PMD: NonThreadSafeSingleton 규칙

# Thread Sanitizer (TSan) - Java용:
# Google의 ThreadSanitizer 플러그인 또는
# JVM TI 기반 도구 사용

# 가장 간단: 코드 리뷰에서 @Service/@Component 클래스의
# final이 아닌 인스턴스 변수 찾기
grep -rn "private [^final]" --include="*.java" \
     $(find . -name "*.java" | xargs grep -l "@Service\|@Component\|@Controller")
```

---

## 💻 실전 실험

### 실험 1: 싱글톤 Race Condition 재현

```java
@SpringBootTest
class RaceConditionTest {
    @Autowired
    private UnsafeCounterService counterService;

    @Test
    void raceConditionDemo() throws InterruptedException {
        int threads = 100;
        int iterations = 1000;
        CountDownLatch latch = new CountDownLatch(threads);
        ExecutorService executor = Executors.newFixedThreadPool(threads);

        for (int i = 0; i < threads; i++) {
            executor.submit(() -> {
                for (int j = 0; j < iterations; j++) {
                    counterService.increment();
                }
                latch.countDown();
            });
        }

        latch.await();
        int expected = threads * iterations;
        int actual = counterService.getCount();
        System.out.printf("기대: %d, 실제: %d, 손실: %d (%.2f%%)%n",
            expected, actual, expected - actual,
            (expected - actual) * 100.0 / expected);
        // actual < expected (Race Condition으로 일부 increment 손실)
    }
}

@Service
class UnsafeCounterService {
    private int count = 0;  // Race Condition!
    public void increment() { count++; }
    public int getCount() { return count; }
}
```

### 실험 2: SimpleAsyncTaskExecutor 스레드 폭발 확인

```java
@SpringBootTest
class AsyncThreadExplosionTest {
    @Autowired
    private EmailService emailService;

    @Test
    void demonstrateThreadExplosion() throws InterruptedException {
        int beforeThreads = Thread.activeCount();
        System.out.println("시작 스레드 수: " + beforeThreads);

        // 500번 @Async 호출
        for (int i = 0; i < 500; i++) {
            emailService.sendWelcomeEmail("user" + i + "@example.com");
        }

        Thread.sleep(100);  // 스레드 생성 대기
        int afterThreads = Thread.activeCount();
        System.out.println("호출 후 스레드 수: " + afterThreads);
        System.out.println("생성된 스레드: " + (afterThreads - beforeThreads));
        // SimpleAsyncTaskExecutor: ~500개 스레드 생성됨
        // 올바른 풀: core-size 수준만 생성
    }
}
```

---

## 📊 성능/비용 비교

```
@Async Executor 종류별 특성:

Executor 종류              | 동시 실행  | 스레드 생성     | OOM 위험
──────────────────────────┼──────────┼──────────────┼─────────
SimpleAsyncTaskExecutor   | 무제한     | 매번 새 생성    | 높음! ⚠️
ThreadPoolTaskExecutor    | 풀 크기    | 풀에서 재사용   | 낮음
VT (Spring Boot 3.2+)     | 무제한     | 매번 새 VT     | 낮음 (힙)

@Transactional 주의 수준:
  자가 호출(self-invocation): ⚠️ 트랜잭션 무시
  새 스레드 생성: ⚠️ 트랜잭션 전파 안 됨
  @Async 내부 호출: ⚠️ 별도 스레드 = 별도 트랜잭션
  올바른 AOP 프록시 사용: ✅

싱글톤 스레드 안전성:
  final 불변 필드: ✅ 안전
  AtomicInteger/LongAdder: ✅ 안전
  ConcurrentHashMap: ✅ 안전
  non-final primitive 필드: ❌ Race Condition
  non-thread-safe 객체: ❌ Race Condition
  메서드 로컬 변수: ✅ 항상 안전
```

---

## ⚖️ 트레이드오프

```
각 이슈별 해결책의 트레이드오프:

@Transactional 멀티스레드:
  이벤트 + @TransactionalEventListener:
    장점: 트랜잭션 커밋 후 안전하게 실행, 비동기 처리
    단점: 이벤트 발행/리스닝 구조 추가, 실패 시 보상 트랜잭션 필요

  Saga 패턴:
    장점: 분산 트랜잭션 지원, 각 서비스 독립 트랜잭션
    단점: 구현 복잡성, 최종 일관성(eventual consistency)

@Async 설정:
  ThreadPoolTaskExecutor:
    장점: 풀 관리, OOM 방지, 모니터링 가능
    단점: 크기 설정 필요 (잘못 설정 시 성능 저하)

  Virtual Thread Executor:
    장점: 풀 크기 튜닝 불필요, 경량
    단점: Pinning 주의, ThreadLocal 관리

싱글톤 상태 관리:
  상태 없는 Bean (Stateless):
    장점: 가장 안전, 성능 최적
    단점: 상태가 필요한 경우 파라미터 전달 복잡

  원자적 타입 (AtomicXxx, LongAdder):
    장점: 간단, 명확
    단점: 복합 연산은 여전히 별도 처리 필요
```

---

## 📌 핵심 정리

```
Spring 동시성 이슈 핵심:

@Transactional:
  ThreadLocal 기반 → 스레드에 바인딩
  새 스레드 생성 시 트랜잭션 전파 안 됨
  자가 호출(self-invocation) → @Transactional 무시
  해결: @TransactionalEventListener, 별도 Bean, 구조적 설계

@Async:
  기본 executor = SimpleAsyncTaskExecutor → 매번 새 OS 스레드!
  고빈도 호출 → OOM 위험
  해결: ThreadPoolTaskExecutor 명시 설정 또는 VT executor
  예외 처리: AsyncUncaughtExceptionHandler 등록 필수

싱글톤 Bean:
  인스턴스 변수 = 모든 요청 스레드가 공유
  non-final 상태 필드 → Race Condition
  SimpleDateFormat 등 non-thread-safe 객체 공유 → 데이터 손상
  해결:
    상태 없는 Bean 지향 (Stateless)
    AtomicXxx / LongAdder (카운터)
    ConcurrentHashMap (캐시)
    DateTimeFormatter (스레드 안전 날짜 처리)
    ThreadLocal (스레드별 인스턴스 필요 시)
    메서드 로컬 변수 활용

진단:
  @Async 문제: Thread Dump에서 스레드 수 확인
  트랜잭션 누락: AOP 로깅, 데이터 불일치
  Race Condition: 멀티스레드 테스트, SpotBugs 정적 분석
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional` 메서드가 같은 클래스의 다른 `@Transactional` 메서드를 호출할 때 트랜잭션 전파가 동작하지 않는 이유는?

<details>
<summary>해설 보기</summary>

Spring의 `@Transactional`은 AOP 프록시를 통해 동작한다. 외부에서 Bean 메서드를 호출하면 Spring이 생성한 프록시 객체를 통해 호출되고, 프록시가 트랜잭션 시작/종료 로직을 감싼다.

자가 호출(self-invocation) 시:
```java
// this.inner() → 프록시가 아닌 실제 객체를 통해 직접 호출
public void outer() {
    this.inner();  // 'this'는 프록시가 아닌 실제 OrderService 인스턴스
}
```

프록시를 거치지 않으므로 AOP가 동작하지 않고, `@Transactional`이 무시된다.

해결책:
1. **별도 Bean으로 분리**: `inner()` 메서드를 다른 서비스 Bean으로 이동
2. **ApplicationContext.getBean()**: 자신의 프록시를 직접 가져와 호출 (안티패턴)
3. **AopContext.currentProxy()**: 현재 프록시를 가져와 호출 (`@EnableAspectJAutoProxy(exposeProxy=true)` 필요)
4. **구조 재설계**: 자가 호출 자체를 피하는 것이 가장 좋음

```java
// 해결: 별도 Bean 분리
@Service
public class OrderService {
    @Autowired private OrderInnerService innerService;

    @Transactional
    public void outer() {
        innerService.inner();  // 프록시를 통해 호출 → REQUIRES_NEW 동작
    }
}

@Service
public class OrderInnerService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void inner() { ... }
}
```

</details>

---

**Q2.** `@Async` 메서드가 `@Transactional` 메서드 내에서 호출될 때 트랜잭션은 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`@Async` 메서드는 별도 스레드(executor)에서 실행된다. 별도 스레드는 호출자의 `ThreadLocal`을 공유하지 않으므로 호출자의 트랜잭션을 상속받지 않는다.

시나리오:
```java
@Transactional
public void parent() {
    repo.save(entity);          // 트랜잭션 T1에서 실행
    asyncService.doAsync();     // 새 스레드(T2 또는 트랜잭션 없음)에서 실행
    // parent() 롤백 시 repo.save() 롤백됨
    // asyncService.doAsync()의 결과는 이미 커밋됐거나 별도 트랜잭션
}

@Async
@Transactional  // 새 트랜잭션으로 시작 (Propagation.REQUIRED의 새 트랜잭션)
public void doAsync() {
    asyncRepo.save(anotherEntity);  // 새 트랜잭션 T2
}
```

중요한 점:
- `@Async` 내의 `@Transactional`은 별도의 새 트랜잭션으로 시작
- 부모 롤백이 `@Async` 작업에 영향을 주지 않음
- 따라서 `@Async` 호출 타이밍 주의: 부모 트랜잭션 커밋 후 호출하려면 `@TransactionalEventListener(phase = AFTER_COMMIT)` 사용

`@TransactionalEventListener(phase = AFTER_COMMIT)` + `@Async` 조합이 가장 안전한 패턴:
- 부모 트랜잭션이 성공적으로 커밋된 후에만 이벤트 리스너 실행
- `@Async`로 비동기 실행

</details>

---

**Q3.** Spring `@Scope("prototype")` Bean은 싱글톤 Bean의 Race Condition 문제가 없는가? 언제 사용하는가?

<details>
<summary>해설 보기</summary>

`@Scope("prototype")` Bean은 요청마다 새로운 인스턴스가 생성되므로 인스턴스 변수의 Race Condition이 없다. 각 요청이 자신만의 Bean 인스턴스를 가진다.

하지만 주의사항이 있다:
1. **싱글톤 Bean에 주입 시 문제**: 싱글톤 Bean이 prototype Bean을 `@Autowired`로 주입받으면 싱글톤 생성 시점에 한 번만 주입되어 사실상 싱글톤처럼 동작한다.

```java
@Service  // 싱글톤
public class SingletonService {
    @Autowired
    private PrototypeBean prototypeBean;  // 주입 시점에 한 번만 → 사실상 싱글톤!
}
```

해결: `ApplicationContext.getBean()`, `@Lookup`, `ObjectProvider<PrototypeBean>` 사용.

2. **메모리 관련**: prototype Bean은 Spring이 소멸을 관리하지 않는다. 직접 `DisposableBean` 또는 `@PreDestroy`로 정리해야 한다.

사용 시나리오:
- 요청별로 다른 상태가 필요한 경우 (예: 파일 파서, 워크플로우 컨텍스트)
- 스레드 안전하지 않은 외부 라이브러리 객체를 Spring Bean으로 관리할 때
- 단순 상태 없는 Bean은 싱글톤으로 충분하므로, prototype은 꼭 필요한 경우에만 사용

</details>

---

<div align="center">

**[⬅️ 이전: 비동기 프로그래밍 비교](./04-async-comparison.md)** | **[홈으로 🏠](../README.md)**

</div>

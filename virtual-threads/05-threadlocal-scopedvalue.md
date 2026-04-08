# ThreadLocal과 ScopedValue — 메모리 누수와 Java 21 해결책

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Virtual Thread 100만 개 생성 시 ThreadLocal 값이 복사되어 메모리 누수가 발생하는 원리는?
- 풀링(pooling)이 Virtual Thread에서 안티패턴인 이유는?
- Java 21 `ScopedValue`가 불변 상태를 스코프 내에서 공유하는 설계 원리는?
- `ThreadLocal`과 `ScopedValue`는 각각 어떤 상황에 적합한가?
- VT 환경에서 요청별 컨텍스트(사용자 ID, 트레이스 ID)를 어떻게 전달하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring Security의 `SecurityContextHolder`, SLF4J의 MDC, DB 트랜잭션 컨텍스트가 모두 `ThreadLocal`을 사용한다. Virtual Thread를 수백만 개 사용하면 이 값들이 수백만 번 복사되어 메모리 압박이 생긴다. Java 21의 `ScopedValue`는 이 문제의 근본적 해결책이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: VT에서 ThreadLocal이 자동으로 공유된다는 오해
  "VT들이 같은 캐리어를 공유하니까 ThreadLocal도 공유되겠지"
  → 아님! 각 Thread(VT 포함)는 독립적인 ThreadLocalMap 보유
  → VT 100만 개 = ThreadLocalMap 100만 개
  → 각 요청의 사용자 컨텍스트가 100만 번 복사됨

실수 2: VT 풀링으로 ThreadLocal 재사용 시도
  ExecutorService pool = Executors.newFixedThreadPool(
      100, Thread.ofVirtual().factory()
  );
  // 100개 VT를 풀에서 재사용 → ThreadLocal 누적!
  pool.submit(() -> {
      MDC.put("requestId", "req-123");
      doWork();
      // MDC.clear()를 안 하면 다음 작업에서 이전 requestId가 남음
  });

실수 3: InheritableThreadLocal로 VT 자식에게 값 전달 시 메모리 이슈
  InheritableThreadLocal<String> ctx = new InheritableThreadLocal<>();
  ctx.set("parent-context");
  // 부모에서 VT 생성 시 자식으로 복사
  Thread.ofVirtual().start(() -> {
      System.out.println(ctx.get()); // "parent-context" 복사됨
  });
  // 100만 VT 생성 = 100만 번 복사 → 메모리 폭증
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// ScopedValue 사용 패턴 (Java 21+)

// ScopedValue 선언 (상수, 공유 가능)
static final ScopedValue<String> USER_ID = ScopedValue.newInstance();
static final ScopedValue<String> TRACE_ID = ScopedValue.newInstance();

// 값 바인딩 및 사용 (스코프 내에서만 유효)
ScopedValue.where(USER_ID, "user-123")
           .where(TRACE_ID, "trace-abc")
           .run(() -> {
               // 이 스코프 안에서 어디서든 읽기 가능
               processRequest();
               callService();
           });

// 어디서든 읽기 (setter 없음)
void processRequest() {
    String userId = USER_ID.get();       // "user-123"
    String traceId = TRACE_ID.get();     // "trace-abc"
    log.info("User: {}, Trace: {}", userId, traceId);
}

// ScopedValue 재바인딩 (중첩, 내부 스코프에서 다른 값)
ScopedValue.where(USER_ID, "admin")
           .run(() -> {
               // 이 스코프에서 USER_ID = "admin"
               ScopedValue.where(USER_ID, "user-456")
                          .run(() -> {
                              System.out.println(USER_ID.get()); // "user-456"
                          });
               System.out.println(USER_ID.get()); // "admin" (원복)
           });

// VT + ScopedValue 조합
try (var exe = Executors.newVirtualThreadPerTaskExecutor()) {
    // 각 VT에 요청 컨텍스트 바인딩
    for (Request req : requests) {
        exe.submit(() ->
            ScopedValue.where(USER_ID, req.userId())
                       .where(TRACE_ID, req.traceId())
                       .run(() -> handleRequest(req))
        );
    }
}
```

---

## 🔬 내부 동작 원리

### 1. ThreadLocal의 메모리 구조

```java
// Thread 내부 (단순화)
class Thread {
    ThreadLocal.ThreadLocalMap threadLocals;      // 이 스레드의 ThreadLocal 맵
    ThreadLocal.ThreadLocalMap inheritableThreadLocals;  // 상속 가능한 맵
}

// ThreadLocal.ThreadLocalMap 구조
class ThreadLocalMap {
    Entry[] table;  // Open-addressing 해시 테이블
    // key = ThreadLocal 인스턴스 (WeakReference)
    // value = ThreadLocal.get()로 반환할 값

    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;
    }
}

// ThreadLocal.get() 동작
T get() {
    Thread t = Thread.currentThread();  // 현재 스레드(VT 포함)의 ThreadLocalMap
    ThreadLocalMap map = t.threadLocals;
    if (map != null) {
        Entry e = map.getEntry(this);   // this = ThreadLocal 인스턴스
        if (e != null) return (T) e.value;
    }
    return setInitialValue();
}

// VT에서 ThreadLocal 메모리 분석:
// VT 100만 개 × MDC threadlocal 1개:
//   ThreadLocalMap 100만 개
//   각 Entry: ~64~128 bytes (key WeakRef + value)
//   총: ~64~128 MB (MDC 값이 짧아도)
//   MDC value가 큰 객체이면: 훨씬 더 많음

// 메모리 누수 조건:
// ThreadLocal에 큰 객체 바인딩 + VT 많이 생성 + remove() 미호출
// Entry의 key(ThreadLocal)가 WeakRef로 수거돼도 value는 남을 수 있음
```

### 2. VT와 ThreadLocal의 상호작용

```
InheritableThreadLocal의 VT 문제:
  부모 스레드에서 VT 생성 시 InheritableThreadLocal 값 복사
  100만 VT 생성 = 100만 번 복사
  
  StructuredTaskScope (Java 21):
    부모 VT의 ScopedValue가 자식 VT에 자동으로 공유됨
    복사 없이 공유 (참조만 전달) → 메모리 효율적

ThreadLocal 사용 금지가 아닌 경우:
  VT당 독립적인 가변 상태가 필요한 경우:
    RNG 시드 (ThreadLocalRandom)
    SimpleDateFormat 인스턴스
    StringBuilder 재사용 버퍼
  → 이 경우 ThreadLocal이 여전히 적합
  → 단, VT 수만큼 인스턴스 생성 → 메모리 고려

VT 풀링 금지 이유:
  VT는 매번 새로 생성 (생성 비용 낮음)
  풀링하면:
    이전 요청의 ThreadLocal 값이 남음
    정리 코드(remove)가 누락되면 데이터 혼재
    VT의 격리 보장이 깨짐
  → Executors.newVirtualThreadPerTaskExecutor() 사용 (매번 새 VT)
```

### 3. ScopedValue의 설계 원리

```java
// ScopedValue 내부 (단순화)
public final class ScopedValue<T> {
    // 각 ScopedValue는 고유 key를 가짐
    private final int hash;  // 고유 식별자

    // 현재 스레드의 ScopedValue 스냅샷에서 값 조회
    public T get() {
        Object[] snapshot = Thread.currentCarrierThread().scopedValueBindings();
        // 스냅샷에서 this key에 해당하는 값 찾기
        return findValue(snapshot);
    }

    // where().run()의 의미
    // ScopedValue.where(KEY, value).run(action):
    //   ① 현재 스레드의 바인딩 스냅샷에 (KEY, value) 추가한 새 스냅샷 생성
    //   ② action 실행 (action 내에서 KEY.get() = value)
    //   ③ 이전 스냅샷 복원 (action 종료 후)
}

ScopedValue의 특성:
  불변(Immutable): 한번 바인딩되면 스코프 내에서 변경 불가
  스코프 한정: run() 블록 내에서만 유효
  상속 자동: StructuredTaskScope의 자식 VT에 자동 전파
  복사 없음: 참조만 전달 (ThreadLocal과 달리 복사 안 함)
  성능: 해시 조회 O(1), 스냅샷 배열 접근

ThreadLocal vs ScopedValue 비교:

              | ThreadLocal            | ScopedValue
─────────────┼───────────────────────┼──────────────────────────────
가변성        | 가변 (set/get/remove)  | 불변 (run() 스코프 내)
생명주기      | 스레드 전체 또는 remove | run() 블록 종료 시 자동 해제
자식 전파    | InheritableThreadLocal | StructuredTaskScope 자동 전파
메모리        | 스레드당 복사           | 참조 공유 (복사 없음)
VT 친화성    | 주의 필요 (대량 복사)   | ✅ 설계부터 VT를 위해 만들어짐
예외 안전성  | finally에 remove() 필요 | run() 자동 정리
```

### 4. Spring + VT 환경에서의 컨텍스트 전달

```java
// Spring Security의 SecurityContextHolder (ThreadLocal 기반)
// VT 환경에서 처리 방법:

// 방법 1: VT별 새 컨텍스트 (스레드 모드 변경)
// application.properties:
// spring.threads.virtual.enabled=true
// spring.security.strategy=INHERITABLETHREADLOCAL (기본)
// → InheritableThreadLocal로 자식 VT에 전파

// 방법 2: 명시적 컨텍스트 전달
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
try (var exe = Executors.newVirtualThreadPerTaskExecutor()) {
    exe.submit(() -> {
        // 새 VT에 보안 컨텍스트 설정
        SecurityContextHolder.getContext().setAuthentication(auth);
        try {
            doWork();
        } finally {
            SecurityContextHolder.clearContext();  // 반드시 정리
        }
    });
}

// 방법 3: ScopedValue로 대체 (미래 방향)
static final ScopedValue<Authentication> AUTH = ScopedValue.newInstance();

void handleRequest(Authentication auth, Request req) {
    ScopedValue.where(AUTH, auth).run(() -> {
        doWork();  // AUTH.get()으로 어디서든 접근
        // 스코프 종료 시 자동 정리, no ThreadLocal
    });
}

// SLF4J MDC (Virtual Thread 환경)
// MDC는 ThreadLocal 기반 → VT에서 각 요청별 독립됨 (장점이기도!)
// 다만 대량 VT에서 MDC put/remove 비용 고려
try (var exe = Executors.newVirtualThreadPerTaskExecutor()) {
    exe.submit(() -> {
        MDC.put("requestId", generateId());
        try { doWork(); }
        finally { MDC.clear(); }  // 반드시 정리
    });
}
```

---

## 💻 실전 실험

### 실험 1: ThreadLocal 메모리 누수 재현

```java
public class ThreadLocalMemoryLeak {
    static ThreadLocal<byte[]> HEAVY_LOCAL = ThreadLocal.withInitial(
        () -> new byte[1024]  // 1KB per VT
    );

    public static void main(String[] args) throws Exception {
        Runtime rt = Runtime.getRuntime();
        System.gc();
        long base = rt.totalMemory() - rt.freeMemory();

        // 10,000개 VT, 각각 ThreadLocal 사용
        int count = 10_000;
        Thread[] vts = new Thread[count];
        for (int i = 0; i < count; i++) {
            vts[i] = Thread.ofVirtual().start(() -> {
                HEAVY_LOCAL.get();  // ThreadLocal 접근 (1KB 할당)
                try { Thread.sleep(1000); } catch (InterruptedException e) {}
                // HEAVY_LOCAL.remove() 없음 → 메모리 유지
            });
        }

        Thread.sleep(100);
        long withVt = rt.totalMemory() - rt.freeMemory();
        System.out.printf("%d VT (ThreadLocal 1KB): %.2f MB%n",
            count, (withVt - base) / 1_048_576.0);
        // 예상: ~10MB (10,000 × 1KB)

        for (Thread vt : vts) vt.interrupt();
        for (Thread vt : vts) vt.join();

        System.gc();
        long after = rt.totalMemory() - rt.freeMemory();
        System.out.printf("VT 종료 후: %.2f MB (회수됨: %.2f MB)%n",
            (after - base) / 1_048_576.0,
            (withVt - after) / 1_048_576.0);
        // VT 종료 후 ThreadLocalMap이 GC됨 → 메모리 회수
    }
}
```

### 실험 2: ScopedValue vs ThreadLocal 메모리 비교

```java
import java.lang.ScopedValue;

public class ScopedVsThreadLocal {
    static final ScopedValue<String> SCOPED = ScopedValue.newInstance();
    static final ThreadLocal<String> LOCAL = new ThreadLocal<>();

    public static void main(String[] args) throws Exception {
        int count = 100_000;
        Runtime rt = Runtime.getRuntime();

        // ThreadLocal 방식
        System.gc();
        long base = rt.totalMemory() - rt.freeMemory();
        Thread[] vts = new Thread[count];
        for (int i = 0; i < count; i++) {
            final int id = i;
            vts[i] = Thread.ofVirtual().start(() -> {
                LOCAL.set("user-" + id);
                try { Thread.sleep(500); }
                catch (InterruptedException e) {}
                finally { LOCAL.remove(); }
            });
        }
        Thread.sleep(100);
        long tlMem = rt.totalMemory() - rt.freeMemory();
        System.out.printf("ThreadLocal %d VT: %.2f MB%n",
            count, (tlMem - base) / 1_048_576.0);
        for (Thread t : vts) { t.interrupt(); t.join(); }

        // ScopedValue 방식
        System.gc();
        base = rt.totalMemory() - rt.freeMemory();
        Thread[] vts2 = new Thread[count];
        for (int i = 0; i < count; i++) {
            final int id = i;
            vts2[i] = Thread.ofVirtual().start(() ->
                ScopedValue.where(SCOPED, "user-" + id)
                           .run(() -> {
                               try { Thread.sleep(500); }
                               catch (InterruptedException e) {}
                           })
            );
        }
        Thread.sleep(100);
        long svMem = rt.totalMemory() - rt.freeMemory();
        System.out.printf("ScopedValue %d VT: %.2f MB%n",
            count, (svMem - base) / 1_048_576.0);
        // ScopedValue가 ThreadLocal보다 메모리 효율적
        for (Thread t : vts2) { t.interrupt(); t.join(); }
    }
}
```

---

## 📊 성능/비용 비교

```
ThreadLocal vs ScopedValue 비교:

특성                  | ThreadLocal              | ScopedValue
─────────────────────┼─────────────────────────┼─────────────────────────────
VT당 메모리           | ThreadLocalMap 복사       | 스냅샷 배열 참조 (복사 없음)
100만 VT + 10 키      | ~수십 MB (맵 복사)       | ~수 MB (스냅샷 배열 참조)
읽기 성능            | 해시 조회 O(1)            | 배열 스캔 O(n), n=바인딩 수
쓰기                 | set/get/remove           | where().run() (스코프)
가변성               | 가변                     | 불변 (스코프 내)
자동 정리            | finally remove() 필요     | run() 종료 시 자동
자식 VT 전파         | InheritableThreadLocal    | StructuredTaskScope 자동
Spring 지원          | 광범위 (SecurityContextHolder, MDC)| 초기 단계 (Spring 6.x)

성능 비교 (1000만 get() 호출):
  ThreadLocal.get():   ~2ns (해시 조회)
  ScopedValue.get():   ~5~10ns (배열 스캔, 바인딩 수에 비례)
  → 읽기 빈도 높고 바인딩 수가 많으면 ScopedValue가 다소 느림
  → 대신 메모리 효율과 자동 정리로 대부분의 상황에서 낫다
```

---

## ⚖️ 트레이드오프

```
ThreadLocal vs ScopedValue 선택 기준:

ThreadLocal 유지가 적합한 경우:
  가변 상태 필요 (요청 중 값이 바뀜): 예) 트랜잭션 상태
  레거시 라이브러리 호환 (Spring Security, MDC)
  스레드별 재사용 가능 객체 (SimpleDateFormat, StringBuilder)
  VT 수가 적어 메모리 문제가 없는 경우

ScopedValue 적합한 경우:
  요청 컨텍스트 전달 (userId, traceId, locale)
  불변 컨텍스트 (한번 설정 후 읽기만)
  대량 VT 환경 (메모리 효율 중요)
  새로 작성하는 코드

마이그레이션 전략:
  1단계: -Djdk.tracePinnedThreads로 ThreadLocal 관련 Pinning 파악
  2단계: VT 수 증가 시 메모리 프로파일링 (ThreadLocal 크기)
  3단계: 요청 컨텍스트 → ScopedValue 점진적 전환
  4단계: Spring Security 등 프레임워크 ScopedValue 지원 대기

VT 풀링 금지 재확인:
  Executors.newFixedThreadPool(N, Thread.ofVirtual().factory())
  → N개 VT만 재사용 → ThreadLocal이 누적됨 → 절대 사용 금지
  → 대신 Executors.newVirtualThreadPerTaskExecutor() 사용
```

---

## 📌 핵심 정리

```
ThreadLocal / ScopedValue 핵심:

ThreadLocal의 VT 문제:
  각 VT에 ThreadLocalMap 독립 생성
  대량 VT = 대량 ThreadLocalMap 복사 → 메모리 압박
  풀링 시 이전 값 잔존 → 데이터 혼재 위험
  VT는 반드시 per-task 생성 (풀링 금지)

ScopedValue (Java 21):
  불변 컨텍스트를 스코프 내에서 공유
  복사 없이 참조 전달 → 메모리 효율
  run() 종료 시 자동 정리
  StructuredTaskScope에서 자식 VT 자동 상속

선택 기준:
  가변 상태 / 레거시 호환: ThreadLocal
  요청 컨텍스트 전달: ScopedValue (권장)
  스레드별 재사용 객체: ThreadLocal (적합)
  VT 대량 환경: ScopedValue 우선 고려

실무 패턴:
  MDC: VT별 독립 (장점), 반드시 clear()
  SecurityContext: INHERITABLETHREADLOCAL 또는 명시적 전달
  요청 ID, 사용자 ID: ScopedValue 마이그레이션 대상
```

---

## 🤔 생각해볼 문제

**Q1.** `ScopedValue`는 불변이므로 요청 처리 중 값을 변경해야 하는 경우(예: 권한 상승)에는 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

`ScopedValue`는 중첩 바인딩(nested binding)을 지원한다. 기존 값을 변경하는 것이 아니라 새 스코프에서 새 값으로 바인딩한다.

```java
static final ScopedValue<String> ROLE = ScopedValue.newInstance();

void handleRequest(String userId) {
    ScopedValue.where(ROLE, "USER").run(() -> {
        doNormalWork();  // ROLE.get() = "USER"

        if (needsElevation()) {
            // 일시적 권한 상승
            ScopedValue.where(ROLE, "ADMIN").run(() -> {
                doAdminWork();  // ROLE.get() = "ADMIN"
            });
            // 중첩 스코프 종료 후 자동으로 "USER"로 복원
        }

        doMoreWork();  // ROLE.get() = "USER" (복원됨)
    });
}
```

이 패턴은 스코프 기반 불변성을 유지하면서도 일시적인 컨텍스트 변경을 표현할 수 있다. 스코프 종료 시 자동 복원되므로 실수로 이전 컨텍스트가 오염될 위험이 없다.

</details>

---

**Q2.** `InheritableThreadLocal`로 VT에 컨텍스트를 전달하면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

`InheritableThreadLocal`은 부모 스레드 생성 시 자식 스레드에 값을 복사한다. VT를 100만 개 생성하면 100만 번 복사가 발생한다. 복사 비용과 메모리 사용이 문제다.

또한 복사는 VT 생성 시점의 스냅샷이다. 부모가 이후 값을 변경해도 이미 생성된 자식 VT에는 반영되지 않는다. 이로 인해 데이터 일관성 문제가 생길 수 있다.

더 심각한 문제: VT를 ExecutorService에서 재사용(풀링)하면 `InheritableThreadLocal`의 값이 누적되어 다른 요청의 데이터가 보일 수 있다. (VT 풀링 자체가 안티패턴이지만)

Java 21의 `StructuredTaskScope`를 사용하면 `ScopedValue`가 자식 VT에 자동으로 상속된다 (복사 없이 참조 공유). 이것이 VT 환경에서 컨텍스트 전달의 올바른 방법이다.

</details>

---

**Q3.** Spring Security의 `SecurityContextHolder`는 기본적으로 `ThreadLocal`을 사용한다. VT 환경에서 어떻게 설정해야 안전한가?

<details>
<summary>해설 보기</summary>

Spring Security 5.7+에서 VT 환경 설정:

1. **기본 설정 (THREAD_LOCAL 모드)**: 각 VT는 독립된 `SecurityContext`를 가짐. `Executors.newVirtualThreadPerTaskExecutor()`로 매번 새 VT를 생성하면 ThreadLocal이 격리되어 안전하다. 단 대량 VT에서 메모리 사용에 주의.

2. **INHERITABLETHREADLOCAL 모드**: 부모의 `SecurityContext`를 자식 VT에 복사. `Thread.sleep()`으로 다른 작업을 위임할 때 컨텍스트 유지. 단 복사 비용과 부모 변경이 전파되지 않는 문제.

3. **명시적 전달 (가장 안전)**:
```java
SecurityContext ctx = SecurityContextHolder.getContext();
try (var exe = Executors.newVirtualThreadPerTaskExecutor()) {
    exe.submit(() -> {
        SecurityContextHolder.setContext(ctx);
        try { doWork(); }
        finally { SecurityContextHolder.clearContext(); }
    });
}
```

4. **Spring Boot 3.2+ + spring.threads.virtual.enabled=true**: Spring이 자동으로 VT 친화적 SecurityContextHolder 전략을 선택한다.

실무에서는 Spring의 자동 설정을 신뢰하고 최신 버전을 사용하는 것이 가장 안전하다.

</details>

---

<div align="center">

**[⬅️ 이전: Pinning 문제](./04-pinning-problem.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spring Boot + Virtual Thread ➡️](./06-spring-boot-virtual-thread.md)**

</div>

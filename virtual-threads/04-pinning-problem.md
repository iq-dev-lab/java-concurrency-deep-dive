# Pinning 문제 — synchronized와 캐리어 스레드 고착

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `synchronized` 블록 안에서 블로킹 I/O 호출 시 Pinning이 발생하는 이유는?
- `-Djdk.tracePinnedThreads=full`과 JFR `jdk.VirtualThreadPinned`로 어떻게 진단하는가?
- `ReentrantLock`으로 교체하면 왜 Pinning이 해결되는가?
- Pinning이 성능에 미치는 구체적인 영향은?
- 모든 `synchronized`를 제거해야 하는가, 아니면 특정 패턴만?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Pinning은 Virtual Thread의 가장 큰 함정이다. 기존 코드에 `synchronized`가 있고 그 안에서 I/O가 발생한다면, VT로 전환해도 처리량이 전혀 개선되지 않거나 오히려 나빠질 수 있다. 특히 JDK 라이브러리와 서드파티 라이브러리에 hidden `synchronized`가 있어서 예상치 못한 Pinning이 발생하는 경우가 많다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: synchronized가 있어도 VT는 Unmount된다는 오해
  synchronized (lock) {
      result = db.query("SELECT ...");  // 이 호출이 내부적으로 블로킹
  }
  "synchronized를 썼어도 VT니까 db.query 동안 Unmount되겠지"
  → 아님! synchronized 블록 내 모든 블로킹은 Pinning
  → 캐리어 스레드가 db.query 완료까지 차지됨

실수 2: 내부 synchronized가 없다고 가정
  // 내 코드: synchronized 없음
  try (var exe = Executors.newVirtualThreadPerTaskExecutor()) {
      exe.submit(() -> {
          connection.prepareStatement("SELECT ...");  // 내부에 synchronized!
      });
  }
  → JDBC 드라이버가 내부적으로 synchronized 사용 → Pinning
  → -Djdk.tracePinnedThreads=full 로 확인 필수

실수 3: Pinning 발생 시 VT가 완전히 멈춘다는 오해
  "Pinning되면 데드락이 생기나?"
  → 아님! JVM이 Pinning 감지 시 캐리어 풀을 확장
     (maximumPoolSize=256까지)
  → 처리량 저하, 메모리 증가지만 데드락은 아님
  → 단, 256개 캐리어 모두 Pinning되면 처리 불가
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Pinning 해결: synchronized → ReentrantLock

// Before (Pinning 발생):
class DataAccessObject {
    private final Object lock = new Object();

    public String fetchData(String key) {
        synchronized (lock) {                 // ← Pinning 원인
            return httpClient.get(url + key); // ← 블로킹 I/O
        }
    }
}

// After (Pinning 해결):
class DataAccessObject {
    private final ReentrantLock lock = new ReentrantLock();

    public String fetchData(String key) {
        lock.lock();                          // ← AQS park → Unmount OK
        try {
            return httpClient.get(url + key); // ← VT Unmount 가능
        } finally {
            lock.unlock();
        }
    }
}

// 또는 락이 필요 없다면 제거:
class DataAccessObject {
    // 상태 없는 객체라면 락 자체가 불필요
    public String fetchData(String key) {
        return httpClient.get(url + key);  // 락 없이 안전
    }
}

// synchronized를 유지해야 하는 경우:
// → I/O를 synchronized 밖으로 이동
class DataAccessObject {
    private final Object lock = new Object();

    public String fetchData(String key) {
        // I/O를 락 밖에서 수행
        String data = httpClient.get(url + key);  // Unmount OK
        synchronized (lock) {
            // 빠른 상태 업데이트만 락 내에서
            cache.put(key, data);
        }
        return data;
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Pinning이 발생하는 이유

```
JVM 모니터(synchronized)의 구현 제약:

synchronized 블록 진입 시:
  Mark Word에 LockRecord 주소 저장 (Thin Lock)
  또는 ObjectMonitor 생성 (Fat Lock)
  ObjectMonitor._owner = 현재 OS 스레드 포인터 저장!

문제:
  ObjectMonitor는 OS 스레드를 가리킴 (JVM 내부 구조)
  VT가 Unmount되면 → 캐리어(OS 스레드)가 바뀔 수 있음
  하지만 ObjectMonitor._owner가 이전 캐리어를 가리킴
  → 다른 캐리어에서 Monitor 해제 불가 (소유자 불일치)

따라서:
  synchronized 블록 내에서는 VT가 캐리어에 묶임 (Pinned)
  park 시도해도 Unmount 금지 → 캐리어도 함께 블로킹

ReentrantLock이 Pinning 없는 이유:
  AQS의 state 필드는 VT의 정체성(Thread 객체)을 추적
  Thread 객체는 VT 자체를 가리킴 (캐리어가 아님)
  park() → VT Unmount → 캐리어 바뀜 → VT Mount
  새 캐리어에서 lock() 재시도 → AQS state 비교: VT == owner
  → 정상 동작

JNI에서도 Pinning:
  네이티브 코드는 JVM 스택과 다른 구조
  Unmount 시 네이티브 스택도 복사 불가 (JVM 제어 밖)
  → JNI 실행 중 park → Pinning

Pinning 완화 메커니즘:
  JVM이 Pinning 감지 시 캐리어 풀 확장 시도
  (jdk.virtualThreadScheduler.maxPoolSize, 기본 256)
  캐리어가 추가로 생성되어 다른 VT 실행 가능
  → Pinning된 캐리어 수 < maxPoolSize이면 시스템 유지
```

### 2. Pinning 진단 방법

```bash
# 방법 1: JVM 플래그 (개발/테스트 환경)
java -Djdk.tracePinnedThreads=full MyApp

# Pinning 발생 시 출력:
# Thread[#27,ForkJoinPool-1-worker-1,5,CarrierThreads]
#     com.example.DataAccessObject.fetchData(DataAccessObject.java:15) <== monitors:1
#         com.example.DataAccessObject@1234 (owner: 0)
# ...

# 방법 2: JVM 플래그 (요약)
java -Djdk.tracePinnedThreads=short MyApp
# Thread[#27,...] at com.example.DataAccessObject.fetchData(DataAccessObject.java:15)

# 방법 3: JFR (운영 환경 권장)
jcmd <pid> JFR.start duration=30s filename=pinning.jfr
# 또는 시작 시: -XX:StartFlightRecording=duration=30s,filename=pinning.jfr

# JFR 분석
jfr print --events jdk.VirtualThreadPinned pinning.jfr
# 출력: stackTrace (Pinning 발생 지점), duration (Pinning 지속 시간)

# 방법 4: async-profiler (상세 분석)
# -e jdk.VirtualThreadPinned 모드로 FlameGraph 생성
./asprof -d 30 -e jdk.VirtualThreadPinned -f pinning-flame.html <pid>
```

### 3. Pinning 발생 상황별 해결 전략

```
전략 1: synchronized → ReentrantLock (가장 일반적)

Before:
  synchronized (this) {
      result = slowOperation();  // I/O 또는 sleep
  }

After:
  lock.lock();
  try {
      result = slowOperation();  // Unmount 가능
  } finally { lock.unlock(); }

전략 2: I/O를 synchronized 밖으로 이동

Before:
  synchronized (this) {
      data = httpClient.fetch(url);  // I/O → Pinning
      cache.put(key, data);
  }

After:
  String data = httpClient.fetch(url);  // 락 없이 I/O
  synchronized (this) {               // 빠른 상태 업데이트만
      cache.put(key, data);
  }

전략 3: 서드파티 라이브러리 Pinning 우회
  # JDBC 드라이버 Pinning 시:
  # 옵션 A: 최신 버전 드라이버 사용 (VT 호환 버전)
  # 옵션 B: 별도 스레드 풀에서 JDBC 실행 (VT 미사용)
  ExecutorService jdbcPool = Executors.newFixedThreadPool(poolSize);
  CompletableFuture.supplyAsync(() -> repo.findById(id), jdbcPool);

전략 4: Pinning이 허용 가능한지 평가
  Pinning 지속 시간이 짧다면 (< 1ms)?
    → maxPoolSize 내에서 캐리어 확장으로 커버 가능
    → 무조건 제거 불필요
  Pinning 지속 시간이 길다면 (> 10ms)?
    → 반드시 해결
  Pinning이 동시에 많이 발생한다면?
    → maxPoolSize 초과 위험 → 반드시 해결
```

---

## 💻 실전 실험

### 실험 1: Pinning 의도적 재현

```java
public class PinningDemo {
    private static final Object MONITOR = new Object();

    public static void main(String[] args) throws Exception {
        // JVM 옵션: -Djdk.tracePinnedThreads=full

        int tasks = 20;
        int carriers = Runtime.getRuntime().availableProcessors();
        System.out.println("캐리어: " + carriers + ", VT 작업: " + tasks);

        // Pinning 있는 버전
        long start = System.nanoTime();
        CountDownLatch latch = new CountDownLatch(tasks);
        try (var exe = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < tasks; i++) {
                exe.submit(() -> {
                    synchronized (MONITOR) {      // Pinning 시작
                        Thread.sleep(100);        // 캐리어도 100ms 블로킹!
                    }                             // Pinning 해제
                    latch.countDown();
                    return null;
                });
            }
            latch.await();
        }
        System.out.printf("Pinning 있음: %d ms (기대 > %d ms)%n",
            (System.nanoTime() - start) / 1_000_000,
            100 * tasks / carriers);

        // Pinning 없는 버전 (ReentrantLock)
        ReentrantLock lock = new ReentrantLock();
        start = System.nanoTime();
        CountDownLatch latch2 = new CountDownLatch(tasks);
        try (var exe = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < tasks; i++) {
                exe.submit(() -> {
                    lock.lock();
                    try {
                        Thread.sleep(100);  // VT Unmount (캐리어 해방)
                    } finally { lock.unlock(); }
                    latch2.countDown();
                    return null;
                });
            }
            latch2.await();
        }
        System.out.printf("Pinning 없음: %d ms (기대 ~%d ms)%n",
            (System.nanoTime() - start) / 1_000_000,
            100 * tasks);  // 직렬화로 100ms × tasks / 병렬도
        // Pinning 있음: ~100 × (tasks/carriers) ms
        // Pinning 없음: ~100 × tasks ms (단, ReentrantLock이 직렬화하므로 동일하지 않음)
        // 핵심: Pinning 있음에서 캐리어가 부족해 전체가 느려짐
    }
}
```

### 실험 2: JFR으로 Pinning 위치 찾기

```bash
cat > PinningJfrTest.java << 'EOF'
public class PinningJfrTest {
    static void hiddenPinning() throws Exception {
        synchronized (PinningJfrTest.class) {
            Thread.sleep(50);  // Pinning!
        }
    }

    public static void main(String[] args) throws Exception {
        try (var exe = java.util.concurrent.Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 100; i++) {
                exe.submit(() -> { hiddenPinning(); return null; });
            }
        }
    }
}
EOF

javac PinningJfrTest.java

# JFR 기록
java -XX:StartFlightRecording=duration=5s,filename=pinning.jfr PinningJfrTest

# Pinning 이벤트 분석
jfr print --events jdk.VirtualThreadPinned pinning.jfr 2>/dev/null | head -50
# 출력에서 stackTrace 확인:
# "at PinningJfrTest.hiddenPinning(PinningJfrTest.java:3)"
# duration: "50.xxx ms"
```

---

## 📊 성능/비용 비교

```
Pinning이 처리량에 미치는 영향:

시나리오: 4코어, 100ms I/O, 100 동시 요청

방법                   | 처리 시간   | 이유
──────────────────────┼───────────┼─────────────────────────────────
OS 스레드 풀 100개       | ~100ms    | 100개 동시 처리
VT + Pinning 없음      | ~100ms    | 캐리어 해방, 동시 처리
VT + Pinning (4코어)   | ~2500ms   | 캐리어 4개 × 25번 = 100ms × 25
VT + Pinning + 캐리어 보충| ~500ms   | maxPoolSize 20개로 보충

Pinning + 캐리어 고갈 시나리오:
  synchronized 내 100ms I/O
  동시 VT 100개
  캐리어 기본 4개 → maxPoolSize 256개까지 생성
  256개 모두 Pinning되면 → 완전 멈춤 (새 VT 실행 불가)

JVM Pinning 보완 메커니즘:
  Pinning 감지 시 캐리어 추가 (최대 maxPoolSize)
  기본 maxPoolSize = 256 (조정 가능)
  -Djdk.virtualThreadScheduler.maxPoolSize=512

Pinning 지속 시간별 영향:
  < 1ms:  캐리어 보충으로 대부분 커버
  1~10ms: 처리량 눈에 띄게 감소
  > 10ms: 심각한 성능 문제
  > 1s:   서비스 불능 위험
```

---

## ⚖️ 트레이드오프

```
Pinning 해결 전략별 트레이드오프:

synchronized → ReentrantLock 교체:
  장점: Pinning 완전 해결, 추가 기능 (tryLock, 공정성)
  단점: 코드 변경 필요, try-finally 추가
  적합: 새 코드 또는 리팩토링 가능한 코드

I/O를 synchronized 밖으로 이동:
  장점: synchronized 유지 (JIT Lock Elision 등 최적화 유지)
  단점: 구조 변경 필요, 일부 경우 로직 복잡화
  적합: 복잡한 synchronized 블록에서 I/O만 분리 가능할 때

별도 OS 스레드 풀로 격리:
  장점: 코드 변경 최소
  단점: VT의 이점을 일부 포기, 추가 풀 관리
  적합: 서드파티 라이브러리 Pinning 우회

maxPoolSize 증가:
  장점: 즉각 적용 가능
  단점: 근본 해결 아님, 메모리 증가 (OS 스레드 추가)
  적합: 임시 완화, Pinning 지속 시간이 짧을 때

모든 synchronized 제거가 맞는가?:
  synchronized 내 I/O 없고, 빠른 연산만 → 유지 가능
  synchronized 내 블로킹 있음 → 반드시 해결
  Java 21 JEP 376: 향후 JVM이 Pinning 없이 synchronized 지원 목표
  → 미래에는 이 제약이 해제될 예정
```

---

## 📌 핵심 정리

```
Pinning 핵심:

발생 원인:
  synchronized의 JVM 모니터가 OS 스레드 포인터를 저장
  → VT Unmount 시 캐리어 바뀜 → 모니터 소유자 불일치
  → synchronized 내에서 park 금지 → 캐리어도 함께 블로킹

발생 조건:
  synchronized 블록 내 모든 블로킹:
    Thread.sleep(), Object.wait() (예외), I/O, ReentrantLock.lock()
  JNI 실행 중 블로킹

진단:
  -Djdk.tracePinnedThreads=full  (표준 출력)
  JFR: jdk.VirtualThreadPinned  (운영 환경)
  async-profiler: -e jdk.VirtualThreadPinned

해결:
  synchronized → ReentrantLock (가장 확실)
  I/O를 synchronized 밖으로 이동
  서드파티 라이브러리: 최신 버전 또는 별도 풀

JVM 보완:
  Pinning 감지 → 캐리어 풀 확장 (최대 maxPoolSize=256)
  근본 해결 아님, 임시 완화

미래:
  JEP 491 (Java 24+): synchronized가 Pinning 없이 동작하도록 개선
```

---

## 🤔 생각해볼 문제

**Q1.** `synchronized` 내에서 `Object.wait()`는 Unmount가 된다고 했다. 그렇다면 `synchronized` 내에서 `ReentrantLock.lock()`을 대기하면 Pinning인가?

<details>
<summary>해설 보기</summary>

Pinning이 된다. `ReentrantLock.lock()`이 획득 실패 시 `LockSupport.park()`를 호출하는데, 이 park가 `synchronized` 블록 내에서 호출되면 Pinning이 발생한다.

`Object.wait()`가 예외인 이유: `wait()`는 JVM이 특별히 처리한다. `wait()` 호출 시 모니터 락을 해제하고, VT를 Wait Set으로 이동시킨 후 park한다. 이 시점에 "모니터 블록을 탈출한 상태"이므로 VT가 Unmount 가능하다.

반면 `ReentrantLock.lock()` 대기는 모니터 블록을 탈출하지 않는다. `synchronized (monitor) { reentrantLock.lock(); }` = 모니터 내에서 park → Pinning.

결론: `synchronized` 내에서 `Object.wait()`만 Unmount 가능. 다른 모든 블로킹은 Pinning.

</details>

---

**Q2.** JEP 491 (Java 24)에서 `synchronized`가 Pinning 없이 동작한다면, 기존 코드의 `synchronized`를 `ReentrantLock`으로 바꿀 필요가 없어지는가?

<details>
<summary>해설 보기</summary>

JEP 491이 완전히 적용되면 `synchronized` 내에서 블로킹 I/O를 해도 Pinning이 발생하지 않게 된다. 이는 JVM 모니터 구현을 VT를 인식하도록 개선하여, Unmount 시 모니터 소유권을 VT에 바인딩하는 방식으로 해결한다.

Java 24 (현재 작업 중, 2025년 예정):
- `synchronized` Pinning 해결
- 기존 코드를 `ReentrantLock`으로 바꾸지 않아도 됨

단, `ReentrantLock`이 `synchronized`보다 나은 점은 여전히 존재:
- `tryLock()` (타임아웃, 비블로킹)
- `lockInterruptibly()`
- 공정성 설정
- 다수의 `Condition`

따라서 JEP 491 이후에도 새 코드에서는 상황에 따라 `ReentrantLock`이 더 적합한 경우가 있다. 다만 VT에서 Pinning을 이유로 반드시 교체해야 하는 강제성은 없어진다.

</details>

---

**Q3.** HikariCP 커넥션 풀을 Virtual Thread 환경에서 사용할 때 Pinning 문제가 있는가?

<details>
<summary>해설 보기</summary>

HikariCP 최신 버전(5.1.0+)은 Virtual Thread와 호환된다. HikariCP 개발팀이 VT 환경을 인식하고 내부 `synchronized`를 최소화하거나 `ReentrantLock`으로 교체했다.

구체적으로:
- HikariCP 5.0.x 이하: 일부 내부 `synchronized`로 Pinning 가능
- HikariCP 5.1.0+: `ConcurrentBag`의 락 구조 개선으로 Pinning 최소화

확인 방법:
```bash
java -Djdk.tracePinnedThreads=short -jar myapp.jar
# HikariCP Pinning이 있으면 HikariCP 클래스가 스택에 보임
```

그래도 남는 Pinning: JDBC 드라이버 자체 (PostgreSQL, MySQL 드라이버 내부)가 `synchronized`를 사용할 수 있다. 드라이버 최신 버전과 VT 호환성 릴리즈 노트를 확인해야 한다:
- PostgreSQL JDBC 42.6.0+: VT 호환 개선
- MySQL Connector/J 8.0.33+: VT 호환 개선

</details>

---

<div align="center">

**[⬅️ 이전: 블로킹 시 동작](./03-blocking-unmount.md)** | **[홈으로 🏠](../README.md)** | **[다음: ThreadLocal과 ScopedValue ➡️](./05-threadlocal-scopedvalue.md)**

</div>

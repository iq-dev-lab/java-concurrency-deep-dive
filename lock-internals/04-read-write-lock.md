# ReadWriteLock — 읽기/쓰기 분리와 Write Starvation

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ReentrantReadWriteLock`이 하나의 `state` 필드로 읽기/쓰기를 어떻게 분리하는가?
- 읽기가 많은 환경에서 Write Starvation이 발생하는 조건은 무엇인가?
- 읽기 락에서 쓰기 락으로 업그레이드가 불가능한 이유는?
- 실제 Cache 구현에서 ReadWriteLock이 어떻게 활용되는가?
- `StampedLock`의 낙관적 읽기와의 성능 차이는 어느 상황에서 갈리는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

많은 시스템에서 읽기 연산이 쓰기보다 훨씬 빈번하다. 캐시, 설정, 라우팅 테이블이 대표적이다. 이런 상황에서 `synchronized`는 읽기끼리도 직렬화시켜 병렬성을 낭비한다. `ReentrantReadWriteLock`은 "읽기끼리는 동시에, 쓰기는 배타적으로"를 구현한다. 하지만 잘못 사용하면 Write Starvation이라는 함정에 빠진다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 읽기 락 보유 중에 쓰기 락 획득 시도 (데드락)
  readLock.lock();
  try {
      if (needsUpdate()) {
          writeLock.lock();   // 데드락! 읽기 락을 놓지 않고 쓰기 락 시도
          try { update(); } finally { writeLock.unlock(); }
      }
  } finally { readLock.unlock(); }

  이유: 쓰기 락은 모든 읽기 락이 해제될 때까지 대기
        현재 스레드가 읽기 락을 보유 중이므로 영원히 대기

실수 2: 읽기 비중이 적을 때 ReadWriteLock 사용
  쓰기가 50% 이상이면 ReadWriteLock의 관리 오버헤드가 손해
  → synchronized가 오히려 더 빠를 수 있음

실수 3: Write Starvation 무시
  읽기 스레드가 계속 들어오는 환경에서 쓰기 스레드가
  수 초 이상 대기 → 타임아웃, 데이터 불일치
  → Fair 모드나 StampedLock 검토 필요
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// Cache 구현: ReadWriteLock 올바른 활용
class SafeCache<K, V> {
    private final Map<K, V> map = new HashMap<>();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock  = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    public V get(K key) {
        readLock.lock();  // 읽기 락: 여러 스레드 동시 가능
        try {
            return map.get(key);
        } finally {
            readLock.unlock();
        }
    }

    public void put(K key, V value) {
        writeLock.lock();  // 쓰기 락: 완전 배타적
        try {
            map.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }

    // 쓰기 락 다운그레이드 (쓰기 → 읽기): 허용됨
    public V getOrLoad(K key, Supplier<V> loader) {
        writeLock.lock();
        try {
            V v = map.get(key);
            if (v == null) {
                v = loader.get();
                map.put(key, v);
            }
            readLock.lock();  // 읽기 락 먼저 획득
            return v;
        } finally {
            writeLock.unlock();  // 쓰기 락 해제 (읽기 락은 유지)
            readLock.unlock();   // 나중에 읽기 락 해제
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. state 필드의 비트 분할

```
ReentrantReadWriteLock의 핵심 설계:
  AQS의 단일 int state(32비트)를 두 부분으로 분할

  ┌──────────────────────────────────────────────────────────┐
  │  32비트 state 필드                                        │
  │                                                          │
  │  상위 16비트 (bit 31~16)   하위 16비트 (bit 15~0)         │
  │  [ 읽기 락 보유 수 ]        [ 쓰기 락 재진입 카운터 ]       │
  │                                                          │
  │  MAX 읽기 동시 보유: 65535 (2^16 - 1)                    │
  │  MAX 쓰기 재진입 깊이: 65535 (2^16 - 1)                  │
  └──────────────────────────────────────────────────────────┘

  비트 연산:
  static final int SHARED_SHIFT = 16;
  static final int SHARED_UNIT  = (1 << 16);  // 읽기 락 1개 = 0x0001_0000
  static final int MAX_COUNT    = (1 << 16) - 1;
  static final int EXCLUSIVE_MASK = (1 << 16) - 1;  // 하위 16비트 마스크

  읽기 보유 수 추출:  state >>> SHARED_SHIFT  (상위 16비트)
  쓰기 보유 수 추출:  state & EXCLUSIVE_MASK  (하위 16비트)

  예시:
  읽기 3개 보유: state = 3 << 16 = 0x0003_0000
  쓰기 1개 보유: state = 1       = 0x0000_0001
  읽기 2, 쓰기 1 동시: 불가 (쓰기는 배타적)
```

### 2. 읽기 락 획득/해제

```
읽기 락 획득 조건:
  ① 쓰기 락이 없음 (state & EXCLUSIVE_MASK == 0)
  ② 또는 쓰기 락을 현재 스레드가 보유 (읽기/쓰기 다운그레이드 위한 허용)

  tryAcquireShared(1):
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current) {
        return -1;  // 다른 스레드가 쓰기 락 보유 → 대기
    }
    int r = sharedCount(c);
    if (!readerShouldBlock() && r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        // 읽기 카운트 +1 성공
        return 1;
    }
    // CAS 실패 시 fullTryAcquireShared로 재시도

읽기 락 해제:
  tryReleaseShared(1):
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;  // 읽기 카운트 -1
        if (compareAndSetState(c, nextc)) {
            return nextc == 0;  // 0이면 쓰기 대기 스레드 unpark
        }
    }

스레드별 읽기 보유 수 추적:
  여러 스레드가 동시에 읽기 락을 보유 → 각 스레드의 보유 수 별도 추적
  ThreadLocal<HoldCounter> readHolds 사용
  HoldCounter: 스레드 ID + 재진입 카운터
  → 읽기 락 재진입 시 카운터 증가
```

### 3. Write Starvation 발생 메커니즘

```
Non-Fair 모드에서 Write Starvation:

시나리오:
  Thread W: 쓰기 락 대기 중 (큐에 있음)
  Thread R1, R2, R3: 계속 읽기 락 획득 후 해제 반복

  ┌────────────────────────────────────────────────────────┐
  │  시간 →                                                │
  │                                                       │
  │  R1: [읽기락 보유]    [해제]                            │
  │  R2:      [읽기락 보유]    [해제]                       │
  │  R3:           [읽기락 보유]    [해제]                  │
  │  W:  [대기 중...→ 읽기락이 항상 있어서 획득 불가]        │
  │                                                       │
  └────────────────────────────────────────────────────────┘

  읽기 락 획득 조건: 쓰기 락이 없음
  → R1~R3이 교대로 읽기 락을 보유하면 쓰기 보유 수 = 0인 순간 없음
  → W는 영원히 대기

해결책 1: Fair 모드
  new ReentrantReadWriteLock(true)
  큐에 쓰기 대기자가 있으면 새 읽기 락 획득 불가 (readerShouldBlock=true)
  → 쓰기 대기자가 있으면 읽기 대기자도 큐에 진입
  단점: 읽기 성능 저하

해결책 2: StampedLock (낙관적 읽기)
  읽기 중에도 쓰기가 들어올 수 있음
  validateStamp()로 쓰기 발생 여부 확인
  → Write Starvation 구조적 해결 (다음 문서)

해결책 3: 읽기 락 보유 시간 최소화
  읽기 락 내부에서 무거운 연산 제거
  읽기 후 복사(Copy-on-Read) 패턴으로 락 외부에서 처리
```

### 4. 락 다운그레이드 (쓰기 → 읽기)

```
허용: 쓰기 락 보유 중 읽기 락 획득 후 쓰기 락 해제 (다운그레이드)
금지: 읽기 락 보유 중 쓰기 락 획득 시도 (업그레이드 불가)

다운그레이드 패턴:
  writeLock.lock();
  try {
      updateData();          // 데이터 변경
      readLock.lock();       // 읽기 락 먼저 획득 (쓰기 락 유지 중)
  } finally {
      writeLock.unlock();    // 쓰기 락 해제 (읽기 락은 유지)
  }
  try {
      readData();            // 읽기 락으로 읽기
  } finally {
      readLock.unlock();
  }

왜 업그레이드(읽기 → 쓰기)가 불가한가:
  Thread A: 읽기 락 보유 + 쓰기 락 대기
  Thread B: 읽기 락 보유 + 쓰기 락 대기
  → A는 B의 읽기 락을 기다리고
    B는 A의 읽기 락을 기다림
  → 데드락!

  따라서 ReentrantReadWriteLock은 업그레이드를 지원하지 않음
  업그레이드 필요 시: 읽기 락 해제 → 쓰기 락 획득 (그 사이 다른 스레드가 수정 가능)
  → 읽은 데이터를 재검증해야 함
```

---

## 💻 실전 실험

### 실험 1: 읽기/쓰기 비율별 성능 측정

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Threads(8)
public class ReadWriteLockBenchmark {

    private final ReentrantLock mutex = new ReentrantLock();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private int data = 0;

    @Benchmark
    @Group("mutex_read_heavy")  // 읽기 90%, 쓰기 10%
    @GroupThreads(7)
    public int mutex_read() {
        mutex.lock();
        try { return data; } finally { mutex.unlock(); }
    }

    @Benchmark
    @Group("mutex_read_heavy")
    @GroupThreads(1)
    public void mutex_write() {
        mutex.lock();
        try { data++; } finally { mutex.unlock(); }
    }

    @Benchmark
    @Group("rwlock_read_heavy")
    @GroupThreads(7)
    public int rw_read() {
        rwLock.readLock().lock();
        try { return data; } finally { rwLock.readLock().unlock(); }
    }

    @Benchmark
    @Group("rwlock_read_heavy")
    @GroupThreads(1)
    public void rw_write() {
        rwLock.writeLock().lock();
        try { data++; } finally { rwLock.writeLock().unlock(); }
    }
}
// 결과 (읽기 비중 높을수록 rwLock 유리):
// 읽기 90%: rwLock 2~4배 빠름
// 읽기 50%: 비슷하거나 mutex가 빠를 수도
// 읽기 10%: mutex가 빠름 (rwLock 오버헤드)
```

### 실험 2: Write Starvation 재현

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;
import java.util.concurrent.atomic.AtomicLong;

public class WriteStarvationDemo {
    static final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock(false); // Non-Fair
    static final AtomicLong readCount  = new AtomicLong();
    static final AtomicLong writeCount = new AtomicLong();

    public static void main(String[] args) throws InterruptedException {
        // 읽기 스레드 7개 (지속적으로 읽기 락 획득)
        for (int i = 0; i < 7; i++) {
            new Thread(() -> {
                while (!Thread.currentThread().isInterrupted()) {
                    rwLock.readLock().lock();
                    try {
                        Thread.sleep(10);  // 읽기 작업 시뮬레이션
                        readCount.incrementAndGet();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    } finally {
                        rwLock.readLock().unlock();
                    }
                }
            }).start();
        }

        // 쓰기 스레드 1개
        Thread writer = new Thread(() -> {
            long start = System.currentTimeMillis();
            rwLock.writeLock().lock();
            long waited = System.currentTimeMillis() - start;
            try {
                System.out.printf("쓰기 락 획득! 대기 시간: %dms%n", waited);
                writeCount.incrementAndGet();
            } finally {
                rwLock.writeLock().unlock();
            }
        });
        writer.start();

        Thread.sleep(3000);
        System.out.printf("읽기 완료 수: %d, 쓰기 완료 수: %d%n",
            readCount.get(), writeCount.get());
        // 쓰기 완료 수가 0일 가능성 (Write Starvation)
    }
}
```

### 실험 3: 락 다운그레이드 패턴 검증

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class LockDowngradeTest {
    static final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    static volatile int value = 0;

    public static void main(String[] args) throws InterruptedException {
        // 올바른 다운그레이드
        rwLock.writeLock().lock();
        System.out.println("쓰기 락 획득");
        try {
            value = 42;
            rwLock.readLock().lock();   // 읽기 락 먼저 획득
            System.out.println("읽기 락 획득 (다운그레이드 준비)");
        } finally {
            rwLock.writeLock().unlock();  // 쓰기 락 해제
            System.out.println("쓰기 락 해제. 읽기 락 유지 중");
        }

        // 이 구간: 읽기 락만 보유
        System.out.println("읽기 값: " + value);  // 42 보장

        rwLock.readLock().unlock();
        System.out.println("읽기 락 해제. 다운그레이드 완료");

        // 업그레이드 시도 → 데드락 확인 (타임아웃으로 방지)
        rwLock.readLock().lock();
        System.out.println("읽기 락 재획득");
        boolean upgraded = rwLock.writeLock().tryLock();  // 업그레이드 시도
        System.out.println("업그레이드 성공: " + upgraded);  // false
        if (upgraded) rwLock.writeLock().unlock();
        rwLock.readLock().unlock();
    }
}
```

---

## 📊 성능/비용 비교

```
읽기/쓰기 비율별 최적 선택:

읽기 비율  | 권장 방법          | 이유
─────────┼────────────────────┼──────────────────────────────────────
90%+     | ReadWriteLock 또는  | 읽기 병렬화로 큰 이득
         | StampedLock         | StampedLock은 낙관적 읽기로 더 빠름
70~90%   | ReadWriteLock       | 읽기 병렬화 이득 > 오버헤드
50~70%   | 측정 후 결정        | 오버헤드와 이득이 비슷
50%미만  | synchronized 또는   | 단순 뮤텍스가 오버헤드 적음
         | ReentrantLock       |

오버헤드 비교 (단일 스레드):
  synchronized:         CAS 1회
  ReentrantLock:        AQS CAS 1회
  ReadWriteLock 읽기:   AQS CAS + ThreadLocal HoldCounter 업데이트
  ReadWriteLock 쓰기:   AQS CAS
  → 읽기 락이 쓰기 락보다 약간 더 비쌈 (ThreadLocal 때문)
```

---

## ⚖️ 트레이드오프

```
ReadWriteLock 트레이드오프:

장점:
  읽기 병렬화로 읽기 집약적 워크로드 처리량 향상
  쓰기는 배타적으로 데이터 일관성 보장
  다운그레이드 지원으로 안전한 상태 전환

단점:
  Write Starvation (Non-Fair 모드)
  업그레이드 불가 → 쓰기 필요 시 락 해제/재획득 필요
  읽기 락 오버헤드 (ThreadLocal 조회)
  쓰기가 많으면 오히려 느림

ReadWriteLock vs StampedLock:
  ReadWriteLock:
    명시적 획득/해제, 재진입 지원
    Write Starvation 가능
    API가 직관적
  StampedLock (다음 문서):
    낙관적 읽기로 훨씬 빠른 읽기
    재진입 불가, API 복잡
    쓰기 빈도 낮을 때 최적

실무 권장:
  읽기 집약적 캐시, 설정, 라우팅 테이블 → ReadWriteLock
  극한의 읽기 성능 필요 → StampedLock (주의사항 숙지 필수)
  동시성이 낮거나 쓰기가 많음 → synchronized/ReentrantLock
```

---

## 📌 핵심 정리

```
ReadWriteLock 핵심:

state 비트 구조:
  상위 16비트: 읽기 락 보유 스레드 수
  하위 16비트: 쓰기 락 재진입 카운터

읽기/쓰기 규칙:
  읽기 동시: 여러 스레드가 동시에 읽기 락 보유 가능
  쓰기 배타: 쓰기 중엔 읽기/쓰기 모두 대기
  다운그레이드 허용: 쓰기 락 보유 중 읽기 락 획득 → 쓰기 락 해제
  업그레이드 불가: 읽기 락 보유 중 쓰기 락 획득 → 데드락

Write Starvation:
  Non-Fair 모드에서 읽기 스레드가 계속 들어오면
  쓰기 스레드가 영원히 대기
  해결: Fair 모드, StampedLock, 읽기 시간 최소화

적합한 상황:
  읽기 >> 쓰기 (읽기 70% 이상)
  읽기 시간이 길고 쓰기 시간이 짧음
  Write Starvation 위험이 없는 환경
```

---

## 🤔 생각해볼 문제

**Q1.** `ReentrantReadWriteLock`에서 쓰기 락을 보유한 스레드가 읽기 락을 획득할 수 있는가? 그 반대는?

<details>
<summary>해설 보기</summary>

**쓰기 락 보유 → 읽기 락 획득: 가능** (락 다운그레이드). `tryAcquireShared()` 내부에서 쓰기 락을 현재 스레드가 보유 중이면 읽기 락 획득을 허용하는 조건이 있다.

**읽기 락 보유 → 쓰기 락 획득: 불가**. 데드락이 발생한다. 쓰기 락은 "모든 읽기 락이 해제될 때까지" 대기하는데, 현재 스레드가 읽기 락을 보유 중이므로 해제 조건이 절대 충족되지 않는다.

실용적 패턴: 업그레이드가 필요하면 읽기 락을 해제하고 쓰기 락을 획득하되, 그 사이 데이터가 변경됐을 수 있으므로 재검증이 필요하다. `StampedLock`의 낙관적 읽기 패턴이 이 문제를 우아하게 해결한다.

</details>

---

**Q2.** `state` 필드의 상위 16비트가 읽기 카운트라면, 동시에 65535개 이상의 스레드가 읽기 락을 획득하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`tryAcquireShared()` 내부에서 `r < MAX_COUNT` 조건을 확인한다. 읽기 락 보유 수가 65535에 도달하면 이 조건이 거짓이 되어 새 읽기 락 획득이 실패하고 대기 큐에 진입한다.

실무에서 동시에 65535개 스레드가 같은 읽기 락을 보유하는 상황은 현실적이지 않다(그 정도 스레드 수면 OOM이 먼저 발생한다). 하지만 이론적 한계가 있음을 인지해야 한다.

재진입의 경우 `HoldCounter`에 스레드별 카운트가 쌓이므로, 하나의 스레드가 재진입을 65535회 이상 하면 오버플로우가 발생할 수 있다. `Error`가 던져진다.

</details>

---

**Q3.** `Collections.synchronizedMap()`과 `ReadWriteLock`으로 감싼 HashMap의 성능은 읽기 집약적 환경에서 얼마나 차이가 나는가?

<details>
<summary>해설 보기</summary>

`Collections.synchronizedMap()`은 내부적으로 `synchronized` 블록을 사용하여 읽기/쓰기 모두 직렬화한다. 8스레드가 모두 읽기만 하는 경우 하나씩 순차 실행된다.

`ReadWriteLock`으로 감싼 HashMap은 읽기 스레드들이 동시에 실행된다. 이론적으로 N개의 읽기 스레드가 있으면 N배 처리량이 가능하다(락 획득 오버헤드 제외).

실제로는 `ConcurrentHashMap`이 더 나은 대안이다. 버킷 단위 락(Java 8 이후 실제로는 CAS + 버킷별 synchronized)으로 읽기뿐 아니라 쓰기도 상당히 병렬화한다. `ReadWriteLock` + HashMap은 쓰기 중 완전 블로킹되지만 `ConcurrentHashMap`은 다른 버킷 쓰기와 충돌 없이 진행된다.

읽기 집약적이고 쓰기가 드문 경우: `ReadWriteLock` + HashMap > `ConcurrentHashMap` ≈ `StampedLock` + HashMap 순으로 고려하되, 반드시 JMH로 실측한다.

</details>

---

<div align="center">

**[⬅️ 이전: ReentrantLock AQS](./03-reentrantlock-aqs.md)** | **[홈으로 🏠](../README.md)** | **[다음: StampedLock ➡️](./05-stamped-lock.md)**

</div>

# synchronized 내부 — Biased→Thin→Fat Lock 확장

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Biased Lock에서 Thin Lock으로 전환되는 정확한 조건은 무엇인가?
- Thin Lock에서 Fat Lock으로 확장되는 과정에서 무슨 일이 일어나는가?
- ObjectMonitor 구조체는 어떻게 대기 스레드를 관리하는가?
- JIT의 Lock Elision과 Lock Coarsening은 어떻게 동작하는가?
- `-XX:+PrintBiasedLockingStatistics`로 무엇을 알 수 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`synchronized` 블록이 있는 코드의 성능 문제를 분석할 때, 락이 어느 단계에 있는지 알면 해결 방향이 명확해진다. "Fat Lock으로 전환됐다"는 것은 경쟁이 심각하다는 신호이고, JIT가 Lock Elision을 적용했다는 것은 특정 코드 경로에서 락이 완전히 사라진다는 의미다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수: 경쟁 없는 synchronized는 느리다고 포기
  "synchronized는 어차피 OS Mutex니까 무조건 피한다"
  → 경쟁 없는 경우: Thin Lock + JIT Lock Elision → 거의 비용 없음
  → 불필요하게 복잡한 Lock-Free 코드를 작성해 버그 유발

또 다른 실수: synchronized 범위를 항상 최소화
  public void process(List<String> items) {
      for (String item : items) {
          synchronized (this) {    // 루프마다 락 획득/해제
              processItem(item);
          }
      }
  }
  → JIT Lock Coarsening: 루프 밖으로 락 확장 가능
  → 개발자가 수동으로 줄이는 것이 오히려 JIT 최적화를 방해할 수 있음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```
경쟁 상황에 맞는 설계:

경쟁 없음 → synchronized 사용 OK
  JIT Lock Elision이 제거 가능
  Thin Lock CAS로 처리 → 빠름

약한 경쟁 → synchronized 또는 ReentrantLock
  Thin Lock 스핀으로 처리
  적응적 스핀이 자동 최적화

강한 경쟁 → 구조적 해결
  락 분리: ConcurrentHashMap 방식 (버킷별 락)
  Lock-Free: AtomicXxx, LongAdder
  락 범위 축소: 임계 구역 내 연산 최소화
  읽기/쓰기 분리: ReadWriteLock, StampedLock

JIT 최적화를 활용:
  synchronized 범위를 작게 유지 → Lock Elision 대상이 됨
  루프 내 synchronized → Lock Coarsening 가능
  → 직접 최적화하기 전에 JMH로 실제 성능 측정
```

---

## 🔬 내부 동작 원리

### 1. 락 확장 전체 흐름

```
synchronized (obj) { ... } 실행 시 JVM 내부 경로:

  ┌──────────────────────────────────────────────────────────────┐
  │  monitorenter 바이트코드 실행                                    │
  └───────────────────────────────────┬──────────────────────────┘
                                      │
                Mark Word 하위 3비트 확인
                ┌──────────────────┬────────────────────┐
                │ 101 (Biased)     │ 01 (Normal/Bias가능)│ 00/10 (Thin/Fat)
                ▼                  ▼                    ▼
         [바이어스 락 처리]        [신규 진입]            [상태별 처리]
              │                   │
              │ thread_id = 나?    │ CAS로 Thin Lock 시도
              ├─YES→ 즉시 진입       │
              └─NO→ Revocation    │ 성공 → Thin Lock 보유
                   Thin Lock으로   │ 실패 (경쟁) → 스핀
                   전환            │ 스핀 한계 → Fat Lock

Fat Lock 확장 과정:
  ① ObjectMonitor 생성 (힙 외부, Native 메모리)
  ② Mark Word를 Monitor 포인터로 교체 (하위 비트 10)
  ③ 경쟁 스레드들을 ObjectMonitor.EntryList에 추가
  ④ 락 소유 스레드가 해제 시 EntryList에서 하나 깨움
```

### 2. ObjectMonitor 구조체

```
Fat Lock의 핵심: ObjectMonitor

struct ObjectMonitor {
  volatile markOop _header;       // 원본 Mark Word 저장
  void* volatile _object;         // 락 객체 포인터
  
  volatile jint _waiters;         // wait() 중인 스레드 수
  volatile intptr_t _count;       // 진입 횟수 (재진입 카운터)
  volatile jint _recursions;      // 재진입 깊이
  
  ObjectWaiter* volatile _EntryList;    // 락 대기 스레드 목록
  ObjectWaiter* volatile _WaitSet;      // wait() 중인 스레드 목록
  
  volatile Thread* _owner;        // 현재 락 소유 스레드
}

락 획득/해제 흐름:
  스레드 A가 synchronized 진입:
    _owner == null → CAS(_owner, null, Thread_A) → 성공 → 진입
    _owner == Thread_A → _recursions++ → 재진입
    _owner == Thread_B → ObjectWaiter 생성 → _EntryList 삽입 → park()

  스레드 A가 synchronized 탈출:
    _recursions > 0 → _recursions-- → 계속 보유
    _recursions == 0 → _owner = null
                     → _EntryList에서 하나 선택 → unpark()

  Object.wait() 호출:
    _WaitSet으로 이동 + 락 해제
    notify()/notifyAll() 시 _WaitSet → _EntryList로 이동
```

### 3. JIT Lock Elision

```
Lock Elision (락 제거):
  JIT가 탈출 분석(Escape Analysis)으로 객체가 스택을 벗어나지 않으면
  해당 객체의 synchronized는 완전히 제거

예시:
  public int calculate(int a, int b) {
      StringBuffer sb = new StringBuffer();  // 로컬 생성
      synchronized (sb) {    // 이 sb는 이 메서드만 사용
          sb.append(a);      // 다른 스레드가 접근 불가
          sb.append(b);
      }
      return Integer.parseInt(sb.toString());
  }

  JIT 분석: sb가 이 메서드 스택 내에만 존재 (escape 안 함)
  → synchronized(sb) 완전 제거
  → 어셈블리에 monitorenter/monitorexit 없음

  -XX:+DoEscapeAnalysis (기본값 on)
  -XX:+EliminateLocks (기본값 on, Lock Elision 활성화)
  비활성화: -XX:-EliminateLocks

Lock Coarsening (락 병합):
  연속된 synchronized 블록을 하나로 합침

  // JIT 이전:
  for (int i = 0; i < 100; i++) {
      synchronized (obj) { list.add(i); }
  }

  // JIT 후 (Lock Coarsening):
  synchronized (obj) {
      for (int i = 0; i < 100; i++) { list.add(i); }
  }
  → 락 획득/해제 횟수 100회 → 1회로 감소
```

---

## 💻 실전 실험

### 실험 1: Lock 확장 단계별 성능 측정

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Group)
@Fork(2)
public class LockEscalationBenchmark {
    private final Object lock = new Object();
    private int value = 0;

    @Benchmark @Group("thin_1thread") @GroupThreads(1)
    public int thin_lock() { synchronized (lock) { return ++value; } }

    @Benchmark @Group("fat_2thread") @GroupThreads(2)
    public int fat_lock_2t() { synchronized (lock) { return ++value; } }

    @Benchmark @Group("fat_8thread") @GroupThreads(8)
    public int fat_lock_8t() { synchronized (lock) { return ++value; } }
}
```

### 실험 2: Lock Elision 확인

```bash
cat > LockElisionTest.java << 'EOF'
public class LockElisionTest {
    public static int noElision(int a, int b) {
        Object lock = new Object();
        int result = 0;
        synchronized (lock) {  // lock이 escape 안 함 → Elision 대상
            result = a + b;
        }
        return result;
    }

    public static void main(String[] args) {
        long count = 0;
        for (int i = 0; i < 100_000_000; i++) {
            count += noElision(i, i + 1);
        }
        System.out.println(count);
    }
}
EOF

# Lock Elision 활성화 (기본)
java -XX:+DoEscapeAnalysis -XX:+EliminateLocks LockElisionTest

# Lock Elision 비활성화 (비교)
java -XX:-EliminateLocks LockElisionTest

# PrintAssembly로 monitorenter 사라짐 확인
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly \
     -XX:CompileOnly=LockElisionTest.noElision \
     LockElisionTest 2>&1 | grep -E "(monitor|lock)"
```

### 실험 3: 경쟁 상황에서 Fat Lock 전환 확인

```bash
cat > FatLockDemo.java << 'EOF'
import java.lang.management.*;
import java.util.*;

public class FatLockDemo {
    static final Object LOCK = new Object();
    static long counter = 0;

    public static void main(String[] args) throws InterruptedException {
        ThreadMXBean bean = ManagementFactory.getThreadMXBean();

        int threadCount = 20;
        Thread[] threads = new Thread[threadCount];

        for (int i = 0; i < threadCount; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1_000_000; j++) {
                    synchronized (LOCK) { counter++; }
                }
            });
        }

        long startCS = bean.getCurrentThreadCpuTime();
        for (Thread t : threads) t.start();
        for (Thread t : threads) t.join();

        System.out.println("counter = " + counter);
        System.out.printf("총 컨텍스트 스위칭: %d%n",
            Arrays.stream(bean.getThreadInfo(bean.getAllThreadIds()))
                  .filter(Objects::nonNull)
                  .mapToLong(ThreadInfo::getBlockedCount)
                  .sum());
        // BLOCKED count가 높으면 Fat Lock 경쟁 심각
    }
}
EOF
java -XX:+UnlockDiagnosticVMOptions FatLockDemo
# jstack으로 BLOCKED 스레드 수 확인
```

---

## 📊 성능/비용 비교

```
락 단계별 비용 (x86, JDK 21 기준):

단계          | 조건          | 주요 연산            | 비용
─────────────┼──────────────┼────────────────────┼─────────────────
Thin Lock 획득| 무경쟁         | CAS 1회            | ~10~30 ns
Thin Lock 해제| 무경쟁         | CAS 1회            | ~10~30 ns
Thin Lock 경합| 약한 경쟁      | CAS 실패 + 스핀      | ~100~500 ns
Fat Lock 획득 | 경합 없음(첫 번)| OS Mutex + wake    | ~1~3 μs
Fat Lock 대기 | 경쟁 중        | 컨텍스트 스위칭       | ~5~50 μs
Fat Lock 해제 | 대기자 있음     | unpark + 스케줄링    | ~3~10 μs

Lock Elision 효과:
  synchronized (localObj) { ... }
  JIT Lock Elision 후: 락 연산 0회 → 일반 메서드 호출과 동일

Lock Coarsening 효과:
  N회 synchronized → 1회 synchronized
  락 오버헤드: N × (획득+해제) → 1 × (획득+해제)
  루프가 크면 클수록 효과 큼
```

---

## ⚖️ 트레이드오프

```
synchronized 사용 전략:

Lock Elision을 유도하는 패턴:
  로컬 객체에만 synchronized → JIT가 제거
  단, 실제 공유 필요 없는 경우만

Lock Coarsening을 유도하는 패턴:
  루프 내 연속 synchronized → JIT가 합침
  단, 합쳐진 임계 구역이 너무 길면 오히려 경쟁 악화

Fat Lock 방지 전략:
  임계 구역 최소화 → 대기 시간 감소
  Lock 분리 → 독립적인 데이터 별도 락
  Lock-Free 알고리즘 → 경쟁 자체를 없앰
  ReadWriteLock → 읽기 병렬화

JIT에 맡기는 것이 나은 경우:
  짧은 임계 구역의 경쟁 없는 synchronized
  → JIT가 Lock Elision 또는 Thin Lock 최적화 적용
  → 개발자가 직접 최적화하면 오히려 JIT 방해
```

---

## 📌 핵심 정리

```
synchronized 락 확장 핵심:

전환 순서 (Java 21):
  Normal → Thin Lock (CAS) → Fat Lock (OS Mutex)
  (Java 15 이전: Normal → Biased Lock → Thin Lock → Fat Lock)

각 단계 비용:
  Thin Lock: CAS 1회 (~30ns)
  Fat Lock: OS Mutex (~수 μs + 컨텍스트 스위칭)

JIT 최적화:
  Lock Elision: 스택 로컬 객체의 synchronized 완전 제거
  Lock Coarsening: 연속 synchronized 블록 합침

ObjectMonitor (Fat Lock):
  _EntryList: 락 대기 스레드
  _WaitSet: wait() 대기 스레드
  _owner: 현재 소유 스레드
  _recursions: 재진입 카운터

성능 문제 진단:
  jstack BLOCKED 스레드 수 → Fat Lock 경쟁 지표
  -XX:+PrintBiasedLockingStatistics (Java 15↓)
  JFR jdk.MonitorEnter 이벤트
```

---

## 🤔 생각해볼 문제

**Q1.** `synchronized` 블록 내부에서 예외가 발생하면 락이 해제되는가?

<details>
<summary>해설 보기</summary>

그렇다. `synchronized` 블록은 예외가 발생해도 반드시 `monitorexit`가 실행된다. 이는 바이트코드 수준에서 보장된다.

바이트코드 구조:
```
monitorenter
try {
    // 블록 내용
} catch (Throwable e) {
    monitorexit  // 예외 시에도 실행
    throw e;
}
monitorexit   // 정상 종료 시
```

따라서 `synchronized` 블록 내에서 예외가 발생해 전파되더라도 락은 항상 해제된다. 이것이 `ReentrantLock`에서 `try-finally`를 써야 하는 것과 달리 `synchronized`에서 명시적 finally가 필요 없는 이유다.

</details>

---

**Q2.** 같은 객체에 대해 `synchronized` 블록과 `ReentrantLock`을 혼용하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**완전히 다른 락 메커니즘이므로 상호 배제가 되지 않는다.** `synchronized`는 객체의 내장 모니터 락(Object Header Mark Word)을 사용하고, `ReentrantLock`은 AQS의 독립적인 내부 상태를 사용한다.

```java
Object lock = new Object();
ReentrantLock reentrantLock = new ReentrantLock();

// Thread A
synchronized (lock) { /* ... */ }

// Thread B
reentrantLock.lock();  // Thread A의 synchronized와 무관!
try { /* ... */ } finally { reentrantLock.unlock(); }
```

두 스레드가 동시에 임계 구역에 진입 가능 → race condition 발생. 하나의 공유 자원에는 일관된 동기화 메커니즘을 사용해야 한다.

</details>

---

**Q3.** JIT의 Lock Elision은 컴파일 시점에 결정되는가, 런타임에 결정되는가?

<details>
<summary>해설 보기</summary>

**런타임에 JIT 컴파일 시 결정**된다. HotSpot JVM의 JIT(C1, C2 컴파일러)는 메서드가 충분히 많이 호출된 후(Tiered Compilation 기준) 바이트코드를 기계어로 컴파일한다.

이 시점에 탈출 분석(Escape Analysis)을 수행하여:
1. 객체가 메서드 스택을 벗어나지 않는지 확인
2. 벗어나지 않으면 해당 객체의 synchronized를 제거

따라서 초기에는 인터프리터로 실행되다가, 충분한 호출 횟수 후 JIT 컴파일로 Lock Elision이 적용된다. 벤치마크에서 충분한 워밍업이 필요한 이유다.

탈출 분석이 정확하지 않은 경우(예: 메서드 인라이닝 실패, 복잡한 호출 그래프) Lock Elision이 적용되지 않을 수 있다.

</details>

---

<div align="center">

**[⬅️ 이전: Object Header Mark Word](./01-object-header-mark-word.md)** | **[홈으로 🏠](../README.md)** | **[다음: ReentrantLock AQS ➡️](./03-reentrantlock-aqs.md)**

</div>

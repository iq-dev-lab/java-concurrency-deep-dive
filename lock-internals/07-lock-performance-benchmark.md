# 락 성능 비교 — JMH 벤치마크와 JIT 최적화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `synchronized`, `ReentrantLock`, `StampedLock`, `AtomicLong`의 처리량은 경쟁 수준에 따라 어떻게 달라지는가?
- 스레드 수 증가에 따른 성능 역전이 일어나는 조건은?
- JIT의 Lock Elision이 적용되는 코드 패턴과 확인 방법은?
- Lock Coarsening이 자동으로 적용되는 조건은?
- JMH 벤치마크에서 락 성능을 측정할 때 주의해야 할 점은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"synchronized는 느리다", "AtomicLong이 ReentrantLock보다 빠르다" 같은 일반화는 모두 특정 조건에서만 맞는 말이다. 실제 성능은 경쟁 수준, 임계 구역 길이, 읽기/쓰기 비율, JIT 최적화 가능 여부에 따라 극적으로 달라진다. JMH로 직접 측정하고, Lock Elision/Coarsening을 이해해야 옳은 설계 결정을 내릴 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 일반화된 성능 주장을 맹신
  "AtomicLong이 synchronized보다 무조건 빠르다"
  → 단일 스레드: synchronized가 Lock Elision으로 더 빠를 수 있음
  → 고경합: LongAdder가 AtomicLong보다 훨씬 빠름

실수 2: 워밍업 없는 마이크로벤치마크
  long start = System.currentTimeMillis();
  for (int i = 0; i < 1000; i++) { synchronized(lock) { count++; } }
  System.out.println(System.currentTimeMillis() - start);
  → JIT 컴파일 전 인터프리터 실행 결과 → 실제 성능 아님
  → JMH 없이는 신뢰할 수 없음

실수 3: 임계 구역 크기를 고려하지 않음
  짧은 임계 구역: AtomicLong CAS가 synchronized보다 빠름
  긴 임계 구역: synchronized가 Lock Elision/Coarsening으로 더 유리할 수 있음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// 성능 측정의 올바른 접근
// 1. JMH 사용 (warmup + 반복 측정)
// 2. 실제 워크로드와 유사한 스레드 수, 경합 수준 설정
// 3. Lock Elision이 적용되지 않도록 공유 객체 사용
// 4. 여러 측정 모드 활용 (Throughput, Latency, Sample)

// 워크로드 분류별 락 선택 기준:
//   단일 카운터 (고경합) → LongAdder
//   단일 카운터 (저경합) → AtomicLong
//   복합 상태 변경 → synchronized 또는 ReentrantLock
//   읽기 집약 → StampedLock (낙관적) 또는 ReadWriteLock
//   타임아웃/인터럽트 필요 → ReentrantLock
```

---

## 🔬 내부 동작 원리

### 1. 락 방법별 내부 메커니즘 요약

```
락 방법별 핵심 동작 비교:

synchronized:
  무경합: Thin Lock CAS 1회 (~10~30ns)
  경합:   Fat Lock (OS Mutex, park/unpark)
  JIT:    Lock Elision (로컬 객체), Lock Coarsening
  
ReentrantLock:
  무경합: AQS state CAS 1회 (~15~35ns)
  경합:   AQS park/unpark (OS futex)
  JIT:    Elision 적용 덜 됨 (JVM이 AQS를 덜 분석)
  
AtomicLong.incrementAndGet():
  무경합: LOCK CMPXCHG 1회 (~20~40ns)
  경합:   CAS 실패 → 스핀 재시도 (OS 개입 없음)
  한계:   고경합 시 CAS 실패 폭증 → 처리량 급락
  
LongAdder.increment():
  무경합: base CAS 1회
  경합:   각 스레드가 별도 Cell에 누적 → 경합 없음
  sum():  모든 Cell 합산 (정확하지만 snapshot 아님)
  
StampedLock 낙관적 읽기:
  무경합: state 읽기 + validate → 락 없음 (~5ns)
  경합:   validate 실패 → 비관적 읽기로 전환
```

### 2. 스레드 수에 따른 성능 역전 조건

```
성능 역전이 일어나는 지점:

시나리오: 카운터 증가 (count++)

스레드 수 1:
  synchronized: Lock Elision 가능 → 0 비용
  AtomicLong:   LOCK CMPXCHG 1회
  LongAdder:    base CAS 1회
  결과: synchronized ≈ LongAdder >> AtomicLong (Elision 적용 시)

스레드 수 4~8 (코어 수 근처):
  synchronized: Thin Lock CAS + 간헐적 Fat Lock
  AtomicLong:   CAS 실패 발생, 재시도
  LongAdder:    Cell 분산으로 경합 없음
  결과: LongAdder >> AtomicLong ≈ synchronized

스레드 수 32~100 (코어 수 대비 4~12배):
  synchronized: Fat Lock 고착 → OS Mutex 병목
  AtomicLong:   CAS 실패율 90%+ → 스핀 낭비
  LongAdder:    Cell별 독립 → 선형 확장
  결과: LongAdder >>>> AtomicLong >> synchronized

성능 역전이 일어나는 이유:
  synchronized: Fat Lock 전환으로 OS Mutex 비용 + 컨텍스트 스위칭
  AtomicLong:   CAS 실패 시 지수적 처리량 저하 (CAS contention)
  LongAdder:    Cell 분산으로 경합 자체를 없앰 (Write 분산, Read는 합산)
```

### 3. JIT Lock Elision 적용 조건과 확인

```
Lock Elision이 적용되는 조건:

① 탈출 분석(Escape Analysis): 락 객체가 스택을 벗어나지 않음
② 메서드 인라이닝 성공: JIT가 코드를 충분히 분석 가능
③ 충분한 실행 횟수: JIT C2 컴파일 임계값 초과 (약 10,000회)

적용 예시:
  public int calculate() {
      Object lock = new Object();  // 매번 새 객체 생성 → 로컬
      synchronized (lock) {
          return a + b;
      }
  }
  JIT: lock이 calculate() 밖으로 나가지 않음 → synchronized 제거

  // StringBuffer는 내부적으로 synchronized
  public String build(int a, int b) {
      StringBuffer sb = new StringBuffer();  // 로컬 객체
      sb.append(a);   // synchronized(sb) → Elision
      sb.append(b);   // synchronized(sb) → Elision
      return sb.toString();
  }
  JIT: sb가 로컬 → 모든 synchronized 제거 → StringBuilder와 동일한 성능

적용 안 되는 예시:
  synchronized (sharedLock) { ... }  // 필드 → 탈출함 → Elision 불가
  synchronized (this) { ... }        // this가 다른 스레드에 보임 → 불가

확인 방법:
  -XX:+PrintEliminateLocks (불행히 Java 11+에서 제거됨)
  -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation
  -XX:+PrintAssembly로 monitorenter 없는지 확인
```

### 4. JIT Lock Coarsening 동작 조건

```
Lock Coarsening이 적용되는 조건:

① 동일한 락 객체에 대한 연속적인 synchronized 블록
② 블록 사이에 제어 흐름 변경 없음
③ 블록이 합쳐져도 임계 구역이 너무 커지지 않음

적용 예시:
  // 원래 코드:
  for (int i = 0; i < 100; i++) {
      synchronized (sb) { sb.append(i); }
  }
  // 코드: 100회 락 획득/해제

  // JIT 후 (Lock Coarsening):
  synchronized (sb) {
      for (int i = 0; i < 100; i++) { sb.append(i); }
  }
  // 결과: 1회 락 획득/해제

연속 블록에서의 적용:
  synchronized (lock) { a(); }
  synchronized (lock) { b(); }
  // JIT가 두 블록을 합침:
  synchronized (lock) { a(); b(); }

적용 제한:
  루프 안에서 무한히 커질 수 있는 경우 적용 안 됨
  블록 사이에 예외 발생 가능 지점이 있으면 제한됨
  -XX:+DoEscapeAnalysis와 연계 (기본 on)

개발자의 역할:
  Lock Coarsening을 믿고 작은 synchronized 블록 여러 개로 나눠도 됨
  JIT가 자동으로 합쳐줄 수 있음
  단, 합쳐진 임계 구역이 경쟁을 늘릴 수 있으니 실측 권장
```

---

## 💻 실전 실험 — 종합 JMH 벤치마크

### 실험 1: 경쟁 수준별 종합 성능 측정

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.*;
import java.util.concurrent.locks.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Warmup(iterations = 5, time = 2)
@Measurement(iterations = 5, time = 2)
public class LockComprehensiveBenchmark {

    private final Object syncLock    = new Object();
    private final ReentrantLock rl   = new ReentrantLock();
    private final StampedLock sl     = new StampedLock();
    private final AtomicLong atomic  = new AtomicLong();
    private final LongAdder adder    = new LongAdder();
    private long syncCount = 0, rlCount = 0, slCount = 0;

    // ── synchronized ──
    @Benchmark @Threads(1)  public void sync_1t()  { synchronized(syncLock){ syncCount++; }}
    @Benchmark @Threads(4)  public void sync_4t()  { synchronized(syncLock){ syncCount++; }}
    @Benchmark @Threads(8)  public void sync_8t()  { synchronized(syncLock){ syncCount++; }}
    @Benchmark @Threads(32) public void sync_32t() { synchronized(syncLock){ syncCount++; }}

    // ── ReentrantLock ──
    @Benchmark @Threads(1)  public void rl_1t()  { rl.lock(); try{rlCount++;}finally{rl.unlock();}}
    @Benchmark @Threads(4)  public void rl_4t()  { rl.lock(); try{rlCount++;}finally{rl.unlock();}}
    @Benchmark @Threads(8)  public void rl_8t()  { rl.lock(); try{rlCount++;}finally{rl.unlock();}}
    @Benchmark @Threads(32) public void rl_32t() { rl.lock(); try{rlCount++;}finally{rl.unlock();}}

    // ── AtomicLong ──
    @Benchmark @Threads(1)  public long atomic_1t()  { return atomic.incrementAndGet(); }
    @Benchmark @Threads(4)  public long atomic_4t()  { return atomic.incrementAndGet(); }
    @Benchmark @Threads(8)  public long atomic_8t()  { return atomic.incrementAndGet(); }
    @Benchmark @Threads(32) public long atomic_32t() { return atomic.incrementAndGet(); }

    // ── LongAdder ──
    @Benchmark @Threads(1)  public void adder_1t()  { adder.increment(); }
    @Benchmark @Threads(4)  public void adder_4t()  { adder.increment(); }
    @Benchmark @Threads(8)  public void adder_8t()  { adder.increment(); }
    @Benchmark @Threads(32) public void adder_32t() { adder.increment(); }

    // ── StampedLock 낙관적 읽기 ──
    @Benchmark @Threads(8) @Group("stamped_read")
    public long stamped_opt_read() {
        long stamp = sl.tryOptimisticRead();
        long v = slCount;
        if (!sl.validate(stamp)) {
            stamp = sl.readLock();
            try { v = slCount; } finally { sl.unlockRead(stamp); }
        }
        return v;
    }

    @Benchmark @Threads(1) @Group("stamped_read")
    public void stamped_write() {
        long stamp = sl.writeLock();
        try { slCount++; } finally { sl.unlockWrite(stamp); }
    }
}
```

### 실험 2: Lock Elision 효과 측정

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
public class LockElisionBenchmark {

    private int a = 1, b = 2;
    private final Object sharedLock = new Object();

    // Elision 대상: 로컬 락 객체
    @Benchmark
    public int localLock() {
        Object lock = new Object();  // 로컬 → Elision 가능
        synchronized (lock) {
            return a + b;
        }
    }

    // Elision 불가: 공유 락 객체
    @Benchmark
    public int sharedLock() {
        synchronized (sharedLock) {  // 공유 → Elision 불가
            return a + b;
        }
    }

    // 비교: 락 없는 버전
    @Benchmark
    public int noLock() {
        return a + b;
    }
}
// 결과 (단일 스레드):
// localLock ≈ noLock (Elision으로 락 제거)
// sharedLock < noLock (CAS 비용)
```

### 실험 3: Lock Coarsening 효과 측정

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
public class LockCoarseningBenchmark {

    private final Object lock = new Object();
    private int count = 0;

    // 많은 작은 synchronized 블록 (JIT가 합칠 수 있음)
    @Benchmark
    public int manySmallLocks() {
        for (int i = 0; i < 100; i++) {
            synchronized (lock) { count++; }
        }
        return count;
    }

    // 하나의 큰 synchronized 블록
    @Benchmark
    public int oneLargeLock() {
        synchronized (lock) {
            for (int i = 0; i < 100; i++) { count++; }
        }
        return count;
    }
}
// 결과: manySmallLocks ≈ oneLargeLock (JIT Coarsening이 합쳐줌)
// 단, 실제 경합 있을 때는 oneLargeLock이 더 불리할 수 있음
```

---

## 📊 성능 비교 종합표

```
경쟁 수준별 성능 순위 (처리량 기준, 참고치):

단일 스레드 (무경쟁):
  1위: synchronized (Lock Elision) ≈ noLock
  2위: LongAdder
  3위: AtomicLong
  4위: ReentrantLock (AQS 오버헤드)
  5위: synchronized (공유 객체, Thin Lock)

4~8 스레드 (낮은 경쟁):
  1위: LongAdder (Cell 분산)
  2위: AtomicLong (CAS 성공률 높음)
  3위: StampedLock 낙관적 읽기 (읽기 위주)
  4위: synchronized Thin Lock
  5위: ReentrantLock

32~100 스레드 (높은 경쟁):
  1위: LongAdder (경쟁 없음, Cell 독립)
  2위: StampedLock 낙관적 읽기 (쓰기 없을 때)
  3위: AtomicLong (CAS 실패 많음)
  4위: ReentrantLock (AQS park/unpark)
  5위: synchronized (Fat Lock, OS Mutex)

※ 위 순위는 단순 카운터 증가 기준
   임계 구역 길이, 읽기/쓰기 비율, JIT 상황에 따라 크게 달라짐
   항상 실제 워크로드로 JMH 측정 필수
```

---

## ⚖️ 트레이드오프

```
락 선택 의사결정 트리:

단순 카운터, 통계 누적?
  └─ 고경합 예상   → LongAdder
  └─ 저경합        → AtomicLong

읽기 >> 쓰기?
  └─ 극한 읽기 성능 필요 → StampedLock 낙관적 읽기
  └─ 일반적 읽기 집약    → ReadWriteLock

복합 상태 변경?
  └─ 타임아웃/인터럽트/다중 Condition 필요 → ReentrantLock
  └─ 그 외                                 → synchronized

JIT 최적화 활용?
  └─ 로컬 객체 락 → Lock Elision (synchronized 권장)
  └─ 연속 synchronized → Lock Coarsening (JIT에게 맡기기)

Virtual Thread 환경?
  └─ I/O 있는 임계 구역 → synchronized → ReentrantLock 전환 검토

"최적의 락은 없다. 워크로드에 맞는 락이 있을 뿐이다."
→ 추측하지 말고 JMH로 측정하라
```

---

## 📌 핵심 정리

```
락 성능 비교 핵심:

단일 카운터:
  무경합: synchronized (Elision) ≈ LongAdder > AtomicLong
  저경합: AtomicLong > synchronized
  고경합: LongAdder >>>> AtomicLong >> synchronized

읽기 집약:
  쓰기 없음: StampedLock 낙관적 읽기 > ReadWriteLock > synchronized
  쓰기 있음: ReadWriteLock > synchronized

JIT 최적화:
  Lock Elision: 로컬 객체 synchronized → 락 완전 제거
  Lock Coarsening: 연속 synchronized → 하나로 합침
  조건: 탈출 분석 성공 + 충분한 실행 횟수

JMH 측정 원칙:
  충분한 워밍업 (JIT 컴파일 완료 후 측정)
  실제 워크로드 스레드 수로 측정
  여러 번 반복 (분산 확인)
  Lock Elision 방지: 공유 상태 사용

판단 원칙:
  추측 금지, 측정 필수
  워크로드가 다르면 최적 락이 다름
  단순한 코드가 JIT 최적화를 더 잘 받음
```

---

## 🤔 생각해볼 문제

**Q1.** LongAdder.sum()은 정확한 값을 보장하지 않는다고 한다. 어떤 상황에서 이것이 문제가 되고, 어떤 상황에서는 괜찮은가?

<details>
<summary>해설 보기</summary>

`LongAdder.sum()`은 `base` + 모든 `Cell` 값의 합산이다. 다른 스레드가 동시에 `increment()`를 수행하는 도중에 `sum()`을 호출하면 최종 카운트가 아닌 스냅샷을 얻는다.

**문제가 되는 경우**: 정확한 값에 기반해 제어 흐름을 결정할 때. 예: "count >= limit이면 더 이상 처리하지 않는다". `sum()`이 limit보다 작게 나와서 초과 처리가 발생할 수 있다.

**괜찮은 경우**: 
- 통계 수집 (초당 요청 수, 평균 응답 시간 등) — 약간의 오차 허용
- 모니터링 대시보드 표시 — 실시간 정확성이 크게 중요하지 않음
- 처리량 계산 — 긴 시간 간격으로 측정 시 오차 무시

정확한 카운터가 필요하면 `AtomicLong`을 사용하거나, `LongAdder`를 쓰되 increment를 멈춘 후 `sum()`을 호출해야 한다.

</details>

---

**Q2.** JMH 벤치마크에서 `@State(Scope.Benchmark)`와 `@State(Scope.Thread)`의 차이는 락 성능 측정에 어떤 영향을 미치는가?

<details>
<summary>해설 보기</summary>

`Scope.Benchmark`: 모든 JMH 스레드가 하나의 State 인스턴스를 공유한다. 락 객체가 공유되므로 실제 경쟁이 발생한다. 락 성능 측정에 적합하다.

`Scope.Thread`: 각 스레드가 독립적인 State 인스턴스를 가진다. 락 객체가 스레드별로 다르므로 경쟁이 발생하지 않는다. Lock Elision이 적용될 가능성이 높아져 락 비용이 0에 가까워진다. 락 성능 측정에는 부적합하다.

락 성능을 올바르게 측정하려면 `Scope.Benchmark`를 사용하고, 락 객체가 공유됨을 확인해야 한다. 또한 임계 구역 내부에 실제 연산이 있어야 JIT가 전체 블록을 제거(Dead Code Elimination)하지 않는다.

```java
@State(Scope.Benchmark)  // 공유 → 실제 경쟁
public class LockBench {
    private final Object lock = new Object();
    private volatile int counter = 0;  // volatile: DCE 방지

    @Benchmark
    @Threads(8)
    public void increment() {
        synchronized (lock) { counter++; }
    }
}
```

</details>

---

**Q3.** 임계 구역 내부 작업 시간이 락 획득 비용보다 훨씬 길면(예: 10ms 데이터베이스 쿼리), 락 종류의 차이가 전체 성능에 미치는 영향은 얼마나 되는가?

<details>
<summary>해설 보기</summary>

임계 구역이 10ms이고 락 획득 비용이 최악 1μs라면, 락 종류의 차이가 전체 응답 시간에 미치는 영향은 0.01%(1μs / 10ms)에 불과하다.

이 경우:
- `synchronized`와 `ReentrantLock`의 차이: 무의미
- `AtomicLong`과 `LongAdder`의 차이: 무의미
- 실제 병목: 임계 구역 자체의 길이

최적화 방향:
1. 임계 구역 내 블로킹 I/O 제거 → 임계 구역 외부로 이동
2. 읽기/쓰기 분리 → 읽기를 임계 구역 밖으로 이동
3. 락 구조 재설계 → 세밀한 락(Fine-grained locking)으로 경쟁 면적 축소

락 최적화는 **임계 구역이 매우 짧을 때** 의미 있다. 긴 임계 구역에서는 락 종류보다 구조적 설계가 훨씬 중요하다.

</details>

---

<div align="center">

**[⬅️ 이전: 조건 변수](./06-condition-variables.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — CAS와 LOCK CMPXCHG ➡️](../lock-free-algorithms/01-cas-lock-cmpxchg.md)**

</div>

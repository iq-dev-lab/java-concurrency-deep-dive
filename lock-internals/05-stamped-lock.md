# StampedLock — 낙관적 읽기와 validateStamp

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 낙관적 읽기(Optimistic Read)가 락 없이 읽기를 시도하는 원리는?
- `validateStamp()`가 쓰기 발생 여부를 어떻게 감지하는가?
- 낙관적 읽기 실패 시 비관적 읽기로 전환하는 올바른 패턴은?
- `StampedLock`이 재진입 불가인 이유와 이로 인한 데드락 조건은?
- `ReentrantReadWriteLock`과 성능 비교에서 어느 상황에 역전이 일어나는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`ReentrantReadWriteLock`의 Write Starvation과 읽기 락 오버헤드를 모두 해결한 것이 `StampedLock`이다. 낙관적 읽기(Optimistic Read)는 락을 획득하지 않고 읽기를 시도하고, 읽기 중 쓰기가 없었는지를 `validateStamp()`로 사후 확인한다. 쓰기가 드문 환경에서 읽기 성능을 극적으로 향상시킬 수 있다. 단, 재진입 불가와 조건 변수 미지원이라는 중요한 제약이 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 낙관적 읽기 후 validate 없이 데이터 사용
  long stamp = sl.tryOptimisticRead();
  int x = point.x;     // 이 시점에 쓰기가 일어났을 수도!
  int y = point.y;
  // validate 없이 바로 사용 → 부분 갱신된 값으로 계산
  return Math.sqrt(x * x + y * y);  // x는 새 값, y는 이전 값일 수 있음

실수 2: StampedLock 재진입 시도
  long stamp = sl.readLock();
  try {
      sl.readLock();  // 데드락! StampedLock은 재진입 불가
  } finally { sl.unlockRead(stamp); }

실수 3: stamp를 잃어버리고 unlock 시도
  long stamp = sl.writeLock();
  stamp = 0;  // 실수로 덮어씀
  sl.unlock(stamp);  // 잘못된 stamp → IllegalMonitorStateException

실수 4: 낙관적 읽기 중 validate 없이 루프
  long stamp = sl.tryOptimisticRead();
  while (someCondition) {
      x = data;  // 루프마다 쓰기가 일어났는지 미확인
  }
  if (!sl.validate(stamp)) { /* 실패 처리 */ }
  // 루프 내 중간 값들은 이미 사용됨
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// StampedLock 올바른 낙관적 읽기 패턴
class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    // 낙관적 읽기: 거리 계산
    double distanceFromOrigin() {
        // 1단계: 낙관적 읽기 시도 (락 없음)
        long stamp = sl.tryOptimisticRead();
        double curX = x, curY = y;  // 데이터 로컬 복사

        // 2단계: 읽기 중 쓰기가 없었는지 검증
        if (!sl.validate(stamp)) {
            // 쓰기 발생 → 비관적 읽기로 전환
            stamp = sl.readLock();
            try {
                curX = x; curY = y;  // 안전하게 재읽기
            } finally {
                sl.unlockRead(stamp);
            }
        }

        return Math.sqrt(curX * curX + curY * curY);
    }

    // 쓰기: 위치 이동
    void move(double deltaX, double deltaY) {
        long stamp = sl.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);
        }
    }

    // 조건부 업그레이드 패턴 (읽기 → 쓰기)
    void moveIfOrigin(double newX, double newY) {
        long stamp = sl.readLock();
        try {
            while (x == 0.0 && y == 0.0) {
                // 쓰기 락으로 업그레이드 시도
                long ws = sl.tryConvertToWriteLock(stamp);
                if (ws != 0L) {
                    stamp = ws;
                    x = newX; y = newY;
                    return;
                } else {
                    // 업그레이드 실패 → 읽기 락 해제 후 쓰기 락 획득
                    sl.unlockRead(stamp);
                    stamp = sl.writeLock();
                }
            }
        } finally {
            sl.unlock(stamp);
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. StampedLock의 상태와 stamp 구조

```
StampedLock 내부 상태 (long state, 64비트):

  ┌──────────────────────────────────────────────────────────────┐
  │  64비트 state                                                 │
  │                                                              │
  │  bit 7 (WBIT):  쓰기 락 비트 (0=없음, 1=쓰기 중)                   │
  │  bit 0~6:       읽기 락 카운트 (0~126, 126 이상은 overflow)       │
  │  bit 8~63:      버전(version) 카운터                            │
  │                                                              │
  │  쓰기 락 획득 시: version 카운터 증가 + WBIT 설정                   │
  │  쓰기 락 해제 시: WBIT 제거 + version 카운터 다시 증가               │
  └──────────────────────────────────────────────────────────────┘

stamp 값의 의미:
  낙관적 읽기 stamp: state & ~(읽기카운트|WBIT) = 현재 버전의 스냅샷
  읽기 락 stamp:    state (읽기 카운트 포함)
  쓰기 락 stamp:    state (WBIT 포함)

validate(stamp) 동작:
  현재 state와 stamp를 비교
  stamp == (state & SBITS)  → 성공 (그 사이 쓰기 없음)
  stamp != (state & SBITS)  → 실패 (쓰기가 일어나 version 변경됨)

  내부 구현:
  public boolean validate(long stamp) {
      // LoadLoad 펜스: 읽기가 이 펜스 이전으로 재정렬 안 됨
      U.loadFence();
      return (stamp & SBITS) == (state & SBITS);
  }
  // loadFence()가 중요: 로컬 복사 후 validate 사이의 순서를 보장
```

### 2. 낙관적 읽기의 정확한 동작 과정

```
낙관적 읽기 전체 흐름:

  ① stamp = sl.tryOptimisticRead()
     → state & SBITS 반환 (쓰기 락 없을 때만 유효 stamp)
     → 쓰기 락 중이면 0 반환 (validate 즉시 실패)
  
  ② 데이터 로컬 변수로 복사
     double curX = x, curY = y;
     // 이 시점에 다른 스레드가 x, y를 수정할 수 있음
     // curX = 새 x, curY = 이전 y → 비일관 상태 가능
  
  ③ if (!sl.validate(stamp))
     → U.loadFence() 삽입 (재정렬 방지)
     → 현재 state와 stamp 버전 비교
     → 쓰기가 있었으면 stamp != current_version → false
  
  ④ validate 성공: curX, curY 사용
     validate 실패: 비관적 읽기로 전환

왜 validate가 필요한가:
  낙관적 읽기는 "쓰기가 없을 것이다"는 가정으로 락 없이 읽음
  쓰기가 없었으면 읽은 값이 일관성 있음
  쓰기가 있었으면 curX, curY가 서로 다른 시점의 값일 수 있음
  → validate로 쓰기 발생 여부를 확인
  
  loadFence의 역할:
  데이터 복사(curX = x) 이후 validate(stamp) 이전에
  LoadLoad 펜스 삽입
  → 데이터 읽기가 validate 이후로 재정렬되는 것을 방지
  → 없으면: CPU가 curX = x를 validate 이후에 실행할 수 있음
            → 검증이 무의미해짐
```

### 3. tryConvertToWriteLock — 읽기에서 쓰기로 원자적 전환

```
tryConvertToWriteLock(stamp): 읽기/낙관적 → 쓰기 원자적 전환 시도

경우 1: 낙관적 읽기 stamp + 다른 락 없음
  → 즉시 쓰기 락으로 전환 (원자적 CAS)
  → 성공 시 새 쓰기 stamp 반환

경우 2: 읽기 락 stamp + 다른 읽기 락 없음 (내가 유일한 읽기자)
  → 쓰기 락으로 전환
  → 성공 시 새 쓰기 stamp 반환

경우 3: 읽기 락 stamp + 다른 읽기 락 있음
  → 전환 불가, 0 반환
  → 호출자가 읽기 락 해제 후 쓰기 락 직접 획득 필요

ReentrantReadWriteLock의 업그레이드와의 차이:
  ReentrantReadWriteLock: 업그레이드 자체 불가 (데드락)
  StampedLock.tryConvertToWriteLock:
    원자적 전환 시도 → 불가하면 0 반환 (블로킹 없음)
    → 데드락 없이 업그레이드 가능성 탐색
```

### 4. 재진입 불가 이유와 데드락 조건

```
재진입 불가 (Non-Reentrant) 이유:
  StampedLock은 stamp 기반으로 락을 추적
  재진입 시 새 stamp 발급 → 두 stamp가 존재
  unlock(stamp1) → unlock(stamp2) → 상태 불일치 발생
  
  AQS처럼 스레드 소유권 추적 없음
  → 현재 스레드가 이미 락을 보유했는지 알 수 없음
  → 재진입 시 쓰기 락이면 자기 자신을 기다리는 상황 발생

데드락 발생 패턴:
  // 데드락 1: 읽기 재진입
  long stamp1 = sl.readLock();
  try {
      long stamp2 = sl.readLock();  // 가능 (읽기는 공유)
      // 하지만 쓰기 대기 후 재진입 시 문제
  } finally { sl.unlockRead(stamp1); }

  // 데드락 2: 쓰기 재진입
  long stamp1 = sl.writeLock();
  try {
      long stamp2 = sl.writeLock();  // 데드락! 자기 자신을 기다림
  } finally { sl.unlockWrite(stamp1); }

  // 데드락 3: 읽기 → 쓰기 업그레이드 직접 시도
  long stamp1 = sl.readLock();
  try {
      long stamp2 = sl.writeLock();  // 데드락! (다른 읽기 락 없어도)
  } finally { sl.unlockRead(stamp1); }

안전한 업그레이드: tryConvertToWriteLock 사용 (앞 절 참조)
```

---

## 💻 실전 실험

### 실험 1: StampedLock vs ReadWriteLock 처리량 비교

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Warmup(iterations = 3)
@Measurement(iterations = 5)
public class StampedVsRWLockBenchmark {

    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final StampedLock sl = new StampedLock();
    private double x = 1.0, y = 2.0;

    // ReadWriteLock 읽기
    @Benchmark @Threads(8) @Group("rwlock")
    public double rwlock_read() {
        rwLock.readLock().lock();
        try { return Math.sqrt(x * x + y * y); }
        finally { rwLock.readLock().unlock(); }
    }

    // StampedLock 낙관적 읽기
    @Benchmark @Threads(8) @Group("stamped_opt")
    public double stamped_optimistic() {
        long stamp = sl.tryOptimisticRead();
        double curX = x, curY = y;
        if (!sl.validate(stamp)) {
            stamp = sl.readLock();
            try { curX = x; curY = y; } finally { sl.unlockRead(stamp); }
        }
        return Math.sqrt(curX * curX + curY * curY);
    }

    // StampedLock 비관적 읽기
    @Benchmark @Threads(8) @Group("stamped_pessimistic")
    public double stamped_pessimistic() {
        long stamp = sl.readLock();
        try { return Math.sqrt(x * x + y * y); }
        finally { sl.unlockRead(stamp); }
    }
}
// 결과 (쓰기 없음, 읽기 집약):
// stamped_optimistic >> stamped_pessimistic > rwlock_read
// 낙관적 읽기: 락 없음 → L1 캐시 수준 성능
```

### 실험 2: 쓰기 빈도별 낙관적 읽기 실패율 측정

```java
import java.util.concurrent.atomic.LongAdder;
import java.util.concurrent.locks.StampedLock;

public class OptimisticReadFailureRate {
    static final StampedLock sl = new StampedLock();
    static double x = 0, y = 0;
    static final LongAdder total = new LongAdder();
    static final LongAdder failures = new LongAdder();

    public static void main(String[] args) throws InterruptedException {
        // 쓰기 스레드 (빈도 조절)
        Thread writer = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) {
                long stamp = sl.writeLock();
                try { x = i; y = i; }
                finally { sl.unlockWrite(stamp); }
                try { Thread.sleep(0, 100); } catch (InterruptedException e) {}
            }
        });

        // 읽기 스레드 10개
        Thread[] readers = new Thread[10];
        for (int i = 0; i < 10; i++) {
            readers[i] = new Thread(() -> {
                while (!Thread.currentThread().isInterrupted()) {
                    long stamp = sl.tryOptimisticRead();
                    double curX = x, curY = y;
                    total.increment();
                    if (!sl.validate(stamp)) {
                        failures.increment();
                        // 비관적 읽기로 전환
                        stamp = sl.readLock();
                        try { curX = x; curY = y; }
                        finally { sl.unlockRead(stamp); }
                    }
                    // curX, curY 사용
                }
            });
        }

        writer.start();
        for (Thread r : readers) r.start();

        Thread.sleep(5000);
        writer.join();
        for (Thread r : readers) r.interrupt();

        long t = total.sum(), f = failures.sum();
        System.out.printf("총 읽기: %,d, 실패: %,d (%.2f%%)%n",
            t, f, 100.0 * f / t);
    }
}
```

### 실험 3: 재진입 데드락 재현 (교육 목적)

```java
import java.util.concurrent.locks.StampedLock;
import java.util.concurrent.TimeUnit;

public class ReentrantDeadlockDemo {
    static final StampedLock sl = new StampedLock();

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            long stamp1 = sl.writeLock();
            System.out.println("쓰기 락 1 획득: " + stamp1);
            try {
                System.out.println("쓰기 락 2 시도 중... (데드락 발생)");
                // long stamp2 = sl.writeLock();  // 이 줄을 주석 해제하면 데드락
                // tryLock으로 안전하게 시도
                long stamp2 = sl.tryWriteLock(1, TimeUnit.SECONDS);
                System.out.println("쓰기 락 2 결과: " + stamp2);  // 0 (실패)
                // 실패 시 stamp2 = 0 → unlock 호출하면 안 됨
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                sl.unlockWrite(stamp1);
                System.out.println("쓰기 락 1 해제");
            }
        });
        t.start();
        t.join();
    }
}
```

---

## 📊 성능/비용 비교

```
락 방법별 읽기 성능 (8스레드, 쓰기 없음):

방법                          | 처리량 (ops/ms)  | 특징
─────────────────────────────┼────────────────┼──────────────────────────
StampedLock 낙관적 읽기         | 2,000,000+     | 락 없음, 검증만
StampedLock 비관적 읽기         | 400,000        | AQS 방식 읽기 락
ReentrantReadWriteLock 읽기   | 350,000        | ThreadLocal 오버헤드 있음
ReentrantLock                | 100,000        | 배타적 락
synchronized                 | 100,000        | 배타적 락

쓰기 빈도에 따른 낙관적 읽기 실패율:
쓰기 빈도   | 실패율     | 결과
──────────┼──────────┼──────────────────────────────
0%        | 0%       | 항상 낙관적 읽기 성공
1%        | ~1~5%    | 대부분 성공, 간혹 비관적 전환
10%       | ~10~30%  | 비관적 전환 빈번 → StampedLock 이점 감소
50%+      | ~50%+    | 낙관적 읽기 이점 거의 없음 → RWLock과 비슷

낙관적 읽기의 한계:
  쓰기 빈도 > 20% → StampedLock 낙관적 읽기 이점 급감
  이 경우 ReentrantReadWriteLock이 더 적합
```

---

## ⚖️ 트레이드오프

```
StampedLock 장단점:

장점:
  낙관적 읽기로 극한의 읽기 성능 (쓰기 없을 때 락 비용 0)
  Write Starvation 없음 (쓰기가 읽기를 밀어낼 수 있음)
  tryConvertToWriteLock으로 안전한 업그레이드

단점:
  재진입 불가 → 코드 구조 제약
  조건 변수(Condition) 없음
  API가 복잡하고 실수 유발 쉬움 (stamp 관리)
  쓰기가 빈번하면 이점 없음

선택 기준:
  쓰기 << 읽기 (5% 미만) → StampedLock 낙관적 읽기
  재진입 필요 → ReentrantReadWriteLock
  Condition 필요 → ReentrantLock + Condition
  단순한 읽기 집약 → ReentrantReadWriteLock (더 안전)

주의사항:
  낙관적 읽기 중 반드시 로컬 변수에 복사
  validate 전에 loadFence 보장 (validate() 내부에서 자동)
  validate 실패 후 재읽기 필수 (이전 로컬 값 무효)
  stamp를 잃어버리면 영구 락 → 항상 finally에서 unlock
```

---

## 📌 핵심 정리

```
StampedLock 핵심:

3가지 모드:
  낙관적 읽기: tryOptimisticRead() → validate() → 성공 시 사용
  비관적 읽기: readLock() / unlockRead(stamp)
  쓰기:       writeLock() / unlockWrite(stamp)

낙관적 읽기 패턴 (항상 이 구조):
  long stamp = sl.tryOptimisticRead();
  // 데이터 로컬 복사
  if (!sl.validate(stamp)) {
      stamp = sl.readLock();
      try { /* 재읽기 */ } finally { sl.unlockRead(stamp); }
  }
  // 로컬 복사본 사용

제약:
  재진입 불가 (데드락 주의)
  Condition 없음
  stamp 분실 → 영구 락

성능:
  낙관적 읽기: 쓰기 없을 때 RWLock 대비 5~10배 처리량
  쓰기 빈도 높아질수록 이점 감소

안전한 업그레이드:
  tryConvertToWriteLock(stamp) 사용
  실패 시 0 반환 → 직접 처리
```

---

## 🤔 생각해볼 문제

**Q1.** 낙관적 읽기에서 `validate()`가 실패했을 때, 이미 읽어온 로컬 변수를 왜 사용하면 안 되는가?

<details>
<summary>해설 보기</summary>

`validate()` 실패는 데이터를 읽는 동안 쓰기가 발생했음을 의미한다. 이때 로컬 변수들은 서로 다른 시점의 값을 담고 있을 수 있다.

예시: `Point(x=0, y=0)` → `move(1, 1)` 발생 → `Point(x=1, y=1)`
- `curX = x` 실행 (x=0 읽음)
- 이때 쓰기 발생: x=1, y=1로 변경
- `curY = y` 실행 (y=1 읽음)
- `validate()` 실패
- `curX=0, curY=1` → 이를 사용하면 `√(0²+1²) = 1.0` 반환
- 올바른 값은 `√(1²+1²) = √2 ≈ 1.414`

이렇게 부분 갱신된 값(torn read)을 사용하면 논리적 오류가 발생한다. `validate()` 실패 시 반드시 비관적 읽기로 재시도해야 한다.

</details>

---

**Q2.** `StampedLock`은 `synchronized`나 `ReentrantLock`처럼 Thread Dump에서 어떻게 보이는가?

<details>
<summary>해설 보기</summary>

`StampedLock`은 AQS 기반이지만 소유 스레드 정보를 명시적으로 추적하지 않는다. Thread Dump에서는:

```
"waiting-thread" #30 prio=5
   java.lang.Thread.State: WAITING (parking)
    at sun.misc.Unsafe.park(Native Method)
    at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
    at java.util.concurrent.locks.StampedLock.acquireRead(StampedLock.java:...)
```

`synchronized`처럼 "locked by Thread-X" 정보가 없어서 데드락 분석이 어렵다. `jstack`의 자동 데드락 감지도 동작하지 않는다.

이 때문에 운영 환경에서 `StampedLock` 데드락 진단이 매우 어렵다. `ReentrantLock`은 AQS가 소유 스레드 정보를 추적하므로 Thread Dump에서 더 많은 정보를 제공한다.

</details>

---

**Q3.** `StampedLock`의 낙관적 읽기는 Java 9의 `VarHandle.getOpaque()`나 `getAcquire()`와 어떻게 다른가?

<details>
<summary>해설 보기</summary>

둘 다 "락 없는 읽기"를 지원하지만 목적과 보장 수준이 다르다.

`VarHandle.getOpaque()`: 컴파일러 재정렬만 방지, CPU 재정렬은 허용. 같은 스레드에서의 관찰 순서 보장.

`VarHandle.getAcquire()`: LoadLoad + LoadStore 펜스 삽입. Acquire 이후 읽기/쓰기가 앞으로 이동 불가.

`StampedLock` 낙관적 읽기:
- `tryOptimisticRead()`는 버전 스탬프를 읽음
- 데이터 읽기는 일반 읽기 (펜스 없음)
- `validate()`에서 `loadFence()`를 삽입하여 데이터 읽기가 검증 이후로 재정렬되지 않음을 보장
- 버전 불일치 시 실패로 처리 → 비관적으로 재시도

`StampedLock`은 "일관성 확인" 메커니즘을 제공하고, `VarHandle`은 개별 접근의 메모리 순서를 제어한다. 완전히 다른 추상화 수준이다.

</details>

---

<div align="center">

**[⬅️ 이전: ReadWriteLock](./04-read-write-lock.md)** | **[홈으로 🏠](../README.md)** | **[다음: 조건 변수 ➡️](./06-condition-variables.md)**

</div>

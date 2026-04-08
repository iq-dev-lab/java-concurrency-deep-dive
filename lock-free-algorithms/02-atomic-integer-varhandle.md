# AtomicInteger/AtomicLong 내부 — Unsafe와 VarHandle

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Java 8 이하에서 `sun.misc.Unsafe`로 CAS를 구현하는 방법은?
- Java 9+에서 `VarHandle`로 전환한 이유는 무엇인가?
- `VarHandle.compareAndSet()`이 `Unsafe`와 동일한 성능을 내는 이유는?
- `AtomicInteger` 소스코드의 핵심 로직은 어떻게 구성되는가?
- `getAndIncrement()`, `updateAndGet()`, `accumulateAndGet()`의 내부 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`AtomicInteger`의 내부를 이해하면 CAS 루프 패턴을 직접 작성할 수 있고, `VarHandle`을 통해 일반 필드에 원자적 접근을 더 세밀하게 제어할 수 있다. Java 9 모듈 시스템에서 `Unsafe` 접근이 제한되면서 `VarHandle`이 표준 대안이 됐다. 프레임워크나 라이브러리 개발 시 이 이해가 필수다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: AtomicInteger를 synchronized의 단순 대체로만 인식
  "synchronized 대신 AtomicInteger 쓰면 더 빠르겠지"
  → 고경쟁 환경에서는 synchronized가 더 공정하고 오히려 효율적
  → 복합 연산(read-modify-write 여러 개)은 여전히 synchronized 필요

실수 2: AtomicInteger를 스레드 안전한 값 컨테이너로만 사용
  AtomicInteger sum = new AtomicInteger(0);
  int current = sum.get();         // 읽기
  current += computeValue();       // 로컬에서 계산
  sum.set(current);                // 다시 쓰기 ← race condition!
  // get → compute → set 사이에 다른 스레드가 끼어들 수 있음

  올바른 방법:
  sum.updateAndGet(v -> v + computeValue());  // 원자적 업데이트

실수 3: Java 9+ 환경에서 Unsafe를 직접 사용 시도
  Field f = Unsafe.class.getDeclaredField("theUnsafe");
  f.setAccessible(true);  // Java 17+ 모듈 경계에서 실패
  → VarHandle로 대체해야 함
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// AtomicInteger 올바른 활용 패턴

// 패턴 1: 단순 증감
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();   // thread-safe ++
counter.getAndIncrement();   // thread-safe i++ (이전 값 반환)
counter.addAndGet(5);        // thread-safe += 5

// 패턴 2: 함수형 업데이트 (Java 8+)
// updateAndGet: 현재 값을 받아 새 값을 반환하는 함수
int newMax = maxVal.updateAndGet(cur -> Math.max(cur, candidate));

// 패턴 3: accumulateAndGet (기존 값 + 인자를 합산)
AtomicLong total = new AtomicLong(0);
total.accumulateAndGet(amount, Long::sum);  // total += amount

// 패턴 4: VarHandle로 일반 필드에 원자적 접근 (Java 9+)
import java.lang.invoke.MethodHandles;
import java.lang.invoke.VarHandle;

class Node {
    volatile int value;
    static final VarHandle VALUE;

    static {
        try {
            VALUE = MethodHandles.lookup()
                .findVarHandle(Node.class, "value", int.class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    boolean cas(int expected, int update) {
        return VALUE.compareAndSet(this, expected, update);
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Java 8 이하: sun.misc.Unsafe 기반 구현

```java
// AtomicInteger 소스 (Java 8, 단순화)
public class AtomicInteger {
    private static final Unsafe U = Unsafe.getUnsafe();
    // Unsafe: JVM 내부 저수준 메모리 조작 API
    // sun.misc.Unsafe → 공개 API 아님, JVM 내부 사용

    private static final long VALUE;
    // VALUE: AtomicInteger 객체 내 'value' 필드의 메모리 오프셋

    static {
        try {
            VALUE = U.objectFieldOffset(
                AtomicInteger.class.getDeclaredField("value"));
            // 필드의 객체 내 바이트 오프셋 계산 (컴파일 타임 상수)
        } catch (ReflectiveOperationException e) {
            throw new Error(e);
        }
    }

    private volatile int value;
    // volatile: 가시성 보장 (읽기는 항상 최신 값)

    public final int incrementAndGet() {
        return U.getAndAddInt(this, VALUE, 1) + 1;
        // Unsafe.getAndAddInt: LOCK XADD 명령어 사용
        // (LOCK CMPXCHG 루프보다 효율적인 원자적 fetch-and-add)
    }

    public final boolean compareAndSet(int expectedValue, int newValue) {
        return U.compareAndSetInt(this, VALUE, expectedValue, newValue);
        // Unsafe.compareAndSetInt → LOCK CMPXCHG 명령어
    }
}

Unsafe.getAndAddInt 구현:
  public final int getAndAddInt(Object o, long offset, int delta) {
      int v;
      do {
          v = getIntVolatile(o, offset);   // volatile 읽기
      } while (!compareAndSwapInt(o, offset, v, v + delta));  // CAS
      return v;  // 이전 값 반환
  }
  // 실제 JIT 컴파일 후: LOCK XADD 단일 명령어로 최적화

sun.misc.Unsafe 문제점:
  비공개 API: 공식 지원 안 됨
  Java 9 모듈 시스템: --add-opens 없이는 리플렉션 접근 불가
  이식성: JVM 구현체마다 다를 수 있음
```

### 2. Java 9+: VarHandle으로 전환

```java
// AtomicInteger 소스 (Java 9+)
public class AtomicInteger {
    private static final VarHandle VALUE;
    // VarHandle: java.lang.invoke 패키지의 공개 API
    // 모듈 시스템과 호환, 성능은 Unsafe와 동일

    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            VALUE = l.findVarHandle(AtomicInteger.class, "value", int.class);
        } catch (ReflectiveOperationException e) {
            throw new Error(e);
        }
    }

    private volatile int value;

    public final int incrementAndGet() {
        return (int) VALUE.getAndAdd(this, 1) + 1;
        // VarHandle.getAndAdd → JIT가 LOCK XADD로 최적화
    }

    public final boolean compareAndSet(int expectedValue, int newValue) {
        return VALUE.compareAndSet(this, expectedValue, newValue);
        // VarHandle.compareAndSet → JIT가 LOCK CMPXCHG로 최적화
    }
}

VarHandle 전환의 이점:
  ① 모듈 경계 존중: 공개 API이므로 모듈 시스템과 충돌 없음
  ② 타입 안전성: 컴파일 타임 타입 체크 (Unsafe는 런타임만)
  ③ 성능: JIT가 Unsafe와 동일하게 최적화 (intrinsic 처리)
  ④ 메모리 오더링 제어: getAcquire/setRelease 등 세밀한 제어 가능

VarHandle intrinsic:
  JVM이 VarHandle.compareAndSet() 등을 특별 처리
  → JIT 컴파일 시 직접 LOCK CMPXCHG 기계어 생성
  → 함수 호출 오버헤드 없음 (Unsafe와 동일 성능)
```

### 3. AtomicInteger 핵심 메서드 분석

```java
// get() / set() / lazySet()
public final int get() {
    return value;  // volatile 읽기: LoadLoad + LoadStore 펜스
}

public final void set(int newValue) {
    value = newValue;  // volatile 쓰기: StoreStore + StoreLoad 펜스
}

public final void lazySet(int newValue) {
    VALUE.setRelease(this, newValue);
    // StoreStore 펜스만 (StoreLoad 펜스 없음) → set()보다 저렴
    // 사용 조건: 이후 volatile 읽기가 필요 없을 때
    // 예: 초기화 완료 신호를 한 방향으로만 전달할 때
}

// getAndSet() - 교환
public final int getAndSet(int newValue) {
    return (int) VALUE.getAndSet(this, newValue);
    // LOCK XCHG 명령어 → fetch-and-store 원자적
}

// updateAndGet(UnaryOperator) - 함수형 업데이트
public final int updateAndGet(IntUnaryOperator updateFunction) {
    int prev = get(), next = 0;
    for (boolean haveNext = false;;) {
        if (!haveNext)
            next = updateFunction.applyAsInt(prev);
        if (VALUE.weakCompareAndSetVolatile(this, prev, next))
            return next;
        haveNext = (prev == (prev = get()));
        // prev가 바뀌지 않았으면 next를 재계산하지 않음 (최적화)
    }
}
// weakCompareAndSetVolatile: 허위 실패 가능 (플랫폼에 따라)
//   → 루프로 재시도하므로 안전
//   → 일부 ARM에서 더 효율적

// accumulateAndGet(long, LongBinaryOperator)
public final long accumulateAndGet(long x, LongBinaryOperator accumulatorFn) {
    long prev = get(), next = 0;
    for (boolean haveNext = false;;) {
        if (!haveNext)
            next = accumulatorFn.applyAsLong(prev, x);
        if (VALUE.weakCompareAndSetVolatile(this, prev, next))
            return next;
        haveNext = (prev == (prev = get()));
    }
}
```

### 4. VarHandle 메모리 오더 모드

```
VarHandle이 제공하는 4단계 메모리 오더:

① Plain (재정렬 허용):
   VALUE.get(obj)        → 일반 필드 읽기 (펜스 없음)
   VALUE.set(obj, val)   → 일반 필드 쓰기
   용도: 단일 스레드 또는 이미 동기화된 컨텍스트

② Opaque (컴파일러 재정렬만 방지):
   VALUE.getOpaque(obj)
   VALUE.setOpaque(obj, val)
   용도: 같은 변수에 대한 다른 스레드와의 순서 보장 (약한 수준)

③ Acquire/Release (절반 펜스):
   VALUE.getAcquire(obj) → LoadLoad + LoadStore 펜스 (읽기 후 재정렬 방지)
   VALUE.setRelease(obj, val) → StoreStore 펜스 (쓰기 전 재정렬 방지)
   용도: Lock 구현 (lock 획득 = acquire, 해제 = release)
   비용: volatile보다 낮음 (StoreLoad 펜스 불필요)

④ Volatile (완전 순차 일관성):
   VALUE.getVolatile(obj) → 완전한 volatile 읽기
   VALUE.setVolatile(obj, val) → 완전한 volatile 쓰기
   VALUE.compareAndSet(obj, exp, new) → CAS + StoreLoad 펜스
   용도: 가장 강한 보장 필요 시

선택 가이드:
  게시(publish) 패턴 (초기화 후 참조 공개):
    쓰기: setRelease (StoreStore 펜스: 초기화 완료 후 참조 노출)
    읽기: getAcquire (LoadLoad 펜스: 참조 로드 후 초기화 데이터 읽기)
    → volatile의 StoreLoad 비용 절약
  
  일반 공유 카운터:
    compareAndSet (volatile 수준 필요)

비용 비교 (x86):
  plain:       ~4 사이클 (일반 읽기/쓰기)
  opaque:      ~4~8 사이클
  getAcquire:  ~4~8 사이클 (x86 LoadLoad 자동 보장)
  setRelease:  ~4~8 사이클 (x86 StoreStore 자동 보장)
  getVolatile: ~4~8 사이클
  setVolatile: ~100~300 사이클 (StoreLoad 펜스)
```

---

## 💻 실전 실험

### 실험 1: AtomicInteger 소스 단계별 추적

```java
import java.lang.invoke.*;
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicInternals {
    public static void main(String[] args) throws Exception {
        AtomicInteger ai = new AtomicInteger(10);

        // compareAndSet 직접 테스트
        System.out.println("초기값: " + ai.get());

        boolean success = ai.compareAndSet(10, 20);
        System.out.println("CAS(10→20): " + success + ", 결과: " + ai.get());

        success = ai.compareAndSet(10, 30);  // 실패 (현재값=20)
        System.out.println("CAS(10→30): " + success + ", 결과: " + ai.get());

        // updateAndGet 함수형 사용
        int result = ai.updateAndGet(v -> v * 2);
        System.out.println("updateAndGet(×2): " + result);

        // accumulateAndGet
        result = ai.accumulateAndGet(5, Integer::max);
        System.out.println("accumulateAndGet(max,5): " + result);
    }
}
```

### 실험 2: VarHandle로 일반 필드에 원자적 접근

```java
import java.lang.invoke.*;

public class VarHandleExample {
    private int counter = 0;
    private static final VarHandle COUNTER;

    static {
        try {
            COUNTER = MethodHandles.lookup()
                .findVarHandle(VarHandleExample.class, "counter", int.class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    public int incrementVolatile() {
        return (int) COUNTER.getAndAdd(this, 1) + 1;
    }

    public boolean cas(int expected, int update) {
        return COUNTER.compareAndSet(this, expected, update);
    }

    // Acquire/Release 패턴 (lock 구현 기반)
    public void setRelease(int val) {
        COUNTER.setRelease(this, val);  // StoreStore 펜스
    }

    public int getAcquire() {
        return (int) COUNTER.getAcquire(this);  // LoadLoad 펜스
    }

    public static void main(String[] args) throws InterruptedException {
        VarHandleExample obj = new VarHandleExample();
        Thread[] threads = new Thread[8];
        for (int i = 0; i < 8; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 100_000; j++) obj.incrementVolatile();
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        System.out.println("최종 카운터: " + obj.counter);  // 항상 800,000
    }
}
```

### 실험 3: JMH로 각 메서드 성능 비교

```java
import org.openjdk.jmh.annotations.*;
import java.lang.invoke.*;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Threads(4)
public class AtomicMethodsBenchmark {

    AtomicInteger ai = new AtomicInteger(0);
    volatile int vi = 0;
    int plainField = 0;

    static final VarHandle PLAIN_VH;
    static {
        try {
            PLAIN_VH = MethodHandles.lookup()
                .findVarHandle(AtomicMethodsBenchmark.class, "plainField", int.class);
        } catch (ReflectiveOperationException e) { throw new Error(e); }
    }

    @Benchmark public int atomicIncrement() { return ai.incrementAndGet(); }
    @Benchmark public void volatileWrite() { vi++; }  // 비원자적 (비교용)
    @Benchmark public int varhandleCas() {
        int prev;
        do { prev = (int) PLAIN_VH.getVolatile(this); }
        while (!PLAIN_VH.compareAndSet(this, prev, prev + 1));
        return prev + 1;
    }
    @Benchmark public int varhandleGetAndAdd() {
        return (int) PLAIN_VH.getAndAdd(this, 1) + 1;
    }
}
```

---

## 📊 성능/비용 비교

```
AtomicInteger 메서드별 비용 (단일 스레드, x86):

메서드                     | 어셈블리 명령어          | 비용 (사이클)
─────────────────────────┼──────────────────────┼──────────────
get()                    | MOV (volatile 읽기)   | ~4
set(v)                   | MOV + LOCK ADDL (펜스)| ~100~300
lazySet(v)               | MOV (StoreStore만)    | ~4~8
getAndSet(v)             | LOCK XCHG            | ~20~40
compareAndSet(exp, new)  | LOCK CMPXCHG         | ~20~40
getAndIncrement()        | LOCK XADD            | ~20~40
incrementAndGet()        | LOCK XADD + 1        | ~20~40
updateAndGet(fn)         | CAS 루프              | ~20~40 (무경쟁)
accumulateAndGet(x, fn)  | CAS 루프              | ~20~40 (무경쟁)

VarHandle 오더 모드별 비용 (x86):
  plain:       ~4 사이클
  opaque:      ~4~8 사이클
  getAcquire:  ~4~8 사이클 (x86 TSO로 LoadLoad 무료)
  setRelease:  ~4~8 사이클 (x86 TSO로 StoreStore 무료)
  compareAndSet: ~20~40 사이클 (LOCK CMPXCHG)
  setVolatile: ~100~300 사이클 (StoreLoad 펜스)
```

---

## ⚖️ 트레이드오프

```
Unsafe vs VarHandle 선택:

Unsafe (Java 8 이하 또는 레거시):
  장점: 성능 동일, 오프셋 계산 유연성
  단점: 비공개 API, Java 9+ 모듈 제한, 타입 안전성 없음

VarHandle (Java 9+, 권장):
  장점: 공식 API, 모듈 호환, 타입 안전, 성능 동일
  단점: MethodHandles.Lookup 설정 코드 필요

메모리 오더 선택:
  volatile 필요: compareAndSet, getVolatile, setVolatile
  단방향 boosts 패턴: getAcquire/setRelease (StoreLoad 비용 절약)
  성능 최우선, 약한 보장 OK: getOpaque/setOpaque
  동기화된 컨텍스트 내부: plain

updateAndGet vs 직접 CAS 루프:
  updateAndGet(fn): 코드 단순, 함수 객체 생성 오버헤드 (JIT 인라이닝 후 0)
  직접 CAS 루프: 상황별 최적화 가능 (early exit 등)
  → 대부분의 경우 updateAndGet이 충분
```

---

## 📌 핵심 정리

```
AtomicInteger / VarHandle 핵심:

Java 8 이하: Unsafe.compareAndSwapInt → LOCK CMPXCHG
Java 9+:    VarHandle.compareAndSet  → 동일 성능 (intrinsic)

주요 메서드:
  incrementAndGet() → LOCK XADD
  compareAndSet() → LOCK CMPXCHG
  getAndSet() → LOCK XCHG
  lazySet() → StoreStore 펜스만 (set보다 저렴)
  updateAndGet(fn) → CAS 루프 (함수형)

VarHandle 4단계 메모리 오더 (비용 순):
  plain < opaque < acquire/release < volatile
  x86에서 acquire/release는 거의 plain과 동일 (TSO로 자동 보장)

사용 원칙:
  카운터, 상태 플래그: AtomicInteger/AtomicLong
  일반 필드 원자적 접근: VarHandle (Java 9+)
  Lock 구현 내부: getAcquire/setRelease 패턴
  고경쟁 카운터: LongAdder (다음 문서)
```

---

## 🤔 생각해볼 문제

**Q1.** `lazySet()`은 언제 `set()` 대신 사용하는가? 실제 사용 사례는?

<details>
<summary>해설 보기</summary>

`lazySet()`(또는 `setRelease()`)은 StoreStore 펜스만 삽입하고 StoreLoad 펜스는 생략한다. 즉, 이 쓰기 이전의 다른 쓰기들이 먼저 완료되는 것은 보장하지만, 이후 읽기와의 순서는 보장하지 않는다.

실제 사용 사례:
1. **생산자-소비자에서 마지막 쓰기**: 생산자가 데이터를 다 쓴 후 "준비 완료" 플래그를 설정할 때 `lazySet()`을 사용하면 StoreLoad 비용 없이 이전 쓰기들이 먼저 보이는 것을 보장한다.

2. **`AtomicReference`로 구현한 스택/큐**: 노드의 next 포인터를 설정할 때 다른 스레드가 이 포인터를 읽기 전에 다른 필드들이 보여야 하지만, 이후 volatile 읽기가 필요 없을 때.

3. **`Striped64`(LongAdder 기반) 내부**: Cell의 value를 리셋할 때 `setRelease()`를 사용하여 이전 누적 값들이 먼저 플러시되도록 보장.

x86에서는 StoreLoad 펜스가 유일하게 비싼 펜스이므로, `lazySet()`과 `set()`의 성능 차이가 ARM에서 더 크다.

</details>

---

**Q2.** `AtomicInteger.updateAndGet(fn)`에서 함수 `fn`이 순수 함수(pure function)여야 하는 이유는?

<details>
<summary>해설 보기</summary>

`updateAndGet(fn)`은 CAS 실패 시 함수를 재호출한다. 함수에 부작용(side effect)이 있으면 CAS 실패 횟수만큼 부작용이 반복 실행된다.

예시:
```java
// 위험한 패턴: 부작용 있는 함수
counter.updateAndGet(v -> {
    log("updating");  // CAS 실패마다 반복 로깅
    return v + 1;
});

// 안전한 패턴: 순수 함수
counter.updateAndGet(v -> v + 1);
```

API 문서도 "적용 함수는 부작용이 없어야 한다"고 명시한다. 같은 이유로 `accumulateAndGet(x, fn)`의 `fn`도 순수 함수여야 한다. 재시도 횟수를 예측할 수 없기 때문에 부작용의 횟수도 예측 불가능해진다.

</details>

---

**Q3.** `VarHandle.weakCompareAndSet()`은 `compareAndSet()`과 어떻게 다른가? 언제 사용하는가?

<details>
<summary>해설 보기</summary>

`weakCompareAndSet()`은 "허위 실패(spurious failure)"가 가능하다. 즉, expected 값이 현재 값과 같더라도 false를 반환할 수 있다. 추가적으로 메모리 오더링 보장이 더 약하다.

왜 존재하는가: 일부 아키텍처(ARM의 LL/SC 기반 구현)에서 일반 CAS를 구현할 때 허위 실패가 발생할 수 있다. x86에서는 LOCK CMPXCHG가 허위 실패 없이 동작하지만, ARM의 STXR(Store-Exclusive)은 메모리 버스 경합 없이도 실패할 수 있다. `weakCompareAndSet`은 이 허위 실패를 허용하므로 ARM에서 더 간단한 명령어 사용 가능 → 성능 이점.

사용 조건: 항상 루프 안에서 사용해야 안전하다.
```java
// 올바른 weakCompareAndSet 사용
int prev;
do {
    prev = (int) VH.get(obj);
} while (!VH.weakCompareAndSet(obj, prev, prev + 1));
// 허위 실패해도 루프가 재시도하므로 최종적으로 성공
```

`AtomicInteger.updateAndGet()`이 내부적으로 `weakCompareAndSetVolatile()`을 사용하는 이유도 이 때문이다.

</details>

---

<div align="center">

**[⬅️ 이전: CAS와 ABA 문제](./01-cas-cpu-level.md)** | **[홈으로 🏠](../README.md)** | **[다음: AtomicReference와 AtomicStampedReference ➡️](./03-atomic-reference-stamped.md)**

</div>

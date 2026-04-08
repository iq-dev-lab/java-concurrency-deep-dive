# 메모리 오더링 — VarHandle의 Acquire/Release/Opaque

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `VarHandle.getAcquire()` / `setRelease()` / `getOpaque()` / `setOpaque()` 각각이 보장하는 메모리 순서 수준은?
- 순서 보장 약화에 따른 성능 향상 폭은 어느 정도인가?
- Acquire/Release 패턴이 volatile보다 저렴하지만 안전한 이유는?
- 각 모드를 안전하게 사용할 수 있는 조건은 무엇인가?
- C++의 `std::memory_order`와 어떻게 대응되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`VarHandle`의 메모리 오더링은 Java 9에서 도입된 정밀 도구다. 라이브러리 개발자나 성능 최적화가 중요한 코드에서, volatile의 비용이 지나치게 높을 때 더 약한 순서 보장으로 성능을 높이면서 안전성을 유지할 수 있다. `LongAdder`, `StampedLock` 등 JDK 내부 클래스가 이 모드들을 직접 활용한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Acquire/Release가 "반쪽 volatile"이라서 위험하다는 오해
  "volatile을 반으로 나누면 버그가 생기지 않나?"
  → Acquire/Release는 특정 패턴에서 완전하게 안전
  → 게시(publication) 패턴: setRelease(쓰기) + getAcquire(읽기) = volatile과 동등
  → 문제는 "full fence가 필요한 곳에 쓸 때"

실수 2: Opaque를 volatile 대신 사용하면 항상 빠를 것이라는 기대
  → Opaque는 가장 약한 순서 보장 (컴파일러 재정렬만 방지)
  → x86에서 volatile과 거의 비용 차이 없음 (x86 TSO)
  → ARM에서 의미 있는 차이
  → 잘못 사용하면 가시성 버그 발생

실수 3: plain 모드가 완전히 안전하다고 가정
  VALUE.set(this, newValue);  // plain write
  → 다른 스레드에서 읽기 시 언제 보일지 보장 없음
  → 동기화된 컨텍스트 밖에서는 데이터 레이스
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// VarHandle 메모리 오더 올바른 사용 패턴

import java.lang.invoke.*;

class PublicationExample {
    private int data = 0;
    private boolean ready = false;

    static final VarHandle DATA, READY;
    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            DATA  = l.findVarHandle(PublicationExample.class, "data", int.class);
            READY = l.findVarHandle(PublicationExample.class, "ready", boolean.class);
        } catch (ReflectiveOperationException e) { throw new Error(e); }
    }

    // 패턴 1: 게시(publication) - setRelease/getAcquire
    void publish(int value) {
        DATA.set(this, value);           // plain: data 쓰기
        READY.setRelease(this, true);   // release: StoreStore 펜스 → data가 먼저 보임 보장
    }

    int consume() {
        if ((boolean) READY.getAcquire(this)) {  // acquire: LoadLoad 펜스
            return (int) DATA.get(this);          // data 읽기 → ready 읽기 이후
        }
        return -1;
    }

    // 패턴 2: 진행 상황 체크 (Opaque)
    // "이 스레드는 종료해야 하는가?"를 주기적으로 확인
    private boolean cancelled = false;
    static final VarHandle CANCELLED;
    // ... VarHandle 설정 ...

    void checkCancelled() {
        // Opaque: 컴파일러가 루프 밖으로 이동 방지, CPU 재정렬은 허용
        if ((boolean) CANCELLED.getOpaque(this)) {
            throw new CancellationException();
        }
    }

    // 패턴 3: 완전한 순서 필요 시 volatile 사용
    public boolean compareAndSet(int exp, int upd) {
        return DATA.compareAndSet(this, exp, upd);  // volatile 수준
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 4가지 메모리 오더 수준

```
Java VarHandle이 제공하는 메모리 오더 수준 (강한 것 → 약한 것):

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
수준 4: Volatile (완전한 순차 일관성, Sequential Consistency)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  getVolatile(obj)       → LoadLoad + LoadStore 펜스 (읽기)
  setVolatile(obj, val)  → StoreStore + StoreLoad 펜스 (쓰기)
  compareAndSet(...)     → volatile 수준 CAS

  보장: 모든 스레드에서 동일한 실행 순서 관찰
  비용: StoreLoad 펜스 (x86 LOCK ADDL/MFENCE)
  = Java volatile 키워드와 동일

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
수준 3: Acquire/Release (절반 펜스)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  getAcquire(obj)        → LoadLoad + LoadStore 펜스 (읽기 후 이후 연산 재정렬 방지)
  setRelease(obj, val)   → StoreStore 펜스 (쓰기 전 이전 연산 재정렬 방지)

  보장: setRelease 이전 쓰기 → setRelease → getAcquire → getAcquire 이후 읽기 순서
  = happens-before 체인 구성 가능 (lock/unlock 패턴)
  비용: x86에서 getAcquire ≈ 무료 (TSO), setRelease ≈ 무료 (TSO)
       ARM에서 getAcquire = LDAR, setRelease = STLR (중간 비용)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
수준 2: Opaque (컴파일러 재정렬만 방지)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  getOpaque(obj)         → 다른 읽기/쓰기와의 순서 보장 없음
  setOpaque(obj, val)    → 하지만 컴파일러가 같은 변수 접근을 최적화/재정렬 금지

  보장: 같은 변수에 대한 접근이 순서대로 보임 (같은 변수 내 일관성)
  비용: x86에서 volatile과 거의 동일 (단순 mov)
       ARM에서 의미 있는 절약 (메모리 배리어 없음)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
수준 1: Plain (재정렬 허용, 가시성 보장 없음)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  get(obj)               → 일반 필드 읽기 (레지스터 캐싱 가능)
  set(obj, val)          → 일반 필드 쓰기 (Store Buffer 활용)

  보장: 없음 (단일 스레드 의미론만 유지)
  비용: 가장 낮음 (~4사이클)
  용도: 이미 동기화된 컨텍스트 내부, 단일 스레드
```

### 2. Acquire/Release 패턴 상세

```
Acquire/Release가 volatile과 동등한 경우:

쓰기 측 (Producer):
  // StoreStore 펜스: 이전 모든 쓰기가 setRelease 이전에 완료
  FIELD_A.set(obj, valueA);     // 데이터 준비
  FIELD_B.set(obj, valueB);     // 데이터 준비
  READY.setRelease(obj, true);  // "준비 완료" 신호 게시
  // 펜스: FIELD_A, FIELD_B 쓰기가 setRelease보다 나중으로 이동 불가

읽기 측 (Consumer):
  // LoadLoad 펜스: getAcquire 이후 모든 읽기가 이 읽기 이후에
  if ((boolean) READY.getAcquire(obj)) {
      int a = (int) FIELD_A.get(obj);  // READY 읽기 이후 → a, b가 보임 보장
      int b = (int) FIELD_B.get(obj);  // ...
  }
  // 펜스: FIELD_A, FIELD_B 읽기가 getAcquire보다 이전으로 이동 불가

happens-before 체인:
  setRelease(READY, true)
    →HB→ (volatile 변수 규칙 유사) getAcquire(READY)
    →HB→ FIELD_A.get(), FIELD_B.get()
  전이성:
    FIELD_A.set(), FIELD_B.set()
    →HB→ setRelease(READY, true)
    →HB→ getAcquire(READY)
    →HB→ FIELD_A.get(), FIELD_B.get()
  결론: a = valueA, b = valueB 보장

volatile과의 차이:
  volatile 쓰기: StoreStore + StoreLoad 펜스
  setRelease:    StoreStore 펜스만 (StoreLoad 없음)
  → volatile은 "이전 모든 쓰기가 이 쓰기 이후의 모든 읽기에 보임"
  → setRelease는 "이전 쓰기가 이 게시 신호 이후의 읽기에 보임"
  → 결과: 게시 패턴에서 동등, StoreLoad 비용 절약

StoreLoad 펜스가 필요한 경우:
  Thread A: flag = true (setRelease)
  Thread B: data = 1 (setRelease)
  Thread A: assert data == 1 (getAcquire) → 보장 안 됨!
  Thread B: assert flag == true (getAcquire) → 보장 안 됨!
  → 이 패턴은 Dekker 알고리즘 같은 상호 관찰 → volatile(StoreLoad) 필요
```

### 3. x86 vs ARM에서의 비용 차이

```
x86 (Intel/AMD) — TSO 모델:
  LoadLoad 재정렬: 없음 (TSO 자동 보장)
  LoadStore 재정렬: 없음 (TSO 자동 보장)
  StoreStore 재정렬: 없음 (TSO 자동 보장)
  StoreLoad 재정렬: 발생! → StoreLoad 펜스(MFENCE/LOCK ADDL) 필요

  x86에서 각 모드 구현:
    getAcquire:  mov (일반 읽기와 동일) → 거의 무료
    setRelease:  mov (일반 쓰기와 동일) → 거의 무료
    getVolatile: mov (LoadLoad 자동 보장)
    setVolatile: mov + LOCK ADDL (StoreLoad 펜스) → 비쌈

  결론: x86에서 getAcquire/setRelease ≈ plain (거의 차이 없음)
  → ARM 타겟 없이 x86만 배포한다면 획득/해제 최적화 효과 미미

ARM (AArch64) — 약한 메모리 모델:
  모든 유형의 재정렬 발생
  
  각 모드 구현:
    get (plain):     LDR (일반 로드)
    getOpaque:       LDR (plain과 동일, 컴파일러 제약만)
    getAcquire:      LDAR (Load-Acquire) → LoadLoad + LoadStore 펜스
    getVolatile:     LDAR + (추가 보장)
    set (plain):     STR (일반 저장)
    setRelease:      STLR (Store-Release) → StoreStore + StoreLoad 방지
    setVolatile:     STLR + DMB ISH (완전 펜스)

  ARM에서 plain vs volatile 비용 차이가 훨씬 큼
  → AWS Graviton, Apple M 시리즈 타겟에서 acquire/release 최적화 효과 큼
```

### 4. JDK 내부 활용 사례

```
JDK 코드에서 VarHandle 오더 활용:

Striped64 (LongAdder 기반):
  Cell.cas() → compareAndSet (volatile)
  cellsBusy: CAS로 Cell 배열 초기화 직렬화
  base: getVolatile/compareAndSet

StampedLock:
  state: getVolatile → 락 상태 읽기
  readerShouldBlock(): getOpaque 활용 가능

ThreadPoolExecutor:
  ctl (상태 + 스레드 수 pack): getVolatile/compareAndSet

ConcurrentHashMap:
  Node의 val, next: getVolatile/setVolatile
  tab 배열 원소: tabAt() = Unsafe.getObjectVolatile (volatile 접근)
  casTabAt() = Unsafe.compareAndSetObject

FutureTask:
  state: volatile (LOCK 없이 상태 전환)
  outcome: setRelease 패턴 (결과 게시 후 상태 전환)

이 패턴들의 공통점:
  "데이터를 먼저 쓰고(setRelease), 신호를 게시하고(setRelease/CAS)
   신호를 확인하고(getAcquire/getVolatile), 데이터를 읽는(get)" 순서
```

---

## 💻 실전 실험

### 실험 1: JMH로 각 오더 모드 처리량 비교

```java
import org.openjdk.jmh.annotations.*;
import java.lang.invoke.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Threads(8)
public class MemoryOrderBenchmark {

    int field = 0;
    static final VarHandle VH;
    static {
        try {
            VH = MethodHandles.lookup()
                .findVarHandle(MemoryOrderBenchmark.class, "field", int.class);
        } catch (ReflectiveOperationException e) { throw new Error(e); }
    }

    @Benchmark public int plainRead()   { return (int) VH.get(this); }
    @Benchmark public int opaqueRead()  { return (int) VH.getOpaque(this); }
    @Benchmark public int acquireRead() { return (int) VH.getAcquire(this); }
    @Benchmark public int volatileRead(){ return (int) VH.getVolatile(this); }

    @Benchmark public void plainWrite()   { VH.set(this, 1); }
    @Benchmark public void opaqueWrite()  { VH.setOpaque(this, 1); }
    @Benchmark public void releaseWrite() { VH.setRelease(this, 1); }
    @Benchmark public void volatileWrite(){ VH.setVolatile(this, 1); }
    @Benchmark public void casWrite()     { VH.compareAndSet(this, 0, 1); }
}
// x86 결과: plainRead ≈ opaqueRead ≈ acquireRead ≈ volatileRead (거의 동일)
//            plainWrite ≈ opaqueWrite ≈ releaseWrite << volatileWrite
// ARM 결과: plain < opaque ≈ acquire/release < volatile 순으로 차이 남
```

### 실험 2: Acquire/Release로 안전한 게시 구현

```java
import java.lang.invoke.*;
import java.util.concurrent.CountDownLatch;

public class SafePublication {
    private int[] data = new int[0];
    private boolean published = false;

    static final VarHandle DATA, PUBLISHED;
    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            DATA      = l.findVarHandle(SafePublication.class, "data", int[].class);
            PUBLISHED = l.findVarHandle(SafePublication.class, "published", boolean.class);
        } catch (ReflectiveOperationException e) { throw new Error(e); }
    }

    void prepare(int size) {
        int[] arr = new int[size];
        for (int i = 0; i < size; i++) arr[i] = i * 2;
        DATA.set(this, arr);                        // plain: data 설정
        PUBLISHED.setRelease(this, true);           // release: data가 먼저 보임 보장
    }

    int[] tryGet() {
        if ((boolean) PUBLISHED.getAcquire(this)) { // acquire: 이후 읽기가 이 이후
            return (int[]) DATA.get(this);          // plain: 이미 acquire 이후
        }
        return null;
    }

    public static void main(String[] args) throws InterruptedException {
        SafePublication pub = new SafePublication();
        CountDownLatch latch = new CountDownLatch(1);

        Thread producer = new Thread(() -> {
            pub.prepare(1000);
            latch.countDown();
        });
        Thread consumer = new Thread(() -> {
            try {
                latch.await();
                int[] result = pub.tryGet();
                System.out.println("데이터 크기: " + (result != null ? result.length : "null"));
                System.out.println("첫 번째 값: " + (result != null ? result[0] : "null"));
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });

        producer.start();
        consumer.start();
        producer.join();
        consumer.join();
    }
}
```

---

## 📊 성능/비용 비교

```
VarHandle 오더 모드별 비용 (x86 기준, 8스레드 JMH):

읽기 연산 처리량 (ops/ms):
  plain     ≈ 2,000,000
  opaque    ≈ 1,900,000  (-5%)
  acquire   ≈ 1,850,000  (-7%)
  volatile  ≈ 1,800,000  (-10%)
  → x86에서 읽기 모드 차이 미미 (TSO 자동 보장)

쓰기 연산 처리량 (ops/ms):
  plain     ≈ 2,000,000
  opaque    ≈ 1,900,000  (-5%)
  release   ≈ 1,850,000  (-7%)
  volatile  ≈   100,000  (-95%!) ← StoreLoad 펜스 비용
  → volatile 쓰기가 압도적으로 비쌈 (StoreLoad 펜스)

ARM 기준 (AWS Graviton, 차이 더 크게 남):
  plain     ≈ 2,500,000
  opaque    ≈ 2,400,000  (-4%)
  acquire   ≈ 1,500,000  (-40%)  ← LDAR 명령어 비용
  release   ≈ 1,600,000  (-36%)  ← STLR 명령어 비용
  volatile  ≈   400,000  (-84%)  ← STLR + DMB ISH

C++ 메모리 오더와의 대응:
  Java plain     ↔ C++ memory_order_relaxed (유사, 완전 동일 아님)
  Java opaque    ↔ C++ memory_order_relaxed (더 가까움)
  Java acquire   ↔ C++ memory_order_acquire
  Java release   ↔ C++ memory_order_release
  Java volatile  ↔ C++ memory_order_seq_cst
```

---

## ⚖️ 트레이드오프

```
어느 모드를 선택할지 결정 기준:

volatile (getVolatile/setVolatile):
  happens-before 완전 보장 필요
  여러 스레드 간 양방향 관찰 패턴 (Dekker 알고리즘 등)
  일반 공유 변수, 특별한 이유 없으면 이것이 기본

Acquire/Release (getAcquire/setRelease):
  게시(publication) 패턴 (쓰기 후 "완료" 신호 게시)
  Lock 구현 내부 (획득 = acquire, 해제 = release)
  비대칭 통신 (한 방향으로만 신호 전달)
  ARM 환경에서 의미 있는 성능 향상

Opaque (getOpaque/setOpaque):
  진행 상황 폴링 ("취소됐는가?" 주기적 확인)
  같은 변수 내 일관성만 필요 (다른 변수와의 순서 불필요)
  JIT 레지스터 캐싱 방지만 필요
  → 실제로 opaque가 plain보다 크게 빠른 경우는 드묾

Plain (get/set):
  이미 동기화된 컨텍스트 내부 (synchronized 블록 안)
  단일 스레드 사용 확신 시
  다른 동기화 메커니즘이 이미 보장하는 경우

실무 조언:
  대부분의 코드: volatile (오버엔지니어링 방지)
  라이브러리/프레임워크 내부 최적화: acquire/release
  x86 전용 배포라면: acquire/release ≈ volatile (성능 이점 적음)
  ARM 포함 크로스 플랫폼: acquire/release 최적화 가치 있음
```

---

## 📌 핵심 정리

```
VarHandle 메모리 오더링 핵심:

4단계 (강한 순서 → 약한 순서):
  volatile:       완전한 순차 일관성 (StoreLoad 포함)
  acquire/release: 절반 펜스 (게시 패턴에 안전)
  opaque:         컴파일러 재정렬만 방지
  plain:          보장 없음 (동기화된 컨텍스트 내부 전용)

x86 vs ARM 비용 차이:
  x86 TSO: volatile 쓰기만 비쌈 (~100ns), 나머지 거의 동일
  ARM:     acquire/release도 의미 있는 비용, volatile은 더 비쌈

Acquire/Release 패턴:
  setRelease(data) → setRelease(flag)  // 쓰기 측
  getAcquire(flag) → get(data)         // 읽기 측
  → volatile과 동등한 happens-before 보장 (게시 패턴)
  → StoreLoad 펜스 비용 없음

JDK 활용:
  LongAdder, StampedLock, ThreadPoolExecutor 등 내부에서 활용

일반 개발자 원칙:
  기본: volatile
  최적화 필요 + 게시 패턴: acquire/release
  직접 메모리 오더 조작은 충분한 이해 후
```

---

## 🤔 생각해볼 문제

**Q1.** `setRelease` 이후 `getVolatile`로 읽으면 happens-before가 성립하는가? 반대로 `setVolatile` 이후 `getAcquire`는?

<details>
<summary>해설 보기</summary>

둘 다 happens-before가 성립한다. JMM 스펙에서 정의하는 happens-before는 "volatile 변수 규칙"을 기반으로 하는데, `setRelease`는 최소 release semantics를, `getVolatile`은 최소 acquire semantics를 제공하므로 이들 간에도 happens-before가 성립한다.

더 강한 보장이 더 약한 것을 포함한다는 원칙:
- setVolatile ≥ setRelease (volatile이 더 강함)
- getVolatile ≥ getAcquire (volatile이 더 강함)
- setRelease + getAcquire = happens-before 성립
- setVolatile + getAcquire = happens-before 성립 (setVolatile ≥ setRelease이므로)
- setRelease + getVolatile = happens-before 성립 (getVolatile ≥ getAcquire이므로)

</details>

---

**Q2.** `getOpaque`는 "같은 변수 내 일관성"을 보장한다고 했다. 이것이 실질적으로 어떤 버그를 방지하는가?

<details>
<summary>해설 보기</summary>

Opaque는 JIT 컴파일러가 해당 변수 접근을 레지스터에 캐시하거나 불필요한 읽기를 제거하는 것을 방지한다. 다음 버그를 방지한다:

```java
// volatile/opaque 없이 루프:
while (condition) {
    // condition이 레지스터에 캐시될 수 있음
    // → 다른 스레드가 condition을 false로 바꿔도 감지 못함
}

// opaque 사용:
while ((boolean) CONDITION.getOpaque(this)) {
    // 매 루프마다 실제 메모리에서 읽음
    // CPU 재정렬은 허용하되, JIT 캐싱은 방지
}
```

CPU 재정렬을 허용하므로 가시성의 완전한 보장은 아니지만, "이전에 쓴 값이 이 읽기에서 안 보이는 경우"는 이론적으로 가능하다. 다만 현실적으로 CPU 캐시 일관성(MESI)이 매우 빠르게 동작하므로 stale 값을 보는 창이 극히 짧다.

x86에서 opaque = plain (추가 명령어 없음, 컴파일러 제약만). ARM에서도 opaque = plain. 두 아키텍처 모두 하드웨어 수준의 추가 보장 없음. 오직 컴파일러/JIT 레벨의 최적화만 억제.

</details>

---

**Q3.** 스핀 락을 `setRelease`/`getAcquire`로 구현하면 synchronized와 동일한 가시성 보장을 제공하는가?

<details>
<summary>해설 보기</summary>

그렇다. 올바르게 구현한다면 동일한 가시성 보장을 제공한다.

```java
class SpinLock {
    private volatile int locked = 0;  // 또는 VarHandle
    static final VarHandle LOCKED; // ... 설정 ...

    void lock() {
        while (!LOCKED.compareAndSet(this, 0, 1)) {
            Thread.onSpinWait();  // CPU 힌트
        }
        // compareAndSet = volatile 수준 CAS
        // → getAcquire semantics 포함
    }

    void unlock() {
        LOCKED.setRelease(this, 0);  // StoreStore 펜스
    }
}
```

이 구조에서:
- `lock()`: CAS 성공 시 acquire semantics → 이후 읽기들이 이 이후에
- `unlock()`: setRelease → 임계 구역의 모든 쓰기가 이 이전에 완료

따라서 lock/unlock 쌍이 happens-before 체인을 형성하여 synchronized와 동일한 가시성을 제공한다. JDK의 `ReentrantLock`이 AQS를 통해 이 패턴을 사용한다. 단, `Thread.onSpinWait()`를 사용하지 않으면 CPU를 계속 낭비하므로 스핀락은 짧은 임계 구역에서만 적합하다.

</details>

---

<div align="center">

**[⬅️ 이전: Lock-Free 자료구조](./05-lock-free-data-structures.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — ConcurrentHashMap ➡️](../concurrent-data-structures/01-concurrent-hashmap.md)**

</div>

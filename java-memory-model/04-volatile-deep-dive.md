# volatile 완전 분해 — 메모리 펜스와 한계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- volatile 읽기/쓰기 시 어떤 메모리 펜스가 삽입되는가?
- `-XX:+PrintAssembly`로 어셈블리에서 volatile의 구현을 어떻게 확인하는가?
- volatile이 가시성을 보장하지만 원자성을 보장하지 못하는 이유는?
- `count++`에 volatile이 충분하지 않은 이유는?
- volatile이 적합한 패턴과 부적합한 패턴은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`volatile`은 Java에서 가장 많이 오용되는 키워드 중 하나다. "캐시를 안 쓴다", "동기화 없이 스레드 안전하다", "synchronized 대신 쓸 수 있다"라는 오해가 만연하다. volatile이 실제로 무엇을 하고 무엇을 못 하는지 이해하면, 언제 volatile로 충분하고 언제 더 강한 동기화가 필요한지 정확히 판단할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: volatile로 카운터 구현
  private volatile int count = 0;

  public void increment() {
      count++;  // 스레드 안전하지 않음!
  }

  count++는 세 단계: read(count) → +1 → write(count)
  volatile은 각 읽기/쓰기를 원자적으로 만들지 않음
  두 스레드가 동시에 count=5를 읽고 각자 6을 쓰면 → count=6 (7이어야 함)

실수 2: volatile이 synchronized를 대체한다는 오해
  class Cache {
      private volatile Map<String, Object> cache = new HashMap<>();

      public void update(String key, Object value) {
          cache.put(key, value);  // HashMap은 스레드 안전하지 않음!
      }
  }
  volatile은 cache 참조 자체에만 적용됨
  HashMap 내부 구조 변경은 별도 동기화 필요

실수 3: 복합 조건 검사에 volatile만 사용
  volatile boolean initialized = false;

  // Thread A
  if (!initialized) {
      initialize();      // 초기화 수행
      initialized = true;
  }
  // Thread B가 동시에 같은 코드 실행 → 이중 초기화 발생
  // check-then-act는 원자적이지 않음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)

```java
// volatile의 올바른 사용 패턴

// 패턴 1: 종료/취소 신호 플래그 (단순 boolean 플래그)
class Worker {
    private volatile boolean cancelled = false;

    public void cancel() { cancelled = true; }

    public void run() {
        while (!cancelled) {
            doWork();
        }
    }
}

// 패턴 2: 게시(publication) 패턴 (완전히 초기화된 객체 참조 공유)
class Holder {
    private volatile Config config;  // 참조 자체가 volatile

    public void updateConfig(Config newConfig) {
        // newConfig가 완전히 초기화된 후 참조를 공개
        config = newConfig;  // volatile 쓰기 → happens-before 확립
    }

    public String getValue() {
        Config c = config;  // volatile 읽기 (참조를 로컬에 캐시)
        return c != null ? c.getValue() : null;
    }
}

// 패턴 3: volatile이 부족한 경우 → AtomicXxx 사용
private AtomicInteger count = new AtomicInteger(0);
public void increment() { count.incrementAndGet(); }  // CAS 기반 원자적 연산

// 패턴 4: 복합 상태 → synchronized
private int x, y;
private volatile boolean ready = false;

public synchronized void setXY(int x, int y) {
    this.x = x;
    this.y = y;
    // ready는 volatile이지만 x,y와의 원자성을 위해 synchronized로 감쌈
}
```

---

## 🔬 내부 동작 원리

### 1. volatile이 삽입하는 메모리 펜스 4종

```
메모리 펜스(Memory Fence/Barrier): CPU에게 "이 지점을 기준으로 메모리 작업의 순서를 지켜라"는 명령

4가지 펜스 유형:
  LoadLoad:  펜스 이전 Load → 펜스 이후 Load 순서 보장
  LoadStore: 펜스 이전 Load → 펜스 이후 Store 순서 보장
  StoreStore: 펜스 이전 Store → 펜스 이후 Store 순서 보장
  StoreLoad: 펜스 이전 Store → 펜스 이후 Load 순서 보장 (가장 비쌈)
             = Store Buffer를 완전히 플러시하는 Full Fence

volatile 읽기 시 삽입되는 펜스:
  ...
  [LoadLoad 펜스]    ← volatile 읽기 이전 모든 Load가 먼저
  v = volatileVar;   ← volatile 읽기
  [LoadStore 펜스]   ← volatile 읽기 이전 Load가 이후 Store보다 먼저
  ...

volatile 쓰기 시 삽입되는 펜스:
  ...
  [StoreStore 펜스]  ← 이전 모든 Store가 volatile 쓰기 전에 완료
  volatileVar = v;   ← volatile 쓰기
  [StoreLoad 펜스]   ← volatile 쓰기가 이후 Load보다 먼저 (가장 중요)
  ...

x86에서의 실제 구현:
  LoadLoad/LoadStore: x86 TSO에서 자동 보장 (펜스 불필요)
  StoreStore:         x86 TSO에서 자동 보장 (펜스 불필요)
  StoreLoad:          필요! → lock addl $0x0, (%rsp) 명령어 사용
                       (MFENCE와 동일 효과지만 더 빠름)

결론:
  x86에서 volatile 읽기: 추가 펜스 거의 없음 → 거의 무료
  x86에서 volatile 쓰기: lock addl 삽입 → 100~300사이클 비용
```

### 2. PrintAssembly로 volatile 구현 확인

```bash
# hsdis 라이브러리 필요 (JVM 플러그인)
# Ubuntu: apt-get install libhsdis0-fcml
# 또는 빌드: https://github.com/openjdk/jdk/tree/master/src/utils/hsdis

cat > VolatileAsm.java << 'EOF'
public class VolatileAsm {
    static volatile int vField = 0;
    static int nField = 0;

    public static void writeVolatile(int v) { vField = v; }
    public static void writeNormal(int v)   { nField = v; }
    public static int  readVolatile()       { return vField; }
    public static int  readNormal()         { return nField; }

    public static void main(String[] args) {
        for (int i = 0; i < 100_000; i++) {
            writeVolatile(i); readVolatile();
            writeNormal(i);   readNormal();
        }
    }
}
EOF

javac VolatileAsm.java
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintAssembly \
     -XX:CompileCommand=compileonly,VolatileAsm.writeVolatile \
     -XX:CompileCommand=compileonly,VolatileAsm.writeNormal \
     VolatileAsm 2>&1 | grep -E "(mov|lock|mfence)" | head -20

# 예상 출력:
# writeVolatile():
#   mov    DWORD PTR [rip+offset],esi    ← 실제 쓰기
#   lock addl DWORD PTR [rsp],0x0       ← StoreLoad 펜스 ← volatile 증거
#
# writeNormal():
#   mov    DWORD PTR [rip+offset],esi    ← 실제 쓰기만 (펜스 없음)
```

### 3. volatile이 보장하는 것 vs 보장하지 않는 것

```
✅ volatile이 보장하는 것:

① 가시성 (Visibility):
   volatile 변수에 대한 쓰기는 이후 읽기에 항상 보임
   StoreLoad 펜스로 Store Buffer 플러시
   다른 코어의 캐시 Invalidate

② 재정렬 금지 (Ordering):
   volatile 읽기/쓰기를 기준으로 재정렬 제한
   happens-before 관계 확립

③ 단일 읽기/쓰기의 원자성:
   int, long, reference 타입의 단일 읽기/쓰기는 원자적
   (long/double도 volatile 시 64비트 원자적 읽기/쓰기 보장)

❌ volatile이 보장하지 않는 것:

① 복합 연산의 원자성:
   count++    → read(count) + increment + write(count) = 3단계 비원자적
   check-then-act → 조건 확인 + 실행 사이에 끼어들기 가능

② 객체 내부 상태의 일관성:
   volatile Config config;
   config = new Config(x, y);  // Config 내부 x, y는 volatile 아님
   → config 참조가 volatile이어도 Config 내부 필드는 별도 보장 없음
   (단, 생성자 완료 후 참조가 공개되면 final 필드는 보장)

③ 여러 volatile 변수 간의 원자성:
   volatile int x, y;
   x = 1;  // 두 쓰기가 원자적으로 일어나지 않음
   y = 2;  // 다른 스레드가 x=1, y=0을 볼 수 있음
```

### 4. check-then-act 안티패턴과 해결책

```
안티패턴: volatile로 싱글톤 구현 시도
  class BadSingleton {
      private static volatile BadSingleton instance;

      public static BadSingleton getInstance() {
          if (instance == null) {     // 1. 확인 (check)
              instance = new BadSingleton();  // 2. 행동 (act)
          }
          return instance;
      }
  }

  문제: 두 스레드가 동시에 1번에서 null을 보고 각자 2번 실행
  → 두 개의 인스턴스 생성 (싱글톤 위반)
  volatile은 이 시나리오에서 race condition 방지 불가

해결책 1: Double-Checked Locking + volatile (Java 5+)
  private static volatile Singleton instance;

  public static Singleton getInstance() {
      if (instance == null) {          // 외부 체크 (성능)
          synchronized (Singleton.class) {
              if (instance == null) {  // 내부 체크 (정확성)
                  instance = new Singleton();
              }
          }
      }
      return instance;
  }
  // volatile 없으면 부분 초기화된 객체를 볼 수 있음 (06-dcl 참조)

해결책 2: Initialization-on-demand Holder (가장 권장)
  class Singleton {
      private Singleton() {}

      private static class Holder {
          static final Singleton INSTANCE = new Singleton();
          // 클래스 로딩 시 JVM이 초기화 보장 (happens-before)
      }

      public static Singleton getInstance() {
          return Holder.INSTANCE;  // 클래스 로딩 시 한 번만 초기화
      }
  }

해결책 3: AtomicReference.compareAndSet()
  AtomicReference<Singleton> ref = new AtomicReference<>();

  public Singleton getInstance() {
      Singleton s = ref.get();
      if (s == null) {
          Singleton newS = new Singleton();
          ref.compareAndSet(null, newS);  // CAS: null이면 설정
          s = ref.get();
      }
      return s;
  }
  // 단점: 여러 인스턴스가 생성될 수 있음 (하나만 등록되지만)
```

---

## 💻 실전 실험

### 실험 1: volatile count++ 경쟁 조건 재현

```java
public class VolatileCounterBug {
    private static volatile int count = 0;
    private static final int THREADS = 10;
    private static final int ITERATIONS = 100_000;

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[THREADS];
        for (int i = 0; i < THREADS; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < ITERATIONS; j++) {
                    count++;  // 비원자적 연산!
                }
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();

        int expected = THREADS * ITERATIONS;
        System.out.printf("기대값: %,d%n", expected);
        System.out.printf("실제값: %,d%n", count);
        System.out.printf("손실:   %,d%n", expected - count);
        // 손실이 0이 아님 → volatile count++는 race condition 있음
    }
}
```

### 실험 2: volatile vs AtomicInteger 정확성 비교

```java
import java.util.concurrent.atomic.AtomicInteger;

public class VolatileVsAtomic {
    static volatile int volatileCount = 0;
    static AtomicInteger atomicCount = new AtomicInteger(0);
    static final int THREADS = 10, ITERS = 100_000;

    public static void main(String[] args) throws InterruptedException {
        // volatile count++
        Thread[] vThreads = new Thread[THREADS];
        for (int i = 0; i < THREADS; i++) {
            vThreads[i] = new Thread(() -> {
                for (int j = 0; j < ITERS; j++) volatileCount++;
            });
            vThreads[i].start();
        }
        for (Thread t : vThreads) t.join();

        // AtomicInteger
        Thread[] aThreads = new Thread[THREADS];
        for (int i = 0; i < THREADS; i++) {
            aThreads[i] = new Thread(() -> {
                for (int j = 0; j < ITERS; j++) atomicCount.incrementAndGet();
            });
            aThreads[i].start();
        }
        for (Thread t : aThreads) t.join();

        System.out.printf("기대:   %,d%n", THREADS * ITERS);
        System.out.printf("volatile: %,d (손실 %,d)%n",
            volatileCount, THREADS * ITERS - volatileCount);
        System.out.printf("atomic:   %,d (손실 %,d)%n",
            atomicCount.get(), THREADS * ITERS - atomicCount.get());
    }
}
```

### 실험 3: JMH로 volatile 읽기/쓰기 비용 측정

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Warmup(iterations = 3)
@Measurement(iterations = 5)
public class VolatileCostBenchmark {

    volatile int vField = 0;
    int nField = 0;

    @Benchmark
    public int readVolatile() { return vField; }

    @Benchmark
    public int readNormal() { return nField; }

    @Benchmark
    public void writeVolatile() { vField = 1; }

    @Benchmark
    public void writeNormal() { nField = 1; }
}
// 결과 해석:
// readVolatile ≈ readNormal (x86에서 읽기 펜스 불필요)
// writeVolatile << writeNormal (쓰기 펜스 비용)
```

---

## 📊 성능/비용 비교

```
volatile vs 동기화 방법별 비용 (x86, 단일 스레드 접근 기준):

방법               | 처리량 (ops/ms) | 비용 배율 | 비고
──────────────────┼────────────────┼──────────┼────────────────────
일반 int 읽기       | 500,000        | 1x        | L1 캐시 히트
volatile int 읽기  | 490,000        | ~1x       | x86 추가 비용 거의 없음
일반 int 쓰기       | 500,000        | 1x        | Store Buffer 활용
volatile int 쓰기  | 10,000~50,000  | 10~50x    | StoreLoad 펜스
AtomicInt CAS      | 30,000~80,000  | 6~17x     | LOCK CMPXCHG
synchronized (무경합)| 20,000~60,000 | 8~25x     | Thin Lock

경합 상황 (10 스레드):
방법               | 처리량 (ops/ms) | 비교
──────────────────┼────────────────┼─────────────────────
volatile 쓰기      | 1,000~5,000    | Cache Invalidate 경쟁
AtomicInt CAS      | 3,000~10,000   | CAS 실패 재시도 포함
synchronized       | 1,000~5,000    | OS Mutex (경합 시)
LongAdder          | 50,000~100,000 | Cell 분산으로 경합 회피
```

---

## ⚖️ 트레이드오프

```
volatile 사용 판단 기준:

volatile만으로 충분한 경우:
  ① 단일 변수 읽기/쓰기 (compound operation 없음)
  ② 단순 boolean 플래그 (취소, 종료 신호)
  ③ 완전히 초기화된 객체의 참조 공개 (publication)
  ④ 단방향 카운터 (한 스레드만 쓰고 여러 스레드가 읽음)

volatile이 부족한 경우:
  ① count++ 같은 read-modify-write
  ② check-then-act (if(null) 후 설정)
  ③ 복합 조건 기반 상태 전환
  → 해결: AtomicXxx, synchronized, ReentrantLock

volatile의 성능 함정:
  volatile 쓰기가 빈번한 핫 경로에서는 병목 될 수 있음
  대안: LongAdder (분산 셀), 로컬 누적 후 volatile 쓰기
```

---

## 📌 핵심 정리

```
volatile 핵심:

메모리 펜스:
  읽기: [LoadLoad][LoadStore] 이후 읽기
  쓰기: [StoreStore] 이전, 이후 [StoreLoad] → Store Buffer 플러시

x86 구현:
  읽기: 추가 명령어 없음 (TSO에서 무료)
  쓰기: lock addl $0x0, (%rsp)

보장하는 것:
  가시성: 쓰기 → 이후 읽기에 항상 보임
  순서: happens-before 관계
  64비트 원자성: long/double의 단일 읽기/쓰기

보장 안 하는 것:
  복합 연산 원자성: count++는 3단계
  check-then-act 원자성: 조건+실행 사이 끼어들기 가능
  복수 변수 간 원자성

적합한 패턴:
  취소/종료 플래그
  완전 초기화된 객체 참조 공개
  단방향 상태 플래그 (한 스레드만 씀)

부적합한 패턴:
  카운터 (→ AtomicInteger)
  lazy initialization (→ DCL + volatile 또는 Holder 패턴)
  복합 상태 변경 (→ synchronized)
```

---

## 🤔 생각해볼 문제

**Q1.** `volatile long`과 그냥 `long`의 차이는 무엇인가? 32비트 JVM에서 특별한 이유가 있는가?

<details>
<summary>해설 보기</summary>

JMM 스펙에서 `long`과 `double`은 64비트이므로, 32비트 JVM에서 두 번의 32비트 연산으로 나뉠 수 있다. 이때 두 스레드가 동시에 `long` 변수에 접근하면 "word tearing"이 발생할 수 있다. 예를 들어 Thread A가 상위 32비트, Thread B가 하위 32비트를 쓰면 엉뚱한 값이 된다.

`volatile long`은 이를 원자적 읽기/쓰기로 보장한다. 64비트 JVM에서는 `long` 읽기/쓰기 자체가 원자적이지만, 스펙상 보장은 아니므로 멀티스레드 환경에서는 `volatile long` 또는 `AtomicLong`을 사용하는 것이 안전하다.

현대 64비트 JVM에서는 실제로 word tearing이 발생하지 않지만, JMM 스펙에 따라 코드를 작성하는 것이 이식성과 정확성을 보장한다.

</details>

---

**Q2.** `volatile`이 붙은 배열 (`volatile int[] arr`)은 배열 원소의 가시성을 보장하는가?

<details>
<summary>해설 보기</summary>

아니다. `volatile int[] arr`는 **배열 참조** `arr`에 volatile이 적용된 것이다. `arr[0] = 1`과 같이 배열 원소를 쓰는 것은 volatile 쓰기가 아니며 가시성이 보장되지 않는다.

배열 원소의 volatile 접근이 필요하다면:
1. `AtomicIntegerArray` 사용 (내부적으로 Unsafe를 통해 원자적 접근)
2. `VarHandle`을 통해 개별 원소에 volatile 접근
3. 배열 원소 접근을 `synchronized`로 감싸기

```java
import java.util.concurrent.atomic.AtomicIntegerArray;
AtomicIntegerArray arr = new AtomicIntegerArray(10);
arr.set(0, 42);           // volatile 쓰기
int v = arr.get(0);      // volatile 읽기
arr.compareAndSet(0, 42, 43);  // CAS
```

</details>

---

**Q3.** `volatile` 변수에 대한 쓰기가 완료된 후, 다른 코어가 읽으면 반드시 최신 값을 보는가? "완료된 후"의 의미는 무엇인가?

<details>
<summary>해설 보기</summary>

"완료된 후"는 happens-before 관계가 성립된 이후를 의미한다. 단순히 물리적 시간(벽시계 시간)이 아닌, JMM이 정의하는 논리적 선후 관계다.

Thread A가 `vol = 1`을 수행한 후, Thread B가 `vol`을 읽어 **`1`을 관찰**했다면, 이는 A의 쓰기 →HB→ B의 읽기가 성립한 것이다. 이때 A의 쓰기 이전의 모든 쓰기(일반 변수 포함)도 B에게 보인다.

반대로 Thread B가 `0`을 읽었다면, A의 쓰기가 B의 읽기보다 happens-before 관계가 성립하지 않은 것이다(동시에 실행된 것으로 간주). 이 경우 0을 보는 것도 JMM 상 합법적이다.

따라서 "최신 값을 반드시 본다"는 것은 "A의 쓰기 이후 B가 읽어서 A의 값을 관찰했다면, A의 이전 모든 쓰기도 보인다"는 의미다. volatile은 "가장 최신 값"을 항상 보장하는 것이 아니라, happens-before 체인이 성립할 때 올바른 값을 보장한다.

</details>

---

<div align="center">

**[⬅️ 이전: happens-before](./03-happens-before.md)** | **[홈으로 🏠](../README.md)** | **[다음: 명령어 재정렬 ➡️](./05-instruction-reordering.md)**

</div>
